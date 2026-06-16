# Trip Planner Agent — Azure Function (Gemini-powered)

An HTTP-triggered Azure Function that generates a structured trip cost
breakdown and travel guide using a configurable Google Gemini model.

## Endpoint

```
POST /api/planTrip
```

### Request body

```json
{
  "origin": "Auckland, New Zealand",
  "destination": "Tokyo, Japan",
  "travellers": 2,
  "duration_days": 7,
  "budget_level": "Mid-range"
}
```

| Field           | Type   | Required | Notes                                       |
|-----------------|--------|----------|---------------------------------------------|
| `origin`        | string | yes      | Departure city/country                       |
| `destination`   | string | yes      | Destination city/country                     |
| `travellers`    | int    | yes      | Number of travellers (≥ 1)                   |
| `duration_days` | int    | yes      | Trip length in days (≥ 1)                    |
| `budget_level`  | string | yes      | One of `Budget`, `Mid-range`, `Luxury`       |

### Response body (success)

Matches the shape of `plan_response.json`:

```json
{
  "success": true,
  "data": {
    "destination": "Tokyo, Japan",
    "currency": "JPY",
    "summary": "...",
    "categories": [
      {
        "id": "flights",
        "icon": "✈️",
        "title": "Flights",
        "subtitle": "...",
        "color": "#4a8fb5",
        "totalCost": 2400,
        "perPerson": 1200,
        "lineItems": [
          { "label": "Economy return", "value": "$1,200 pp" }
        ]
      }
      // ... accommodation, food, sightseeing, transport, misc
    ],
    "tips": ["..."],
    "visaInfo": "...",
    "bestTimeToVisit": "..."
  }
}
```

### Response body (error)

```json
{
  "success": false,
  "error": "Human-readable error message"
}
```

- `400` — invalid/missing request fields
- `502` — the Gemini API call failed or returned a response that couldn't
  be parsed/validated as the expected JSON shape
- `500` — unexpected internal error

## Configuration (Application Settings / environment variables)

| Setting           | Required | Default                                                | Description                                  |
|--------------------|----------|---------------------------------------------------------|-----------------------------------------------|
| `GEMINI_API_KEY`   | yes      | —                                                       | Your Google AI Studio / Gemini API key       |
| `GEMINI_MODEL`     | no       | `gemini-2.0-flash`                                       | Any Gemini model name, e.g. `gemini-1.5-pro` |
| `GEMINI_API_BASE`  | no       | `https://generativelanguage.googleapis.com/v1beta`       | Override for proxies / other API versions    |

Set these via `local.settings.json` for local development (already
templated — fill in your real API key, do not commit it), or via
**Configuration > Application settings** in the Azure Portal / `az
functionapp config appsettings set` for deployment.

## Project structure

```
trip-planner-function/
├── function_app.py      # Function code (Python v2 programming model)
├── requirements.txt      # Python dependencies
├── host.json              # Azure Functions host configuration
├── local.settings.json    # Local-only settings (fill in your API key)
└── .funcignore
```

## Local development

Prerequisites:
- Python 3.10+
- [Azure Functions Core Tools v4](https://learn.microsoft.com/azure/azure-functions/functions-run-local)
- [Azurite](https://learn.microsoft.com/azure/storage/common/storage-use-azurite) (for local storage emulation), or set
  `AzureWebJobsStorage` to a real storage account connection string.

```bash
cd trip-planner-function
python -m venv .venv
source .venv/bin/activate          # on Windows: .venv\Scripts\activate
pip install -r requirements.txt

# Fill in GEMINI_API_KEY in local.settings.json, then:
func start
```

Test it:

```bash
curl -X POST http://localhost:7071/api/planTrip \
  -H "Content-Type: application/json" \
  -d '{
        "origin": "Auckland, New Zealand",
        "destination": "Tokyo, Japan",
        "travellers": 2,
        "duration_days": 7,
        "budget_level": "Mid-range"
      }'
```

## Deploying to Azure

1. Create the required Azure resources (resource group, storage account,
   and a Python Function App on the Consumption or Premium plan):

```bash
az group create --name trip-planner-rg --location <region>

az storage account create \
  --name <uniquestorageaccount> \
  --location <region> \
  --resource-group trip-planner-rg \
  --sku Standard_LRS

az functionapp create \
  --resource-group trip-planner-rg \
  --consumption-plan-location <region> \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --os-type Linux \
  --name <your-function-app-name> \
  --storage-account <uniquestorageaccount>
```

2. Configure the Gemini settings:

```bash
az functionapp config appsettings set \
  --name <your-function-app-name> \
  --resource-group trip-planner-rg \
  --settings GEMINI_API_KEY="<your-gemini-api-key>" GEMINI_MODEL="gemini-2.0-flash"
```

3. Deploy the code:

```bash
cd trip-planner-function
func azure functionapp publish <your-function-app-name>
```

4. Once deployed, the function will be available at:

```
https://<your-function-app-name>.azurewebsites.net/api/planTrip?code=<function-key>
```

(The `code` query parameter is required because the function uses
`AuthLevel.FUNCTION`. Retrieve it from the Azure Portal under
**Functions > plan_trip > Function Keys**, or set `AuthLevel.ANONYMOUS`
in `function_app.py` if you front the API with your own auth layer.)

## Notes

- The function asks Gemini to return `application/json` directly
  (`responseMimeType: "application/json"`), and additionally strips any
  stray markdown code fences and validates the structure before returning
  it, to make the response robust against minor model formatting drift.
- `validate_plan()` checks that all six required categories
  (`flights`, `accommodation`, `food`, `sightseeing`, `transport`, `misc`)
  and all required top-level fields are present. If Gemini's response is
  missing required fields, the function returns a `502` with details.
- Swap models at any time by changing the `GEMINI_MODEL` app setting — no
  code changes or redeploy required.
