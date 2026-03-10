# Day 1: API Foundations & FastAPI Intro

*Why does an AI Engineer care about this? Because machine learning models are useless if they sit on your laptop. To serve models to users, you need APIs. Mastering HTTP and FastAPI is how you put AI into production.*

---

## 🌐 HTTP Methods (HTTP Verbs)
HTTP methods are instructions sent to the server. While the URL is the *address* of the data, the HTTP method tells the server exactly *what to do* with it.

> **💡 Core Concept: Idempotency**
> If a method is idempotent, making the same request 1 time or 100 times will produce the exact same result on the server.

- **GET** (Retrieve): Fetch data from a resource. 
  - *Location:* Data is sent in the URL (query strings).
  - *Idempotent:* Yes. Refreshing a profile page doesn't change the profile.
- **POST** (Create): Send data to create a new resource.
  - *Location:* Hidden in the request body (secure for passwords/large data).
  - *Idempotent:* No. Clicking "Submit Payment" twice might charge you twice.
- **PUT** (Replace): Replaces the entire resource.
  - *Idempotent:* Yes. 
  - *Warning:* If you PUT a username update but leave the email field empty, the email will be overwritten as `null`.
- **PATCH** (Modify): Applies a partial update (e.g., changing just the email).
  - *Idempotent:* Usually not, depending on complex backend logic.
- **DELETE** (Remove): Deletes the specified resource.
  - *Idempotent:* Yes. Deleting a file 10 times results in the same outcome: the file is gone.
- **HEAD:** Identical to GET, but returns only headers (no body). Used to check file size or availability.
- **OPTIONS:** The browser sends a "preflight" request to ask the server which HTTP methods are allowed for a specific URL before actually sending the main request.

---

## 🚦 Common Status Codes in FastAPI

### Success (2xx)
- **200 OK:** The default. Used for successful GET, PUT, or PATCH requests.
- **201 Created:** Used for a successful POST request.
- **204 No Content:** Used for a successful DELETE request.

### Client Errors (4xx) - *The client messed up*
- **400 Bad Request:** The client sent bad data.
- **401 Unauthorized:** The user isn't logged in.
- **403 Forbidden:** The user is logged in, but lacks permission to see this data.
- **404 Not Found:** The requested ID or route doesn't exist in the database.
- **422 Unprocessable Entity:** FastAPI's superpower. It automatically sends this if the data fails type validation (e.g., sending a string instead of an integer).

### Server Errors (5xx) - *The backend messed up*
- **500 Internal Server Error:** Something crashed in the Python code.

---

## Introduction to FastAPI

<p align="center">
  <img src="https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi&logoColor=white" height="60">
</p>

FastAPI is a modern, high-performance web framework for building APIs with standard Python type hints.

1. **Production Speed:** Significantly faster than Flask because it supports `async`/`await` natively.
2. **Automatic Docs:** Automatically generates interactive Swagger UI (`/docs`) and ReDoc (`/redoc`) documentation. 
3. **Data Safety:** No manual parsing. FastAPI automatically validates data and throws a `422` error if the payload is wrong.

---

## 💻 Sample Code (`main.py`)
*Notice how we use `HTTPException` to manually trigger the 404 status code we learned about above!*

```python
from fastapi import FastAPI, Body, HTTPException, status

app = FastAPI()

# Our "In-Memory" Database
db = {1: "Laptop", 2: "Mouse"}

# 1. READ (GET)
@app.get("/items", status_code=status.HTTP_200_OK)
def read_items():
    return db

# 2. CREATE (POST) - 'name' is sent as a raw string in the request body
@app.post("/items/{item_id}", status_code=status.HTTP_201_CREATED)
def create_item(item_id: int, name: str = Body(...)):
    if item_id in db:
        raise HTTPException(status_code=400, detail="Item already exists")
    db[item_id] = name
    return {"message": "Added", "item": name}

# 3. UPDATE (PUT)
@app.put("/items/{item_id}", status_code=status.HTTP_200_OK)
def update_item(item_id: int, name: str = Body(...)):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    db[item_id] = name
    return {"message": "Updated"}

# 4. DELETE (DELETE)
@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(item_id: int):
    if item_id not in db:
        raise HTTPException(status_code=404, detail="Item not found")
    del db[item_id]
    return 

```
## Conclusion
--
**What we covered today:**
- HTTP Methods & Idempotency (GET, POST, PUT, PATCH, DELETE)
- Status Codes (200, 201, 204, 400, 401, 403, 404, 422, 500)
- FastAPI basics with automatic docs (`/docs`)
- Error handling with `HTTPException` (404 Not Found)

**Tomorrow (Day 2): Pydantic + FastAPI + Async/Await**
- Pydantic models for automatic request/response validation
- `async def` vs `def` - why AI Engineers need async for model serving
- Real-world models (User, Item, embeddings) instead of simple strings
- Dependency injection basics

*Mastering Pydantic is your FastAPI superpower - it eliminates 90% of validation bugs!*