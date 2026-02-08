# Phase 1: Backend Foundation

This phase focuses on setting up the core backend infrastructure, including the FastAPI application, Redis integration, JWT authentication, and defining the initial API routes.

---

## 2. Tech Stack (Backend)

```
Framework:     FastAPI 0.104+
Language:      Python 3.11+
ASGI Server:   Uvicorn
Async:        asyncio + httpx
Validation:   Pydantic v2
Auth:         Alien JWT verification
```

### Database
```
Primary:      Redis 7.0+ (in-memory)
Data Types:   Hashes, Sets, Sorted Sets, Strings
Persistence:  None (ephemeral game state only)
TTL:         Auto-expire finished games
```

---

## 3. System Architecture (Backend Relevant)

```
┌─────────────────────────────────────────────────────────────┐
│                   FASTAPI BACKEND                           │
│  • JWT validation                                           │
│  • Game logic & state management                            │
│  • Payment webhook handler                                  │
│  • Number calling background task                           │
└────────────────┬────────────────────────────────────────────┘
                 │
         ┌───────┴───────┐
         │               │
    ┌────▼────┐    ┌────▼─────┐
    │  REDIS  │    │  ALIEN   │
    │IN-MEMORY│    │ PAYMENT  │
    │  STORE  │    │   API    │
    └─────────┘    └──────────┘
```

---

## 5. Data Models (Pydantic Models)

This section details the Pydantic models used for data validation and serialization in the FastAPI backend.

```python
from pydantic import BaseModel, Field
from typing import List, Literal, Optional
from datetime import datetime

class NumberSelection(BaseModel):
    numbers: List[int] = Field(..., min_items=9, max_items=9)
    
    @validator('numbers')
    def validate_numbers(cls, v):
        if len(set(v)) != 9:
            raise ValueError("Numbers must be unique")
        if not all(1 <= n <= 50 for n in v):
            raise ValueError("Numbers must be between 1 and 50")
        return v

class GridArrangement(BaseModel):
    grid: List[List[int]]  # 3x3 nested list
    
    @validator('grid')
    def validate_grid(cls, v):
        if len(v) != 3 or any(len(row) != 3 for row in v):
            raise ValueError("Grid must be 3x3")
        flat = [n for row in v for n in row]
        if len(set(flat)) != 9:
            raise ValueError("Grid must contain 9 unique numbers")
        return v

class Player(BaseModel):
    alien_id: str
    numbers: List[int]
    grid: List[List[int]]
    active: bool = True
    joined_at: datetime

class GameState(BaseModel):
    lobby_id: str
    status: Literal['forming', 'arranging', 'active', 'finished']
    buy_in_amount: int
    pot: int
    players: dict[str, Player]
    numbers_called: List[int] = []
    latest_number: Optional[int] = None
    previous_number: Optional[int] = None
    winner: Optional[str] = None
    created_at: datetime
    started_at: Optional[datetime] = None
    finished_at: Optional[datetime] = None

class InvoiceRequest(BaseModel):
    alien_id: str
    buy_in_amount: int
    lobby_id: str

class InvoiceResponse(BaseModel):
    invoice_id: str
    amount: str
    recipient: str

class ClaimRequest(BaseModel):
    alien_id: str

class ClaimResponse(BaseModel):
    valid: bool
    winner: bool = False
    kicked: bool = False
    pot: Optional[int] = None
    message: str
    pattern: Optional[str] = None
```

---

## 6. Backend API Specification (General & Authentication)

### Base URL
```
Development:  http://localhost:8000
Production:   https://{your-ngrok-url}.ngrok.io
```

### Authentication
All endpoints (except `/health` and `/webhook`) require Alien JWT:
```
Authorization: Bearer <alien_jwt_token>
```

#### 6.1 Health Check

```http
GET /health
```

**Response 200:**
```json
{
  "status": "healthy",
  "redis_connected": true,
  "timestamp": "2026-02-08T10:00:00Z"
}
```

---

## 8. Alien Integration (Authentication Flow - Backend)

```python
# Backend: Verify JWT
from fastapi import Depends, HTTPException, Header
import jwt
import httpx

async def verify_alien_token(authorization: str = Header(...)):
    if not authorization.startswith("Bearer "):
        raise HTTPException(status_code=401, detail="Invalid auth header")
    
    token = authorization.split(" ")[1]
    
    # Fetch JWKS from Alien SSO
    async with httpx.AsyncClient() as client:
        response = await client.get("https://sso.alien-api.com/oauth/jwks")
        jwks = response.json()
    
    # Verify JWT signature
    try:
        payload = jwt.decode(
            token,
            key=jwks,
            algorithms=["RS256"],
            audience="YOUR_PROVIDER_ADDRESS",
            issuer="https://sso.alien-api.com"
        )
        return payload["sub"]  # alien_id
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## 9. Redis Schema (Data Structures)

### 9.1 Data Structures

#### Lobby State (Hash)

```
Key:    lobby:{lobby_id}
Type:   Hash
TTL:    600 seconds (10 minutes after creation)

Fields:
  lobby_id          : string
  status            : 'forming' | 'arranging' | 'active' | 'finished'
  buy_in_amount     : int
  pot               : int
  winner            : string | null
  created_at        : timestamp
  started_at        : timestamp | null
  finished_at       : timestamp | null
```

#### Player Data (Hash per player)

```
Key:    lobby:{lobby_id}:player:{alien_id}
Type:   Hash
TTL:    Inherit from lobby

Fields:
  alien_id          : string
  numbers           : JSON array [1,5,12,...]
  grid              : JSON nested array [[...],[...],[...]]
  active            : boolean
  joined_at         : timestamp
```

#### Numbers Called (List)

```
Key:    lobby:{lobby_id}:numbers_called
Type:   List
TTL:    Inherit from lobby

Values: [12, 23, 7, 34, ...]  (pushed as game progresses)
```

#### Active Lobbies Index (Sorted Set)

```
Key:    active_lobbies:{buy_in_amount}
Type:   Sorted Set
Score:  timestamp (for sorting by creation time)
Member: lobby_id

Purpose: Find available lobbies quickly when player joins
```

#### Payment Intents (Hash)

```
Key:    invoice:{invoice_id}
Type:   Hash
TTL:    600 seconds

Fields:
  invoice_id        : string
  alien_id          : string
  lobby_id          : string
  amount            : int
  status            : 'pending' | 'finalized' | 'failed'
  created_at        : timestamp
```

### 9.2 Redis Operations Examples (Backend Foundation)

```python
import redis.asyncio as aioredis
import json

redis = aioredis.from_url("redis://localhost:6379", decode_responses=True)

# Create lobby
async def create_lobby(buy_in_amount: int) -> str:
    lobby_id = f"lobby_{uuid.uuid4().hex[:8]}"
    
    await redis.hset(f"lobby:{lobby_id}", mapping={
        "lobby_id": lobby_id,
        "status": "forming",
        "buy_in_amount": buy_in_amount,
        "pot": 0,
        "created_at": datetime.utcnow().isoformat()
    })
    
    await redis.expire(f"lobby:{lobby_id}", 600)
    await redis.zadd(f"active_lobbies:{buy_in_amount}", {lobby_id: time.time()})
    
    return lobby_id

# Get lobby state
async def get_lobby(lobby_id: str) -> dict:
    lobby_data = await redis.hgetall(f"lobby:{lobby_id}")
    
    # Get all players
    player_keys = await redis.keys(f"lobby:{lobby_id}:player:*")
    players = {}
    for key in player_keys:
        player_data = await redis.hgetall(key)
        alien_id = player_data["alien_id"]
        players[alien_id] = {
            "alien_id": alien_id,
            "numbers": json.loads(player_data["numbers"]),
            "grid": json.loads(player_data["grid"]),
            "active": player_data["active"] == "true"
        }
    
    lobby_data["players"] = players
    return lobby_data
```

---

## 11. Implementation Checklist (Phase 1: Backend Foundation)

- [ ] FastAPI project setup
  - [ ] `main.py` with app initialization
  - [ ] `.env` file with config
  - [ ] `requirements.txt` dependencies
  
- [ ] Redis connection
  - [ ] `redis_client.py` with async client
  - [ ] Connection health check
  - [ ] Test basic operations
  
- [ ] JWT authentication
  - [ ] `auth.py` middleware
  - [ ] JWKS fetching from Alien SSO
  - [ ] Token verification dependency
  
- [ ] API routes structure
  - [ ] `/health` endpoint
  - [ ] `/api/invoices` endpoint
  - [ ] `/api/game/*` endpoints skeleton
  - [ ] `/api/webhooks/payment` endpoint

---

## Appendix A: Environment Variables (Backend)

```bash
# Backend (.env)
REDIS_URL=redis://localhost:6379
ALIEN_PROVIDER_ADDRESS=your_provider_address_from_dev_portal
WEBHOOK_PUBLIC_KEY=ed25519_public_key_hex_from_dev_portal
ALLOWED_ORIGINS=https://your-ngrok-url.ngrok.io
```

---

## Appendix B: Key Commands (Backend)

```bash
# Backend
cd backend
pip install -r requirements.txt
uvicorn main:app --reload --port 8000

# Redis
redis-server
```
