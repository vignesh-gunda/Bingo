# Phase 6: Deployment

This phase details the steps required to deploy the Fair Bingo application, including setting up development tunnels and configuring the Alien Dev Portal.

---

## 2. Tech Stack (Infrastructure)

```
Infrastructure
```
Tunnel: ngrok (dev/hackathon)
Production: Railway / Render / Fly.io
Monitoring: Structured logging (JSON)
```
```

---

## 8. Alien Integration (Mini App Configuration)

### 8.1 Mini App Configuration

**Dev Portal Settings:**
- **Name:** Fair Bingo
- **Provider URL:** https://{your-ngrok}.ngrok.io
- **Allowed Origins:** https://{your-ngrok}.ngrok.io
- **Icon:** 512×512 PNG logo
- **Description:** "Multiplayer Bingo with verified human players"

---

## 11. Implementation Checklist (Phase 6: Deployment)

- [ ] ngrok setup
  - [ ] Start tunnel
  - [ ] Update frontend API URL
  
- [ ] Alien Dev Portal
  - [ ] Update allowed origins
  - [ ] Set webhook URL
  - [ ] Test via deeplink
  
- [ ] Final testing
  - [ ] Full game with real payments
  - [ ] Verify webhook confirmations

---

## Appendix B: Key Commands (ngrok)

```bash
# ngrok
ngrok http 5173
```
