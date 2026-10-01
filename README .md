# FastAPI + PostgreSQL — Backend Documentation

> Step-by-step backend documentation for [`Dhanvanth23/FastAPI-practice`](https://github.com/Dhanvanth23/FastAPI-practice)  
> Stack: **Python · FastAPI · SQLAlchemy · PostgreSQL · Pydantic**

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Backend File Structure](#2-backend-file-structure)
3. [database.py — Connecting to PostgreSQL](#3-databasepy--connecting-to-postgresql)
4. [database_models.py — ORM Table Model](#4-database_modelspy--orm-table-model)
5. [models.py — Pydantic Schema](#5-modelspy--pydantic-schema-api-validation)
6. [main.py — FastAPI App & Routes](#6-mainpy--fastapi-app--api-routes)
7. [End-to-End Data Flow](#7-end-to-end-data-flow--how-a-request-works)
8. [How to Run](#8-how-to-run-this-backend)
9. [API Quick Reference](#9-api-quick-reference)

---

## 1. Project Overview

This project is a RESTful backend API built with **FastAPI** that manages a product catalogue. It connects to a **PostgreSQL** database using **SQLAlchemy**, uses **Pydantic** for request/response validation, and exposes full CRUD (Create, Read, Update, Delete) endpoints for products.

| Item | Detail |
|------|--------|
| **Database** | PostgreSQL (local, database name: `fast`) |
| **ORM** | SQLAlchemy (Core + ORM layer) |
| **Validation** | Pydantic |
| **Server** | Uvicorn (ASGI) |

---

## 2. Backend File Structure

The backend consists of four Python files. The `frontend/` folder is excluded from this documentation.

```
FastAPI-practice/
├── database.py          ← DB connection setup (engine + session)
├── database_models.py   ← SQLAlchemy ORM model (maps to DB table)
├── models.py            ← Pydantic schema (validates API input/output)
└── main.py              ← FastAPI app + all API route handlers
```

> **How the files connect:** `main.py` imports from all three files. `database.py` creates the database engine. `database_models.py` uses that engine to create tables. `models.py` validates the data coming in from API requests. `main.py` ties everything together into usable API endpoints.

---

## 3. `database.py` — Connecting to PostgreSQL

This is the entry point for all database communication. It creates the **engine** (the connection to PostgreSQL) and the **session factory** (the object you use to run queries).

### Full Code

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

db_url = "postgresql://postgres:1234@localhost:5432/fast"

engine = create_engine(db_url)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
session = SessionLocal()
```

### Line-by-Line Explanation

---

#### `from sqlalchemy import create_engine`

`create_engine()` is the core SQLAlchemy function. It takes a database URL and creates a **connection pool** that talks to your database.

---

#### `from sqlalchemy.orm import sessionmaker`

`sessionmaker` is a factory for `Session` objects. A **Session** is the "workspace" where you run queries, add rows, and commit changes.

---

#### `db_url = "postgresql://postgres:1234@localhost:5432/fast"`

The database URL format is: `dialect://user:password@host:port/dbname`

| Part | Value | Meaning |
|------|-------|---------|
| `postgresql://` | dialect | Tells SQLAlchemy to use the psycopg2 PostgreSQL driver |
| `postgres` | user | The PostgreSQL username |
| `1234` | password | The password |
| `localhost` | host | Database server is on the same machine |
| `5432` | port | Default PostgreSQL port |
| `fast` | dbname | The database to connect to |

---

#### `engine = create_engine(db_url)`

This creates the engine but does **NOT** open a connection yet. Connections are opened lazily when the first query is made. The engine also manages a **connection pool** behind the scenes.

---

#### `SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)`

Creates a **class** (not an instance). Every time you call `SessionLocal()` you get a brand new database session.

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `autocommit` | `False` | Changes are NOT saved automatically. You must call `db.commit()` manually. |
| `autoflush` | `False` | SQLAlchemy won't sync pending changes before each query automatically. |
| `bind` | `engine` | Tells this session factory which database engine to use. |

---

#### `session = SessionLocal()`

Creates one **global session**. This is used in `init_db()` to seed initial data. For actual API requests, a fresh session is created per request (see `get_db()` in `main.py`).

> ⚠️ **Production Note:** Hardcoding credentials in the URL is only okay for learning. In a real project, use environment variables:
> ```python
> import os
> db_url = os.environ["DATABASE_URL"]
> ```

---

## 4. `database_models.py` — ORM Table Model

This file defines the Python class that maps to a real table in your PostgreSQL database. SQLAlchemy reads this class and knows exactly what columns to create in the DB.

### Full Code

```python
from sqlalchemy import Column, Integer, String, Float
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class Product(Base):
    __tablename__ = "product"

    id          = Column(Integer, primary_key=True, index=True)
    name        = Column(String)
    description = Column(String)
    price       = Column(Float)
    quantity    = Column(Integer)
```

### Line-by-Line Explanation

---

#### `from sqlalchemy import Column, Integer, String, Float`

`Column()` defines a table column. The type classes determine what PostgreSQL data type that column gets:

| SQLAlchemy Type | PostgreSQL Type |
|-----------------|-----------------|
| `Integer` | `INT` |
| `String` | `VARCHAR` |
| `Float` | `DOUBLE PRECISION` |

---

#### `from sqlalchemy.ext.declarative import declarative_base`

`declarative_base()` returns a base class. Any Python class that **inherits from it** becomes an ORM model — SQLAlchemy will treat it as a database table definition.

---

#### `Base = declarative_base()`

`Base` is the parent class all your models will inherit from. It keeps a **registry of all models**, which is why `Base.metadata.create_all()` can find and create all tables at once.

---

#### `class Product(Base):`

By inheriting from `Base`, this class becomes a SQLAlchemy ORM model. SQLAlchemy will map it to a database table.

---

#### `__tablename__ = "product"`

This is the actual name of the table in PostgreSQL. When SQLAlchemy creates the table, it creates it as `"product"`. All SQL queries will target this table name.

---

#### Column Definitions

```python
id          = Column(Integer, primary_key=True, index=True)
name        = Column(String)
description = Column(String)
price       = Column(Float)
quantity    = Column(Integer)
```

| Column | Type | Notes |
|--------|------|-------|
| `id` | `Integer` | `primary_key=True` → auto-incrementing unique ID. `index=True` → adds a DB index for fast lookups. |
| `name` | `String` | Maps to `VARCHAR`. Stores text. |
| `description` | `String` | Same as name — a text field. |
| `price` | `Float` | Maps to `DOUBLE PRECISION`. Stores decimals like `699.99`. |
| `quantity` | `Integer` | Maps to `INT`. Stores whole numbers. |

> **How ORM works — the analogy:** Think of the Python class as a "blueprint" for a database table. SQLAlchemy reads the blueprint and creates the actual table in PostgreSQL. When you do `db.add(product_object)`, SQLAlchemy converts that Python object into a SQL `INSERT` statement automatically. You never write raw SQL.

---

## 5. `models.py` — Pydantic Schema (API Validation)

This file defines the **shape of the data** that the API accepts and returns. It has nothing to do with the database — it only validates what comes in through the HTTP request body.

### Full Code

```python
from pydantic import BaseModel

class Product(BaseModel):
    id:          int
    name:        str
    description: str
    price:       float
    quantity:    int
```

### Line-by-Line Explanation

---

#### `from pydantic import BaseModel`

Pydantic is FastAPI's data validation library. Any class that inherits `BaseModel` automatically gets:
- Type checking on all fields
- Automatic error messages for bad input (FastAPI returns `422` if validation fails)
- JSON serialization

---

#### `class Product(BaseModel):`

This is **not** a database model. It's a **schema** — it describes what a valid product JSON body looks like. FastAPI uses this to validate incoming request bodies automatically.

---

#### Field Declarations

Each field has a Python type annotation. Pydantic enforces these at runtime. If a client sends `price` as a string (`"ten"`), FastAPI returns a `422` error **automatically** before your code even runs.

---

> **Pydantic Model vs SQLAlchemy Model — the key difference:**
>
> | | File | Purpose |
> |-|------|---------|
> | **Pydantic** | `models.py` | Validates the JSON shape of API requests/responses |
> | **SQLAlchemy** | `database_models.py` | Defines how data is stored in PostgreSQL |
>
> They look similar but serve completely different purposes. In `main.py` you convert between them using `**product.model_dump()` to unpack Pydantic into SQLAlchemy.

---

## 6. `main.py` — FastAPI App & API Routes

This is the heart of the application. It creates the FastAPI app, connects it to the database, seeds initial data, and defines all the API endpoints.

### 6.1 Imports

```python
from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware
from models import Product
from database import session, engine, SessionLocal
import database_models
from sqlalchemy.orm import Session
```

| Import | Purpose |
|--------|---------|
| `FastAPI` | The main app class |
| `Depends` | FastAPI's dependency injection tool — used to inject a DB session into each route |
| `CORSMiddleware` | Allows the frontend (on a different port) to make requests to this backend |
| `Product` | The Pydantic model for request validation (from `models.py`) |
| `session, engine, SessionLocal` | The DB connection objects from `database.py` |
| `database_models` | The SQLAlchemy table model (from `database_models.py`) |
| `Session` | The SQLAlchemy Session type, used for type hints in route functions |

---

### 6.2 App Initialization & CORS

```python
app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["*"]
)
```

`app = FastAPI()` creates the FastAPI application instance. All routes are registered on this object.

**CORS** (Cross-Origin Resource Sharing) is a browser security rule. By default, a browser blocks JavaScript on port 3000 from calling an API on port 8000. This middleware tells the browser: *"It's okay, I allow requests from localhost:3000."*

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `allow_origins` | `["http://localhost:3000"]` | Only allow requests from this origin (the React frontend) |
| `allow_methods` | `["*"]` | Allow all HTTP methods: GET, POST, PUT, DELETE |

---

### 6.3 Create Database Tables

```python
database_models.Base.metadata.create_all(bind=engine)
```

Reads all classes that inherit from `Base` (i.e., the `Product` ORM model) and **creates their corresponding tables** in PostgreSQL — if they don't already exist. Safe to call every time the app starts; it won't drop existing tables.

What this runs in PostgreSQL:
```sql
CREATE TABLE IF NOT EXISTS product (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR,
    description VARCHAR,
    price       DOUBLE PRECISION,
    quantity    INTEGER
);
```

---

### 6.4 `get_db()` — The Dependency Injector

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

This is a **generator function** that FastAPI uses as a dependency. Here's what happens step by step:

1. A new DB session is created: `db = SessionLocal()`
2. The session is **handed to the route function** via `yield db`
3. The route function runs and uses `db` to query the database
4. When the route finishes (whether success or error), `finally: db.close()` runs — **always**

`Depends(get_db)` in each route function is FastAPI's way of calling `get_db()` automatically before the route runs. This pattern ensures:
- Every request gets its **own isolated session**
- The session is **always closed**, preventing connection leaks

---

### 6.5 `init_db()` — Seed Initial Data

```python
def init_db():
    db = session
    count = db.query(database_models.Product).count()
    if count == 0:
        for product in products:
            db.add(database_models.Product(**product.model_dump()))
        db.commit()

init_db()
```

| Line | What it does |
|------|-------------|
| `db.query(database_models.Product).count()` | Runs `SELECT COUNT(*) FROM product` |
| `if count == 0` | Only seed data if the table is empty (avoids duplicates on restart) |
| `product.model_dump()` | Converts Pydantic object to a dict: `{"id": 1, "name": "Phone", ...}` |
| `database_models.Product(**product.model_dump())` | Unpacks the dict into the SQLAlchemy ORM constructor |
| `db.add(...)` | Stages the new row for insertion (not saved yet) |
| `db.commit()` | Saves all staged rows to PostgreSQL in one transaction |

---

### 6.6 API Route Handlers

#### GET `/` — Health Check

```python
@app.get("/")
def greet():
    return "Go ahead buddy"
```

A simple root route. Returns a plain string. Useful for checking that the server is running. No DB interaction.

---

#### GET `/products` — Fetch All Products

```python
@app.get("/products")
def get_all_products(db: Session = Depends(get_db)):
    db_products = db.query(database_models.Product).all()
    return db_products
```

| Line | Explanation |
|------|-------------|
| `db: Session = Depends(get_db)` | FastAPI automatically calls `get_db()`, gets a fresh session, and passes it here as `db` |
| `db.query(database_models.Product).all()` | Translates to `SELECT * FROM product;` — returns a list of SQLAlchemy Product objects |

---

#### GET `/product/{id}` — Fetch One Product

```python
@app.get("/product/{id}")
def get_product_by_id(id: int, db: Session = Depends(get_db)):
    db_product = db.query(database_models.Product).filter(
        database_models.Product.id == id
    ).first()
    if db_product:
        return db_product
    return "Product not found"
```

| Line | Explanation |
|------|-------------|
| `{id}` in URL | FastAPI extracts the id from the URL (e.g. `/product/3`) and passes it as the `id` parameter |
| `.filter(database_models.Product.id == id)` | Translates to `WHERE id = <value>` |
| `.first()` | Returns the first matching row or `None`. Translates to `LIMIT 1` |

---

#### POST `/products` — Add a New Product

```python
@app.post("/products")
def add_product(product: Product, db: Session = Depends(get_db)):
    db.add(database_models.Product(**product.model_dump()))
    db.commit()
    return product
```

| Line | Explanation |
|------|-------------|
| `product: Product` | FastAPI reads the JSON request body and validates it against the Pydantic `Product` schema |
| `product.model_dump()` | Converts the Pydantic object to a plain Python dict |
| `database_models.Product(**product.model_dump())` | Creates a new SQLAlchemy ORM instance |
| `db.add(...)` | Stages the new row for `INSERT` — not saved yet |
| `db.commit()` | Executes the `INSERT` and saves the row to PostgreSQL |

---

#### PUT `/products/{id}` — Update a Product

```python
@app.put("/products/{id}")
def update_product(id: int, product: Product, db: Session = Depends(get_db)):
    db_product = db.query(database_models.Product).filter(
        database_models.Product.id == id
    ).first()
    if db_product:
        db_product.name        = product.name
        db_product.description = product.description
        db_product.price       = product.price
        db_product.quantity    = product.quantity
        db.commit()
        return "Product updated"
    else:
        return "No product found"
```

This route demonstrates how SQLAlchemy **tracks changes**. You don't write `UPDATE` SQL. You simply:
1. Fetch the existing row as a Python object (`db_product`)
2. Change its attributes in Python (`db_product.name = product.name`)
3. Call `db.commit()` — SQLAlchemy detects the changed attributes and generates the `UPDATE` SQL automatically

---

#### DELETE `/products/{id}` — Delete a Product

```python
@app.delete("/products/{id}")
def delete_product(id: int, db: Session = Depends(get_db)):
    db_product = db.query(database_models.Product).filter(
        database_models.Product.id == id
    ).first()
    if db_product:
        db.delete(db_product)
        db.commit()
        return "Product deleted successfully"
    else:
        return "No product found"
```

| Line | Explanation |
|------|-------------|
| `db.delete(db_product)` | Marks the ORM object for deletion. Translates to `DELETE FROM product WHERE id = <value>` on commit |
| `db.commit()` | Executes the `DELETE` statement in PostgreSQL |

---

## 7. End-to-End Data Flow — How a Request Works

When you call `POST /products` with a JSON body, here is the exact sequence:

```
Client (Postman / Browser / Frontend)
        │
        │  POST /products  { "id": 5, "name": "Watch", ... }
        ▼
FastAPI receives the HTTP request
        │
        ├─► Calls get_db() → creates SessionLocal() → yields db session
        │
        ├─► Reads JSON body → validates against Pydantic Product schema
        │       └─ If invalid → auto-returns 422 error
        │
        ├─► Calls add_product(product, db)
        │       ├─ product.model_dump()  → {"id": 5, "name": "Watch", ...}
        │       ├─ database_models.Product(**dict)  → SQLAlchemy ORM object
        │       ├─ db.add(orm_object)   → stage INSERT
        │       └─ db.commit()          → INSERT INTO product ... (sent to PostgreSQL)
        │
        ├─► Returns product as JSON response
        │
        └─► finally: db.close()  → session released
```

---

## 8. How to Run This Backend

### Step 1 — Install dependencies

```bash
pip install fastapi uvicorn sqlalchemy psycopg2-binary
```

### Step 2 — Set up PostgreSQL

```sql
-- Run in psql or pgAdmin:
CREATE DATABASE fast;
-- Make sure user "postgres" with password "1234" exists
```

### Step 3 — Start the server

```bash
uvicorn main:app --reload
```

### Step 4 — Test the API

Open your browser and go to:

```
http://localhost:8000/docs
```

FastAPI auto-generates an **interactive Swagger UI** where you can test all endpoints without writing any frontend code. You can POST a new product, GET all products, and DELETE by ID directly from the browser.

---

## 9. API Quick Reference

| Method | Endpoint | Action | SQL Operation |
|--------|----------|--------|---------------|
| `GET` | `/` | Health check | None |
| `GET` | `/products` | List all products | `SELECT * FROM product` |
| `GET` | `/product/{id}` | Get one product | `SELECT ... WHERE id=? LIMIT 1` |
| `POST` | `/products` | Add product | `INSERT INTO product ...` |
| `PUT` | `/products/{id}` | Update product | `UPDATE product SET ... WHERE id=?` |
| `DELETE` | `/products/{id}` | Delete product | `DELETE FROM product WHERE id=?` |

---

*Documentation generated from source: [github.com/Dhanvanth23/FastAPI-practice](https://github.com/Dhanvanth23/FastAPI-practice)*
