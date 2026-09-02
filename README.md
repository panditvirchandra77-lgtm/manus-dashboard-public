# Manus Box — OpenClaw Dashboard ka Permanent Public URL 🦋

Manus AI sandbox box (`ihy36c3tpo9s7aljl86fc`) ke andar OpenClaw dashboard
(`127.0.0.1:18789`) ko **public URL** par expose karne ka exact setup guide.

## Problem

Manus box ne port-based public URLs pehle se expose kiye the:
```
https://<port>-ihy36c3tpo9s7aljl86fc-8e2a5044.sg2.manus.computer/
```
Par dashboard (`18789`) par **403** aa raha tha:

```json
{"error":{"message":"Proxy client attribution is required. Configure gateway.trustedProxies narrowly...","type":"proxy_attribution_required"}}
```

## Root Cause

Manus ka reverse-proxy box ke andar se traffic forward karta hai internal IP se
(log se mila: `10.69.55.1`, baad mein `10.44.177.1`). OpenClaw gateway un
forwarded client headers ko `trustedProxies` list se validate karta hai — jo
proxy list mein na ho, uska traffic reject ho jata hai (403).

## Tools Istemal Kiye

| Tool | Kaam |
|------|------|
| `ss` (netstat) | Gateway PID + port `18789` dhundhna |
| `jq` | Config JSON ko safely modify karna (merge-safe) |
| `kill -USR1` / `kill -9` | Gateway reload/restart |
| `curl` | Public URL verify (HTTP code check) |
| `openclaw devices approve` | Browser ko Control UI allow karna |
| `grep` | Log se proxy IP nikaalna |

## Steps (exact commands)

### 1. Config Backup (hamesha pehle)
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak.$(date +%Y%m%d%H%M%S)
```

### 2. Gateway PID + Port dhundhna
```bash
ss -ltnp | grep 18789
# LISTEN 0 511 0.0.0.0:18789 ... openclaw-gatewa pid=153486
```

### 3. Proxy IP Log se nikaalna
```bash
grep "unattributable proxy-shaped" /tmp/openclaw/openclaw-2026-09-02.log | grep -oP 'from \K[0-9.]+'
# 10.69.55.1  →  baad mein 10.44.177.1 (proxy IP change hota hai)
```

### 4. Config Update — 3 keys
```bash
# trustedProxies (broad private CIDR — proxy IP change bhi ho toh kaam kare)
jq '.gateway.trustedProxies = ["10.0.0.0/8","172.16.0.0/12","192.168.0.0/16"]' \
  ~/.openclaw/openclaw.json > /tmp/oc.json && mv /tmp/oc.json ~/.openclaw/openclaw.json

# publicOrigin + allowedOrigins (public URL ko origin check ke liye allow)
jq '.gateway.publicOrigin = "https://18789-ihy36c3tpo9s7aljl86fc-8e2a5044.sg2.manus.computer"' ~/.openclaw/openclaw.json > /tmp/oc.json && mv /tmp/oc.json ~/.openclaw/openclaw.json
```

### 5. Gateway Restart
```bash
# Graceful reload try karo
kill -USR1 <PID>
# Agar kaam na kare, force kill (systemd/Restart=always auto-ubhaar dega)
kill -9 <PID>
# Wait karo, phir naya PID check karo
sleep 8 && ss -ltnp | grep 18789
```

### 6. Public URL Verify
```bash
curl -s -o /dev/null -w "HTTP_CODE:%{http_code}\n" \
  https://18789-ihy36c3tpo9s7aljl86fc-8e2a5044.sg2.manus.computer/
# HTTP_CODE:200  ✅
```

### 7. Browser Device Approve
Dashboard browser se kholne par yeh maangta hai:
```bash
openclaw devices approve ddddcfe8-a74d-4ffb-9cbd-a5077efcc7bf
# Approved 31161354...
```

## Final Result

- **Public URL:** `https://18789-ihy36c3tpo9s7aljl86fc-8e2a5044.sg2.manus.computer/`
- **Login:** `gateway.auth.token` (config file se) — URL open karte hi maangega
- **Dashboard title:** "OpenClaw Control" (200 OK)

## Key Lessons

1. **Manus box public URLs pehle se exist karte hain** — port-based subdomain
   routing. Naya tunnel banane ki zaroorat nahi.
2. **403 "proxy_attribution_required"** = `gateway.trustedProxies` mein reverse
   proxy ka IP/range add karo.
3. **Proxy IP baar baar change hota hai** (10.69.x → 10.44.x) — isliye broad
   private CIDR (`10.0.0.0/8`) use karo, specific IP nahi.
4. **Config edit se pehle backup zaroori** — gateway boot fail ho sakta hai
   invalid key se.
5. **Gateway restart deferred hota hai** jab active agent work chal raha ho —
   jab tak turn khatam na ho, restart pending rehta hai.
