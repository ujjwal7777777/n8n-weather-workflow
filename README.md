# n8n-weather-workflow
# AI Weather Agent (n8n + Open-Meteo)

Importable n8n Cloud workflow that turns a natural-language location into the **latest current weather** from Open-Meteo. The AI only extracts the place name and formats the report. All temperature, wind, humidity, and forecast numbers come from the API.

## Import on n8n Cloud

1. Open [n8n Cloud](https://app.n8n.cloud) and sign in.
2. Click **Workflows** → menu (**⋮** or **+ Add workflow**) → **Import from File**.
3. Choose `n8n/weather-agent.json`.
4. Open **OpenAI - Location Extractor** and **OpenAI - Response Formatter**.
5. Create or select an **OpenAI** credential (store the API key in n8n, not in this repo).
6. **Save**, then **Publish** (or use **Listen for test event** on the webhook).

Open-Meteo public geocoding and forecast APIs do **not** need an API key.

## Call the webhook

Production URL (after publish):

```text
POST https://<your-instance>.app.n8n.cloud/webhook/weather-agent
Content-Type: application/json
```

Body (either shape):

```json
{ "message": "What's the weather in Aligarh?" }
```

```json
{ "location": "Delhi" }
```

Example:

```bash
curl -sS -X POST "https://<your-instance>.app.n8n.cloud/webhook/weather-agent" ^
  -H "Content-Type: application/json" ^
  -d "{\"message\":\"What's the weather in Aligarh?\"}"
```

Successful response:

```json
{
  "ok": true,
  "request_id": "wx_...",
  "reply": "## 🌤️ Weather in Aligarh, India\n...",
  "location": { "name": "Aligarh", "country": "India" },
  "weather_timestamp": "2026-08-24T11:15",
  "error": null
}
```

## Flow

1. Webhook receives `message` or `location`.
2. Simple place names skip the first AI call; full sentences go through location extraction.
3. Open-Meteo Geocoding (`count=5`) resolves coordinates and timezone.
4. The workflow picks a unique match, or asks you to clarify (for example Springfield).
5. Open-Meteo Forecast returns current conditions plus today's high/low, rain chance, sunrise, and sunset (`timezone=auto`).
6. Weather codes are mapped to readable conditions.
7. An LLM formats Markdown from the verified payload (draft numbers are already filled from the API). If the model fails, the deterministic draft is returned instead.

## Configuration

Default endpoints live in **02 - Normalize Input** (`config`):

| Key | Default |
| --- | --- |
| `geocoding_url` | `https://geocoding-api.open-meteo.com/v1/search` |
| `forecast_url` | `https://api.open-meteo.com/v1/forecast` |
| `language` | `en` |
| `forecast_days` | `1` |

HTTP nodes use a 15s timeout and retry up to 3 times on transient failure.
