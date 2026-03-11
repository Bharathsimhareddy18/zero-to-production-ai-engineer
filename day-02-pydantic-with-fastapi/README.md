# Day 2: Pydantic + FastAPI + Async/Await

*Why does an AI Engineer care about this? Because without Pydantic your LLM APIs will crash on bad JSON from users or frontend. Pydantic + FastAPI = automatic validation, parsing, serialization and 422 errors before your code even runs. This is how you serve models in production without bugs.*

<p align="center">
  <img src="https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi&logoColor=white" height="60">
</p>

## Pydantic Basics
Pydantic is a Python library for data validation & parsing that uses native Python type hints.  

For an AI Engineer, it acts as the **strict boundary layer** for your production APIs. This guarantees that the messy JSON coming from a client is perfectly transformed into exact Python objects your models & database queries expect — or it instantly raises with a clean error.

### Core Pydantic Terminology
- **BaseModel**: The foundational class you inherit from to define your data schemas  
- **Type Hints**: Python native syntax (`str`, `int`, `List[dict]`) used inside a BaseModel  
- **Field**: A function used to enforce extra data constraints (min_length, max_length, description)  
- **Serialization**: Converting your model instance back into dicts or JSON strings to send to the client  
- **Validation Error**: Automatic Pydantic exception → FastAPI turns it into 422 HTTP response  

## Basic Pydantic Model
```python
from pydantic import BaseModel

class UserQuery(BaseModel):
    query_id: int
    text: str
    is_active: bool = True
```
##  Nested Pydantic Models + List + Literal
Perfect for LLM prompts (list of messages).
```python
from pydantic import BaseModel, Field
from typing import List, Literal

class Message(BaseModel):  # nested
    role: Literal["user", "system", "assistant"]
    content: str = Field(..., min_length=1, max_length=1000)

class LLMRequest(BaseModel):  # main request
    messages: List[Message]

```
## Pydantic Model with Field Validator
Custom validation logic.

```python
from pydantic import BaseModel, field_validator

class UserRegistration(BaseModel):
    username: str

    @field_validator("username")
    @classmethod
    def check(cls, value: str) -> str:
        if len(value) < 3:
            raise ValueError("Username too short")
        return value.lower()

```
## Structured Outputs from LLM
Use Pydantic to force LLM to return clean JSON that we instantly validate.

```python 

# Inside endpoint (example)
completion = await client.chat.completions.create(
    model="gpt-4o-mini",
    messages=...,
    response_format={"type": "json_object"}
)
parsed = LLMResponse.model_validate_json(completion.choices[0].message.content)
```

## Pydantic Settings (clean & secure env vars)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    OPENAI_API_KEY: str

    class Config:
        env_file = ".env"
        extra = "ignore"

settings = Settings()  # create instance once
api_key = settings.OPENAI_API_KEY
```

- **Clean data tip**: request.model_dump(exclude_none=True) or model_dump_json() when you need dict/JSON.

## model_dump() & model_dump_json()

- **.model_dump()** → converts Pydantic model to Python dict (for DB, logging, passing to LLM)

- **.model_dump_json()** → converts to JSON string

```python
user = UserQuery(query_id=1, text="hello")
print(user.model_dump())        # {'query_id': 1, 'text': 'hello', ...}
print(user.model_dump_json())   # '{"query_id":1,"text":"hello",...}'
```

## Async/Await Small Notes (must for AI serving)

- **async def + await** = non-blocking.   
Your server can handle 100s of simultaneous LLM calls.
Use AsyncOpenAI client.
Without async your FastAPI server freezes under load — production killer.

## Final Production Code: FastAPI Server with Single LLM
```python 
from fastapi import FastAPI
from pydantic import BaseModel, Field, field_validator
from pydantic_settings import BaseSettings
from typing import List, Literal
from openai import AsyncOpenAI

app = FastAPI(title="Day 2 LLM Server")

# ==================== PYDANTIC SETTINGS ====================
class Settings(BaseSettings):
    OPENAI_API_KEY: str

    class Config:
        env_file = ".env"
        extra = "ignore"

settings = Settings()
client = AsyncOpenAI(api_key=settings.OPENAI_API_KEY)

# ==================== INPUT/OUTPUT SCHEMAS ====================
class Message(BaseModel):  # Nested + List + Literal
    role: Literal["user", "system", "assistant"]
    content: str = Field(..., min_length=1, max_length=1000)

class LLMRequest(BaseModel):
    messages: List[Message]

    @field_validator("messages")
    @classmethod
    def check_not_empty(cls, v: List[Message]) -> List[Message]:
        if not v:
            raise ValueError("Messages list cannot be empty")
        return v

class LLMResponse(BaseModel):  # Structured output
    answer: str

# ==================== ENDPOINT WITH LLM CALL ====================
@app.post("/llm/generate", response_model=LLMResponse)
async def generate_llm(request: LLMRequest):
    # Demo: model_dump() + structured output
    messages_for_llm = [{"role": "system", "content": "Always respond in valid JSON with only key 'answer'."}] + \
                       [msg.model_dump() for msg in request.messages]

    completion = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages_for_llm,
        response_format={"type": "json_object"}
    )

    raw_json = completion.choices[0].message.content
    parsed_response = LLMResponse.model_validate_json(raw_json)  # Pydantic validation

    return parsed_response
```

## RUN IT 
```
Run it: uvicorn main:app --reload → go to /docs and test.
```

## What we covered today:

- Pydantic basic + nested model + List + Literal + Field + validator
- Structured output from LLM
- Pydantic Settings
- model_dump + model_dump_json
- Async/await for LLM serving
- Full production FastAPI + single LLM endpoint with input/output schemas