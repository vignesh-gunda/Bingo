# Phase 4: Payment Integration

This phase details the implementation of the payment system, covering both the frontend interaction with the Alien SDK and the backend's invoice generation and webhook processing.

---

## 4.5 Payment Flow (Core Game Mechanics)

**Entry Fee:**
- User selects buy-in amount
- Frontend creates invoice via backend
- Payment requested through Alien SDK
- Webhook confirms transaction
- Player added to lobby

**Winner Payout:**
- **Phase 1 (MVP):** Manual payout after demo
- **Phase 2:** Automated if Alien API supports programmatic sends

---

## 6. Backend API Specification (Payment Endpoints)

#### 6.2 Create Invoice

```http
POST /api/invoices
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "alien_id": "user_abc123",
  "buy_in_amount": 3500,
  "lobby_id": "lobby_xyz789"
}
```

**Response 200:**
```json
{
  "invoice_id": "inv_abc123",
  "amount": "3500",
  "recipient": "YOUR_ALIEN_PROVIDER_ADDRESS"
}
```

**Implementation:**
1. Validate JWT → extract alien_id
2. Generate unique invoice_id (UUID)
3. Store in Redis: `invoice:{invoice_id}` → {alien_id, lobby_id, amount, status}
4. Return invoice for frontend payment

---

#### 6.8 Payment Webhook

```http
POST /api/webhooks/payment
Content-Type: application/json
X-Webhook-Signature: <ed25519_signature_hex>

{
  "invoice": "inv_abc123",
  "status": "finalized",
  "txHash": "...",
  "amount": "3500",
  "token": "ALIEN",
  "network": "alien",
  "test": false
}
```

**Response 200:**
```json
{
  "success": true
}
```

**Implementation:**
1. Read raw body as bytes
2. Get X-Webhook-Signature header
3. Verify Ed25519 signature with WEBHOOK_PUBLIC_KEY
4. If invalid → 401 Unauthorized
5. Parse JSON payload
6. Find payment intent by invoice ID
7. If status == "finalized":
   - Increment lobby pot
   - Mark player as paid
   - Check if lobby should transition to next state
8. Return success

---

## 7.2 Key Hooks (usePayment Hook)

```typescript
import { usePayment } from '@alien_org/react';

function PaymentComponent() {
  const { pay, isPaid, isLoading, txHash, errorCode } = usePayment({
    onPaid: (txHash) => console.log('Payment successful!', txHash),
    onCancelled: () => console.log('User cancelled'),
    onFailed: (errorCode) => console.error('Payment failed:', errorCode)
  });

  async function handleBuyIn() {
    // Step 1: Create invoice on backend
    const { invoice_id } = await fetch('/api/invoices', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${authToken}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        alien_id: myAlienId,
        buy_in_amount: 3500,
        lobby_id: currentLobbyId
      })
    }).then(r => r.json());

    // Step 2: Request payment
    await pay({
      recipient: 'YOUR_ALIEN_PROVIDER_ADDRESS',
      amount: '3500',  // String!
      token: 'ALIEN',
      network: 'alien',
      invoice: invoice_id,
      item: {
        title: 'Bingo Entry Fee',
        iconUrl: 'https://your-app.com/icon.png',
        quantity: 1
      },
      test: 'paid'  // For testing only
    });
  }
}
```

---

## 8. Alien Integration (Payment Integration)

**Frontend Payment Flow:**

```typescript
import { usePayment } from '@alien_org/react';

function BuyInButton() {
  const { pay } = usePayment({
    onPaid: async (txHash) => {
      console.log('Payment confirmed!', txHash);
      // Poll for webhook confirmation
      await waitForPaymentConfirmation(invoiceId);
    },
    onCancelled: () => alert('Payment cancelled'),
    onFailed: (errorCode) => alert(`Payment failed: ${errorCode}`)
  });

  async function handleBuyIn() {
    // Create invoice
    const invoice = await createInvoice(alienId, buyInAmount, lobbyId);
    
    // Request payment
    await pay({
      recipient: PROVIDER_ADDRESS,
      amount: String(buyInAmount),
      token: 'ALIEN',
      network: 'alien',
      invoice: invoice.invoice_id,
      item: {
        title: 'Bingo Entry Fee',
        iconUrl: 'https://app-url/icon.png',
        quantity: 1
      }
    });
  }
}
```

**Backend Webhook Handler:**

```python
from fastapi import Request, HTTPException
from cryptography.hazmat.primitives.asymmetric import ed25519
from cryptography.hazmat.primitives import serialization

@app.post("/api/webhooks/payment")
async def payment_webhook(request: Request):
    # Read raw body
    body = await request.body()
    signature_hex = request.headers.get("x-webhook-signature")
    
    if not signature_hex:
        raise HTTPException(status_code=401, detail="No signature")
    
    # Verify Ed25519 signature
    public_key = ed25519.Ed25519PublicKey.from_public_bytes(
        bytes.fromhex(WEBHOOK_PUBLIC_KEY)
    )
    
    try:
        public_key.verify(
            bytes.fromhex(signature_hex),
            body
        )
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    # Parse payload
    payload = json.loads(body)
    
    if payload["status"] == "finalized":
        # Update game state in Redis
        invoice_id = payload["invoice"]
        invoice_data = await redis.hgetall(f"invoice:{invoice_id}")
        
        lobby_id = invoice_data["lobby_id"]
        alien_id = invoice_data["alien_id"]
        amount = int(payload["amount"])
        
        # Increment pot
        await redis.hincrby(f"lobby:{lobby_id}", "pot", amount)
        
        # Mark player as paid
        await redis.hset(f"lobby:{lobby_id}:players", alien_id, "paid")
    
    return {"success": True}
```

---

## 9.1 Data Structures (Payment Intents - Redis)

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

---

## 11. Implementation Checklist (Phase 4: Payment Integration)

- [ ] Invoice creation
  - [ ] Backend endpoint implementation
  - [ ] Redis storage
  
- [ ] Payment flow
  - [ ] Frontend payment button
  - [ ] usePayment hook setup
  - [ ] Success/failure handling
  
- [ ] Webhook handler
  - [ ] Ed25519 signature verification
  - [ ] Payment confirmation logic
  - [ ] Lobby state updates
