# Phase 3: Frontend Core

This phase covers the setup of the React frontend, including its structure, key components, and integration with the Alien Mini App SDK.

---

## 2. Tech Stack (Frontend)

```
Framework:     React 18.3+
Build Tool:    Vite 5.0+
SDK:          @alien_org/react (Mini App SDK)
State:        React hooks + polling
Styling:      Tailwind CSS
```

---

## 5. Data Models (TypeScript Types)

This section defines the TypeScript interfaces used in the frontend for type safety and consistency with the backend data models.

```typescript
// types.ts
export interface Player {
  alien_id: string;
  numbers: number[];
  grid: number[][];
  active: boolean;
  joined_at: string;
}

export interface GameStatus {
  lobby_id: string;
  status: 'forming' | 'arranging' | 'active' | 'finished';
  buy_in_amount: number;
  pot: number;
  player_count: number;
  players: Record<string, Player>;
  latest_number: number | null;
  previous_number: number | null;
  winner: string | null;
  time_elapsed: number;
}

export interface ClaimResult {
  valid: boolean;
  winner: boolean;
  kicked: boolean;
  pot?: number;
  message: string;
  pattern?: string;
}
```

---

## 7. Frontend Implementation

### 7.1 Project Structure

```
frontend/
├── src/
│   ├── App.tsx                  # Root with AlienProvider
│   ├── main.tsx                 # Entry point
│   ├── config.ts                # API URL, constants
│   ├── hooks/
│   │   ├── useGameState.ts      # Polling hook
│   │   ├── usePayment.ts        # Payment flow
│   │   └── useAuth.ts           # Alien auth wrapper
│   ├── components/
│   │   ├── Lobby.tsx            # Waiting room
│   │   ├── NumberSelector.tsx   # Pick 9 numbers
│   │   ├── GridArranger.tsx     # Drag-drop 3x3 grid
│   │   ├── BingoCard.tsx        # Game card display
│   │   ├── NumberDisplay.tsx    # Current/previous numbers
│   │   ├── ClaimButton.tsx      # Bingo claim
│   │   ├── WinnerScreen.tsx     # Victory UI
│   │   ├── KickedScreen.tsx     # Invalid claim UI
│   │   └── LoadingSpinner.tsx   # Loading states
│   ├── services/
│   │   └── api.ts               # API client
│   └── types.ts                 # TypeScript definitions
├── package.json
├── vite.config.ts
├── tailwind.config.js
└── index.html
```

### 7.2 Key Hooks

#### useAlien Hook (from @alien_org/react)

```typescript
import { useAlien } from '@alien_org/react';

function MyComponent() {
  const { authToken, isReady } = useAlien();
  
  // authToken: JWT for backend API calls
  // isReady: true when Mini App fully loaded
}
```

#### useGameState Hook (Custom)

```typescript
// hooks/useGameState.ts
import { useState, useEffect } from 'react';
import { GameStatus } from '../types';

export function useGameState(lobbyId: string | null, authToken: string) {
  const [gameState, setGameState] = useState<GameStatus | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!lobbyId || !authToken) return;

    const interval = setInterval(async () => {
      try {
        const response = await fetch(`/api/game/${lobbyId}/status`, {
          headers: { Authorization: `Bearer ${authToken}` }
        });
        
        if (response.ok) {
          const data = await response.json();
          setGameState(data);
        } else {
          setError('Failed to fetch game state');
        }
      } catch (err) {
        setError(err.message);
      }
    }, 1500); // Poll every 1.5 seconds

    return () => clearInterval(interval);
  }, [lobbyId, authToken]);

  return { gameState, error };
}
```

### 7.3 Component Examples

#### App.tsx

```typescript
import { AlienProvider } from '@alien_org/react';
import GameFlow from './components/GameFlow';

function App() {
  return (
    <AlienProvider autoReady={true}>
      <div className="min-h-screen bg-gray-900 text-white">
        <GameFlow />
      </div>
    </AlienProvider>
  );
}

export default App;
```

#### NumberSelector.tsx

```typescript
import { useState } from 'react';

interface Props {
  onConfirm: (numbers: number[]) => void;
}

export default function NumberSelector({ onConfirm }: Props) {
  const [selected, setSelected] = useState<number[]>([]);

  const toggleNumber = (num: number) => {
    if (selected.includes(num)) {
      setSelected(selected.filter(n => n !== num));
    } else if (selected.length < 9) {
      setSelected([...selected, num]);
    }
  };

  return (
    <div className="p-4">
      <h2 className="text-2xl font-bold mb-4">
        Select 9 Numbers ({selected.length}/9)
      </h2>
      
      <div className="grid grid-cols-10 gap-2 mb-6">
        {Array.from({ length: 50 }, (_, i) => i + 1).map(num => (
          <button
            key={num}
            onClick={() => toggleNumber(num)}
            className={`
              h-12 rounded-lg font-bold text-lg
              ${selected.includes(num) 
                ? 'bg-green-500 text-white' 
                : 'bg-gray-700 text-gray-300'}
              hover:scale-105 transition-transform
            `}
          >
            {num}
          </button>
        ))}
      </div>

      <button
        onClick={() => onConfirm(selected)}
        disabled={selected.length !== 9}
        className="w-full py-3 bg-blue-600 rounded-lg font-bold disabled:opacity-50"
      >
        Confirm Selection
      </button>
    </div>
  );
}
```

---

## 8. Alien Integration (Authentication Flow - Frontend)

```typescript
// Frontend: Auto-injected JWT
import { useAlien } from '@alien_org/react';

function MyComponent() {
  const { authToken, isReady } = useAlien();
  
  useEffect(() => {
    if (isReady && authToken) {
      // Token available for API calls
      localStorage.setItem('alien_token', authToken);
    }
  }, [isReady, authToken]);
}
```

---

## 11. Implementation Checklist (Phase 3: Frontend Core)

- [ ] React setup
  - [ ] Vite project initialization
  - [ ] Tailwind CSS configuration
  - [ ] @alien_org/react installation
  
- [ ] Components
  - [ ] App.tsx with AlienProvider
  - [ ] NumberSelector component
  - [ ] GridArranger component
  - [ ] BingoCard component
  - [ ] NumberDisplay component
  
- [ ] Hooks
  - [ ] useGameState polling hook
  - [ ] useAuth wrapper
  - [ ] usePayment integration

---

## Appendix B: Key Commands (Frontend)

```bash
# Frontend
cd frontend
npm install
npm run dev
```

---

## Appendix A: Environment Variables (Frontend)

```bash
# Frontend (.env)
VITE_API_URL=https://your-ngrok-url.ngrok.io
VITE_PROVIDER_ADDRESS=your_provider_address_from_dev_portal
```
