# Phase 2: Game Logic

This phase focuses on implementing the core game rules, state management, number calling system, and win verification logic on the backend.

---

## 4. Core Game Mechanics

### 4.1 Number Selection & Arrangement

**Numbers:** 1-50 (no duplicates within a grid)  
**Grid:** 3×3 = 9 numbers per player  
**Selection Time:** Unlimited (during lobby formation)  
**Arrangement Time:** 90 seconds after selection  
**Timeout Behavior:** Auto-arrange randomly OR move to new lobby

### 4.2 Lobby System

**Capacity:** 2 (min) to 10 (max) players  
**Formation Timer:** 2 minutes OR until full  
**Buy-In Tiers:** 3,500 / 7,000 / 17,500 Alien coins  
**Matchmaking:** Players grouped by buy-in amount  
**State Persistence:** Redis with 10-minute TTL

### 4.3 Game Play

**Number Calling Interval:** 3 seconds per number  
**Display:** Only current + previous number (no history)  
**Polling Rate:** 1.5 seconds (frontend)  
**Manual Marking:** Player clicks to mark called numbers  
**No Auto-Mark:** Strategic memory challenge

### 4.4 Win Conditions

**Valid Patterns:**
- 3 Rows: Horizontal lines
- 3 Columns: Vertical lines  
- 2 Diagonals: Top-left to bottom-right, Top-right to bottom-left

**Claim Process:**
1. Player clicks "CLAIM BINGO" button
2. Backend verifies ALL 8 possible patterns
3. First valid claim wins pot
4. Invalid claims result in kick

**No Winner Scenario:**
- All 50 numbers called without valid claim → House keeps pot
- All players kicked for false claims → House keeps pot

---

## 6. Backend API Specification (Game Endpoints)

#### 6.3 Join Lobby

```http
POST /api/game/join
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "alien_id": "user_abc123",
  "buy_in_amount": 3500
}
```

**Response 200:**
```json
{
  "lobby_id": "lobby_xyz789",
  "status": "forming",
  "player_count": 3,
  "pot": 10500
}
```

**Implementation:**
1. Validate JWT
2. Check for available lobby with matching buy_in_amount
3. If found and not full → add player
4. If not found → create new lobby
5. Check if player already in this lobby → reject
6. Return lobby info

---

#### 6.4 Submit Numbers

```http
POST /api/game/{lobby_id}/numbers/select
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "alien_id": "user_abc123",
  "numbers": [5, 12, 23, 7, 34, 45, 2, 19, 38]
}
```

**Response 200:**
```json
{
  "success": true,
  "message": "Numbers selected. Now arrange your grid."
}
```

**Validation:**
- Exactly 9 numbers
- All unique
- All in range 1-50

---

#### 6.5 Submit Grid Arrangement

```http
POST /api/game/{lobby_id}/numbers/arrange
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "alien_id": "user_abc123",
  "grid": [
    [12, 23, 45],
    [7, 34, 50],
    [2, 19, 38]
  ]
}
```

**Response 200:**
```json
{
  "success": true,
  "verified": true,
  "message": "Grid arrangement verified"
}
```

**Validation:**
- 3x3 structure
- All 9 positions filled
- All numbers unique
- Matches previously selected numbers

---

#### 6.6 Get Game Status

```http
GET /api/game/{lobby_id}/status
Authorization: Bearer <jwt>
```

**Response 200:**
```json
{
  "lobby_id": "lobby_xyz789",
  "status": "active",
  "buy_in_amount": 3500,
  "pot": 35000,
  "player_count": 10,
  "players": {
    "user_abc123": {
      "alien_id": "user_abc123",
      "numbers": [5, 12, 23, 7, 34, 45, 2, 19, 38],
      "grid": [[12,23,45], [7,34,50], [2,19,38]],
      "active": true,
      "joined_at": "2026-02-08T10:00:00Z"
    }
  },
  "latest_number": 23,
  "previous_number": 7,
  "winner": null,
  "time_elapsed": 45
}
```

**Note:** `numbers_called` array is NOT included (server-side only)

---

#### 6.7 Claim Bingo

```http
POST /api/game/{lobby_id}/claim
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "alien_id": "user_abc123"
}
```

**Response 200 (Valid):**
```json
{
  "valid": true,
  "winner": true,
  "pot": 35000,
  "message": "🎉 YOU WON! +35,000 Alien coins",
  "pattern": "row_1"
}
```

**Response 400 (Invalid):**
```json
{
  "valid": false,
  "kicked": true,
  "message": "Invalid claim. You've been removed from the game.",
  "missing_numbers": [23, 45]
}
```

**Verification Logic:**
1. Check if game is active
2. Check if player is active
3. Get player's grid
4. Test all 8 patterns (3 rows + 3 cols + 2 diagonals)
5. For each pattern: verify all 3 numbers in `numbers_called`
6. If any pattern valid → Winner
7. If no pattern valid → Kick player

---

## 9.2 Redis Operations Examples (Game Logic)

```python
import redis.asyncio as aioredis
import json

# Assuming 'redis' client is initialized as in Phase 1

# Add player to lobby
async def add_player(lobby_id: str, alien_id: str, numbers: List[int], grid: List[List[int]]):
    await redis.hset(f"lobby:{lobby_id}:player:{alien_id}", mapping={
        "alien_id": alien_id,
        "numbers": json.dumps(numbers),
        "grid": json.dumps(grid),
        "active": "true",
        "joined_at": datetime.utcnow().isoformat()
    })

# Call number
async def call_number(lobby_id: str, number: int):
    await redis.rpush(f"lobby:{lobby_id}:numbers_called", number)
    await redis.hset(f"lobby:{lobby_id}", "latest_number", number)
    
    # Get previous number
    numbers_called = await redis.lrange(f"lobby:{lobby_id}:numbers_called", -2, -2)
    if numbers_called:
        await redis.hset(f"lobby:{lobby_id}", "previous_number", numbers_called[0])
```

---

## 10. Game Flow State Machine

```
┌──────────┐
│ forming  │  ← Players joining, selecting numbers
└────┬─────┘
     │ Trigger: Min 2 players + (timer expired OR 10 players)
     ↓
┌──────────┐
│arranging │  ← Players arranging 3x3 grids (90s timer)
└────┬─────┘
     │ Trigger: All players arranged OR timer expired
     ↓
┌──────────┐
│  active  │  ← Numbers being called every 3 seconds
└────┬─────┘
     │ Trigger: Valid claim OR all 50 numbers called
     ↓
┌──────────┐
│ finished │  ← Winner determined, payouts processed
└──────────┘
```

### State Transition Logic

```python
async def check_state_transition(lobby_id: str):
    lobby = await redis.hgetall(f"lobby:{lobby_id}")
    status = lobby["status"]
    
    if status == "forming":
        player_count = await redis.hlen(f"lobby:{lobby_id}:players")
        created_at = datetime.fromisoformat(lobby["created_at"])
        time_elapsed = (datetime.utcnow() - created_at).seconds
        
        if player_count >= 2 and (time_elapsed >= 120 or player_count >= 10):
            await redis.hset(f"lobby:{lobby_id}", "status", "arranging")
            # Start 90-second arrangement timer
            asyncio.create_task(arrangement_timer(lobby_id))
    
    elif status == "arranging":
        # Check if all players have arranged
        all_arranged = await check_all_players_arranged(lobby_id)
        if all_arranged:
            await redis.hset(f"lobby:{lobby_id}", "status", "active")
            await redis.hset(f"lobby:{lobby_id}", "started_at", datetime.utcnow().isoformat())
            # Start number calling
            asyncio.create_task(call_numbers_task(lobby_id))
    
    elif status == "active":
        # State transitions handled by claim verification
        pass
```

---

## 11. Implementation Checklist (Phase 2: Game Logic)

- [ ] Lobby management
  - [ ] `lobby.py` with create/join logic
  - [ ] Player addition/removal
  - [ ] State transitions
  
- [ ] Number calling system
  - [ ] Background task with `asyncio`
  - [ ] 3-second interval
  - [ ] Store in Redis list
  
- [ ] Win verification
  - [ ] Pattern checking (8 patterns)
  - [ ] Claim validation
  - [ ] Player kicking logic
