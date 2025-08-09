# Alexa Smart Home Skill + AWS Lambda Bridge to Home Assistant

This guide documents how to expose specific Home Assistant (HA) entities (e.g., a Zigbee light) to Alexa using a **custom Alexa Smart Home skill** and an **AWS Lambda bridge**.

**Your setup**  
- Home Assistant URL: `https://ha.mywonderful.xyz` (Cloudflared)  
- Entity exposed: `light.living_room_philips_light`

---

## 1) Amazon Account Location Rules (Important)

Your **Amazon account region** determines the `client_id` host and best Lambda region.

| Amazon Account Locale | Client ID (`client_id`)                   | Lambda Region (recommended) | Alexa Server Host          |
|---|---|---|---|
| **US / BR** | `https://pitangui.amazon.com/` | `us-east-1` (N. Virginia) | `pitangui.amazon.com` |
| **EU / UK** | `https://layla.amazon.com/`    | `eu-west-1` (Ireland)     | `layla.amazon.com`    |
| **Japan**   | `https://alexa.amazon.co.jp/`  | `ap-northeast-1` (Tokyo)  | `alexa.amazon.co.jp`  |

> ⚠ The **trailing slash** in the `Client ID` is required.

---

## 2) Home Assistant Configuration

Edit `configuration.yaml`, then restart HA:

```yaml
alexa:
  smart_home:
    filter:
      include_entities:
        - light.living_room_philips_light
    entity_config:
      light.living_room_philips_light:
        name: "Living Room Light"
        display_categories: LIGHT
```

In HA: **Settings → System → Network → External URL** = `https://ha.mywonderful.xyz`

---

## 3) Create AWS Lambda Function (Bridge)

1. **Region:** pick from the table in §1.  
2. Create a new Lambda (**Python 3.x**).  
3. Role: **AWSLambdaBasicExecutionRole**.  
4. Add **Alexa Smart Home** trigger (paste Skill ID; enable Skill ID verification).  
5. Paste the code below into `lambda_function.py`.  
6. **Environment variables:**
   - `BASE_URL = https://ha.mywonderful.xyz`
   - `DEBUG = True`
   - *(Optional for pre‑link testing)* `LONG_LIVED_ACCESS_TOKEN = <your HA token>`
7. **Deploy**.

### Lambda Bridge Code (with DEBUG toggle)
```python
import json
import logging
import os
import sys
import time
from urllib import request, error, parse

# =========
# Settings (from Lambda environment variables)
# =========
BASE_URL = os.getenv("BASE_URL", "").rstrip("/")              # e.g. https://ha.mywonderful.xyz
DEBUG = os.getenv("DEBUG", "False").lower() in ("1", "true", "yes", "on")
# Optional: for pre-link testing only (remove after account linking works)
LLAT = os.getenv("LONG_LIVED_ACCESS_TOKEN", "").strip()

# Network settings
TIMEOUT_SECS = float(os.getenv("TIMEOUT_SECS", "6.0"))        # keep under Alexa's expectations
USER_AGENT = os.getenv("USER_AGENT", "ha-alexa-bridge/2025.08")

# Logging
LOG_LEVEL = os.getenv("LOG_LEVEL", "DEBUG" if DEBUG else "INFO").upper()
logger = logging.getLogger()
for h in logger.handlers:
    logger.removeHandler(h)
logging.basicConfig(stream=sys.stdout, level=LOG_LEVEL, format="%(asctime)s %(levelname)s %(message)s")


def _extract_token(evt: dict) -> str:
    """
    Extract the OAuth access token that Alexa passed after Account Linking.
    We try the standard v3 locations, in order:
      - endpoint.scope.token
      - payload.scope.token
      - payload.grantee.token
    If DEBUG and LLAT is set, we fall back to LLAT to allow pre-link testing.
    """
    try_paths = [
        ("directive", "endpoint", "scope", "token"),
        ("directive", "payload", "scope", "token"),
        ("directive", "payload", "grantee", "token"),
    ]
    for path in try_paths:
        node = evt
        try:
            for key in path:
                node = node[key]
            if isinstance(node, str) and node:
                return node
        except Exception:
            pass

    if DEBUG and LLAT:
        logger.warning("Using LONG_LIVED_ACCESS_TOKEN (DEBUG mode). Do not use in production.")
        return LLAT

    return ""


def _forward_to_home_assistant(evt: dict, token: str) -> dict:
    """
    POST the entire Alexa directive to Home Assistant's Alexa Smart Home endpoint:
      POST {BASE_URL}/api/alexa/smart_home
      Authorization: Bearer <token>
    Returns HA's JSON body as a Python dict.
    """
    if not BASE_URL:
        raise ValueError("BASE_URL env var is not set.")
    if not token:
        raise ValueError("No OAuth token present (check Account Linking or DEBUG/LLAT).")

    url = f"{BASE_URL}/api/alexa/smart_home"
    body = json.dumps(evt).encode("utf-8")

    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
        "Accept": "application/json",
        "User-Agent": USER_AGENT,
    }

    if DEBUG:
        # Redact token in logs
        safe_headers = dict(headers)
        safe_headers["Authorization"] = "Bearer ***redacted***"
        logger.debug("Forwarding to HA %s", url)
        logger.debug("Headers: %s", safe_headers)
        logger.debug("Request size: %d bytes", len(body))
        # Log shortened directive name/type if present
        try:
            n = evt["directive"]["header"]["namespace"]
            nm = evt["directive"]["header"]["name"]
            logger.debug("Directive: %s.%s", n, nm)
        except Exception:
            logger.debug("Directive headers not found for debug preview.")

    req = request.Request(url=url, data=body, headers=headers, method="POST")

    start = time.time()
    try:
        with request.urlopen(req, timeout=TIMEOUT_SECS) as resp:
            raw = resp.read()
            latency = (time.time() - start) * 1000.0
            if DEBUG:
                logger.debug("HA status: %s, bytes: %d, latency: %.1f ms",
                             resp.status, len(raw), latency)
            if resp.status // 100 != 2:
                raise RuntimeError(f"Non-2xx from HA: {resp.status} - {raw[:512]!r}")
            return json.loads(raw or "{}")
    except error.HTTPError as e:
        payload = e.read()[:2048] if hasattr(e, "read") else b""
        logger.error("HTTPError from HA: %s %s", e.code, payload)
        raise
    except error.URLError as e:
        logger.error("URLError contacting HA: %s", e.reason)
        raise


def lambda_handler(event, context):
    """
    Main Lambda entrypoint.
    - Extract the OAuth token from the Alexa directive.
    - Forward the directive to HA's /api/alexa/smart_home.
    - Return HA's response back to Alexa.
    - On error, return a valid Alexa error payload when possible.
    """
    if DEBUG:
        # Shallow preview of the incoming directive
        try:
            hdr = event.get("directive", {}).get("header", {})
            logger.debug("Incoming: %s", json.dumps({
                "namespace": hdr.get("namespace"),
                "name": hdr.get("name"),
                "messageId": hdr.get("messageId"),
                "correlationToken": (hdr.get("correlationToken") or "")[:10] + "...",
            }))
        except Exception:
            logger.debug("Could not pretty-print incoming event.")

    try:
        token = _extract_token(event)
        response = _forward_to_home_assistant(event, token)
        # HA returns a complete Alexa response envelope; just echo it back
        return response

    except Exception as ex:
        logger.exception("Bridge error: %s", ex)

        # Build a minimal Alexa-compliant ErrorResponse fallback
        header = {
            "namespace": "Alexa",
            "name": "ErrorResponse",
            "payloadVersion": "3",
            "messageId": "bridge-" + str(int(time.time() * 1000))
        }
        try:
            req_header = event["directive"]["header"]
            header["correlationToken"] = req_header.get("correlationToken", "")
        except Exception:
            pass

        payload = {
            "type": "INTERNAL_ERROR",
            "message": "Failed to reach Home Assistant. Check BASE_URL/SSL/Cloudflare rules or Account Linking."
        }

        if "No OAuth token" in str(ex):
            payload["type"] = "INVALID_AUTHORIZATION_CREDENTIAL"
            payload["message"] = "Missing/invalid OAuth token. Re-link the skill."

        return { "event": { "header": header, "payload": payload } }
```

---

## 4) Create Alexa Smart Home Skill

1. **Create Skill** → Smart Home → Provision your own → Payload v3.  
2. **Smart Home service endpoint** → Default endpoint = your Lambda ARN (region per §1).  
3. **Save**.

---

## 5) Account Linking

- **Authorization Code Grant**  
- **Authorization URI:** `https://ha.mywonderful.xyz/auth/authorize`  
- **Access Token URI:** `https://ha.mywonderful.xyz/auth/token`  
- **Client ID:** (pick one per §1)  
  - US/BR: `https://pitangui.amazon.com/`  
  - EU/UK: `https://layla.amazon.com/`  
  - JP: `https://alexa.amazon.co.jp/`  
- **Client Secret:** any non-empty string  
- **Scope:** `smart_home`  
- **Save**.

> Link the skill using the **matching marketplace/app**:  
> - US → alexa.amazon.com / US app (Client ID = pitangui)  
> - EU/UK → alexa.amazon.co.uk / UK app (Client ID = layla)  
> - JP → alexa.amazon.co.jp / JP app (Client ID = alexa.amazon.co.jp)

---

## 6) Enable Skill & Discover

1. Alexa app → **Your Skills → Dev → [Your Skill] → Enable**.  
2. HA login/consent appears → login → **Authorize**.  
3. **Discover devices** (Alexa auto-discovers or say “Alexa, discover devices”).

---

## 7) Testing via Lambda (optional)

Use **Test events** in Lambda console:

**Discovery:**
```json
{
  "directive": {
    "header": {
      "namespace": "Alexa.Discovery",
      "name": "Discover",
      "payloadVersion": "3",
      "messageId": "abc-123"
    },
    "payload": { "scope": { "type": "BearerToken", "token": "" } }
  }
}
```

**TurnOn:**
```json
{
  "directive": {
    "header": { "namespace": "Alexa.PowerController", "name": "TurnOn", "payloadVersion": "3", "messageId": "abc-456", "correlationToken": "ct" },
    "endpoint": { "scope": { "type": "BearerToken", "token": "" }, "endpointId": "light.living_room_philips_light" },
    "payload": {}
  }
}
```

> For pre‑link smoke tests, set a temporary `LONG_LIVED_ACCESS_TOKEN` in env vars, then remove it later.

---

## 8) Production Tidy-Up

- Remove `LONG_LIVED_ACCESS_TOKEN`.  
- Set `DEBUG = False`.  
- Ensure Cloudflare doesn’t challenge `/auth/authorize`, `/auth/token`, `/api/alexa/smart_home`.  
- Set CloudWatch alarms (Errors metric; optional log filter for `"Bridge error:"`).

---

## 9) Troubleshooting Quick Notes

- **Invalid redirect URI:** client_id host (pitangui/layla/jp) must match redirect host; trailing slash required.  
- **Unable to link:** Cloudflare Access/Bot Fight Mode blocking OAuth paths.  
- **No devices:** wrong Lambda region or HA filter doesn’t include the entity.  
- **401/403 from HA:** token missing/invalid; re-link; ensure no WAF challenge on `/api/alexa/smart_home`.

---

## References

- Home Assistant Alexa Smart Home: https://www.home-assistant.io/integrations/alexa.smart_home/  
- Amazon Smart Home API: https://developer.amazon.com/en-US/docs/alexa/device-apis/alexa-smart-home.html  
- Host a Smart Home Skill on Lambda: https://developer.amazon.com/en-US/docs/alexa/smarthome/host-a-smart-home-skill-as-an-aws-lambda-function.html
