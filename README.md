## Criação dos Modelos Pydantic

from pydantic import BaseModel, Field
from typing import Optional
from bson import ObjectId
from datetime import datetime


class PyObjectId(ObjectId):
    @classmethod
    def __get_validators__(cls):
        yield cls.validate

    @classmethod
    def validate(cls, v):
        if not ObjectId.is_valid(v):
            raise ValueError('Invalid object id')
        return ObjectId(v)


class Item(BaseModel):
    id: Optional[PyObjectId] = Field(alias='_id')
    name: str
    description: Optional[str] = None
    price: float
    quantity: int
    created_at: Optional[datetime] = None
    updated_at: Optional[datetime] = None

    class Config:
        allow_population_by_field_name = True
        arbitrary_types_allowed = True
        json_encoders = {ObjectId: str}


## Criação dos Endpoints fastAPI

from fastapi import FastAPI, HTTPException
from pymongo import MongoClient
from typing import List
from datetime import datetime
from bson import ObjectId
from pydantic import BaseModel

app = FastAPI()

client = MongoClient("mongodb://localhost:27017/")
db = client.shop
items_collection = db.items


class UpdateItemModel(BaseModel):
    name: Optional[str]
    description: Optional[str]
    price: Optional[float]
    quantity: Optional[int]
    updated_at: Optional[datetime] = None


@app.post("/items/", response_model=Item)
async def create_item(item: Item):
    item.created_at = datetime.utcnow()
    item.updated_at = datetime.utcnow()
    item = item.dict(by_alias=True)
    try:
        result = items_collection.insert_one(item)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Error inserting item: {str(e)}")
    item["_id"] = result.inserted_id
    return Item(**item)


@app.get("/items/", response_model=List[Item])
async def read_items():
    items = list(items_collection.find())
    return [Item(**item) for item in items]


@app.get("/items/{item_id}", response_model=Item)
async def read_item(item_id: str):
    item = items_collection.find_one({"_id": ObjectId(item_id)})
    if item is None:
        raise HTTPException(status_code=404, detail="Item not found")
    return Item(**item)


@app.patch("/items/{item_id}", response_model=Item)
async def update_item(item_id: str, item: UpdateItemModel):
    update_data = item.dict(exclude_unset=True)
    update_data["updated_at"] = datetime.utcnow()
    result = items_collection.update_one({"_id": ObjectId(item_id)}, {"$set": update_data})
    if result.matched_count == 0:
        raise HTTPException(status_code=404, detail="Item not found")
    item = items_collection.find_one({"_id": ObjectId(item_id)})
    return Item(**item)


@app.delete("/items/{item_id}", response_model=dict)
async def delete_item(item_id: str):
    result = items_collection.delete_one({"_id": ObjectId(item_id)})
    if result.deleted_count == 0:
        raise HTTPException(status_code=404, detail="Item not found")
    return {"status": "Item deleted successfully"}


@app.get("/items/price_range/", response_model=List[Item])
async def filter_items_by_price(min_price: float, max_price: float):
    items = list(items_collection.find({"price": {"$gt": min_price, "$lt": max_price}}))
    return [Item(**item) for item in items]

## Teste com Pytest

from fastapi.testclient import TestClient
from bson import ObjectId
from main import app, items_collection
from datetime import datetime

client = TestClient(app)


def test_create_item():
    response = client.post("/items/", json={"name": "Item 1", "price": 1000.0, "quantity": 10})
    assert response.status_code == 200
    data = response.json()
    assert "id" in data
    assert data["name"] == "Item 1"
    assert data["price"] == 1000.0
    assert data["quantity"] == 10


def test_read_items():
    response = client.get("/items/")
    assert response.status_code == 200
    data = response.json()
    assert isinstance(data, list)


def test_read_item():
    item_id = str(ObjectId())
    items_collection.insert_one({
        "_id": ObjectId(item_id),
        "name": "Item 2",
        "price": 2000.0,
        "quantity": 20,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow()
    })
    response = client.get(f"/items/{item_id}")
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Item 2"
    assert data["price"] == 2000.0
    assert data["quantity"] == 20


def test_update_item():
    item_id = str(ObjectId())
    items_collection.insert_one({
        "_id": ObjectId(item_id),
        "name": "Item 3",
        "price": 3000.0,
        "quantity": 30,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow()
    })
    response = client.patch(f"/items/{item_id}", json={"name": "Updated Item 3", "price": 3500.0})
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Updated Item 3"
    assert data["price"] == 3500.0
    assert data["quantity"] == 30


def test_delete_item():
    item_id = str(ObjectId())
    items_collection.insert_one({
        "_id": ObjectId(item_id),
        "name": "Item 4",
        "price": 4000.0,
        "quantity": 40,
        "created_at": datetime.utcnow(),
        "updated_at": datetime.utcnow()
    })
    response = client.delete(f"/items/{item_id}")
    assert response.status_code == 200
    data = response.json()
    assert data["status"] == "Item deleted successfully"


def test_filter_items_by_price():
    items_collection.insert_many([
        {"name": "Item 5", "price": 6000.0, "quantity": 50, "created_at": datetime.utcnow(), "updated_at": datetime.utcnow()},
        {"name": "Item 6", "price": 7000.0, "quantity": 60, "created_at": datetime.utcnow(), "updated_at": datetime.utcnow()},
        {"name": "Item 7", "price": 8000.0, "quantity": 70, "created_at": datetime.utcnow(), "updated_at": datetime.utcnow()}
    ])
    response = client.get("/items/price_range/?min_price=5000&max_price=8000")
    assert response.status_code == 200
    data = response.json()
    assert len(data) == 2
    assert data[0]["price"] == 6000.0
    assert data[1]["price"] == 7000.0


## Executando o Servidor FastAPI

uvicorn main:app --reload

