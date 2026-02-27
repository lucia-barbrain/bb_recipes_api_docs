# BarBrain's Recipe COGS API - User Guide

> **Last Updated:** February 27, 2026
> **Owner:** Lucia Schlegel
> **Status:** Draft

This guide explains how to use the BarBrain Recipe COGS API to fetch Cost of Goods Sold (COGS) data for recipes programmatically, typically for POS integrations, profitability dashboards, or menu engineering tools.

## Overview

The Recipe COGS API provides three endpoints that allow authorized external systems to retrieve cost data for recipes managed in BarBrain:

| Endpoint | Purpose |
| --- | --- |
| `GET /fetchRecipeCogs` | Get the total calculated COGS for a single recipe |
| `GET /fetchRecipeIngredientCogs` | Get COGS broken down per ingredient for a single recipe, scaled by quantity sold |
| `POST /fetchBatchRecipeIngredientCogs` | Get COGS broken down per ingredient for multiple recipes in a single request |

All endpoints identify recipes using the `companyInternalId` — a user-defined identifier that companies assign to recipes in BarBrain (e.g., a POS item code or menu item number). This makes it easy to correlate BarBrain recipe data with external systems.

**Use cases:**

- Real-time COGS sync to POS systems
- Profitability analysis per menu item
- Ingredient-level cost tracking after a sales period
- Automated margin reporting and menu engineering
- End-of-day batch COGS analysis for all items sold

---

## Authentication

Authentication works identically to the Inventory API. Use a **Bearer token** issued through the BarBrain app or API.

See the [Inventory API - Authentication Guide](https://lucia-barbrain.github.io/barbrain-api-docs/#/) for full details on generating, rotating, and revoking tokens.

### Headers

| Header | Value | Required |
| --- | --- | --- |
| Authorization | `Bearer <api_token>` | Yes |

---

## How COGS Are Calculated

Understanding how BarBrain calculates COGS is important for interpreting the API responses correctly.

### Per-Ingredient Cost

For each ingredient in a recipe, the cost is calculated as:

```
ingredientCogs = (wholesalePrice / effectiveBaseUnitVolume) * quantity
```

Where:

- **wholesalePrice**: The purchase price per product unit (e.g., price for a 700ml bottle)
- **effectiveBaseUnitVolume**: The product's volume in its base unit (e.g., 700 for a 700ml bottle)
- **quantity**: The amount used in the recipe (e.g., 60ml)

If the ingredient has a **processing loss** (e.g., 10% waste when juicing limes), the usable volume is reduced:

```
usableVolume = effectiveBaseUnitVolume * (1 - processingLoss)
ingredientCogs = (wholesalePrice / usableVolume) * quantity
```

For **pre-products** (recipes used as ingredients in other recipes), the cost uses the source recipe's `costPerYieldUnit` instead.

### Total Recipe COGS

The total COGS includes a **3% breakage margin** applied on top of the sum of all ingredient costs:

```
totalCogs = sum(ingredientCogs for all ingredients) * 1.03
```

> **Important:** Because of the breakage margin, `totalCogs` will always be slightly higher than the sum of individual `ingredientCogs` values. This is intentional — it accounts for spillage, breakage, and overpouring that occur in real bar operations.

### Unit Conversions

The API handles unit conversions automatically. For example, if a recipe uses 60ml of a product priced per 700ml bottle, the conversion is handled internally. Supported conversions:

- Ml <-> Litre (/ * 1000)
- Gram <-> Kilogram (/ * 1000)

---

## Fetching Recipe COGS

Returns the total calculated COGS for a single recipe, identified by its company internal ID.

### Endpoint

```bash
GET https://api.barbrain.com/v2-live/fetchRecipeCogs
```

### Query Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `companyInternalId` | string | Yes | The company-defined internal identifier for the recipe (e.g., POS item code) |

### Example Request

```bash
curl -X GET "https://api.barbrain.com/v2-live/fetchRecipeCogs?companyInternalId=MOJITO-001" \
  -H "Authorization: Bearer a1b2c3d4e5f6..."
```

### Success Response (200 OK)

```json
{
  "requestDate": "2026-02-27T10:30:00Z",
  "companyId": "comp-123",
  "companyName": "Tapped Bar Munich",
  "recipe": {
    "recipeId": "rec-456",
    "recipeName": "Classic Mojito",
    "companyInternalId": "MOJITO-001",
    "category": "Drink",
    "totalCogs": 2.42,
    "currency": "EUR",
    "yield": {
      "type": "PORTION",
      "amount": 1,
      "unit": null
    },
    "ingredients": [
      {
        "productId": "prod-rum-1",
        "productName": "White Rum Premium",
        "quantity": 60,
        "unit": "Ml",
        "ingredientCogs": 1.50
      },
      {
        "productId": "prod-lime-1",
        "productName": "Fresh Lime Juice",
        "quantity": 30,
        "unit": "Ml",
        "ingredientCogs": 0.24
      },
      {
        "productId": "prod-sugar-1",
        "productName": "Simple Syrup",
        "quantity": 20,
        "unit": "Ml",
        "ingredientCogs": 0.06
      },
      {
        "productId": "prod-soda-1",
        "productName": "Soda Water",
        "quantity": 90,
        "unit": "Ml",
        "ingredientCogs": 0.18
      },
      {
        "productId": "prod-mint-1",
        "productName": "Fresh Mint",
        "quantity": 8,
        "unit": "Gram",
        "ingredientCogs": 0.33
      },
      {
        "productId": null,
        "productName": "House Syrup Base",
        "quantity": 10,
        "unit": "Ml",
        "ingredientCogs": 0.04
      }
    ]
  }
}
```

> **Note:** The last ingredient above has `"productId": null` — this is a **pre-product** (a recipe used as an ingredient in this recipe). Its cost is derived from the source recipe's cost per yield unit.

> **Note:** `totalCogs` (2.42) is slightly higher than the sum of individual `ingredientCogs` (2.35) because of the 3% breakage margin. See [How COGS Are Calculated](#how-cogs-are-calculated).

### Response Fields

#### Root Level

| Field | Type | Description |
| --- | --- | --- |
| `requestDate` | string | ISO 8601 timestamp of when the request was made |
| `companyId` | string | The company ID associated with the API token |
| `companyName` | string | Human-readable company name |
| `recipe` | object | The recipe object with COGS data |

#### Recipe Object

| Field | Type | Description |
| --- | --- | --- |
| `recipeId` | string | Unique identifier for this recipe in BarBrain |
| `recipeName` | string | Human-readable recipe name |
| `companyInternalId` | string | The company-defined internal identifier used to look up this recipe |
| `category` | string | Recipe category: `"Food"`, `"Drink"`, or `"Bundle"` |
| `totalCogs` | number | Total COGS for one recipe yield, **including 3% breakage margin** |
| `currency` | string | ISO 4217 currency code (e.g., `"EUR"`) |
| `yield` | object? | Recipe yield configuration (null if not configured) |
| `yield.type` | string | Yield type: `"PORTION"`, `"WEIGHT"`, or `"VOLUME"` |
| `yield.amount` | number | Yield amount (e.g., 1 portion, 500ml, 200g) |
| `yield.unit` | string? | Unit for WEIGHT/VOLUME yields (`"Ml"`, `"Litre"`, `"Gram"`, `"Kilogram"`). Null for PORTION |
| `ingredients` | array | List of ingredient objects with individual cost breakdown |

#### Ingredient Object

| Field | Type | Description |
| --- | --- | --- |
| `productId` | string? | Product identifier in BarBrain. **Null for pre-products** (recipes used as ingredients) |
| `productName` | string | Product name, or source recipe name for pre-products |
| `quantity` | number | Amount of this ingredient used per recipe yield |
| `unit` | string | Unit of measurement: `"Ml"`, `"Litre"`, `"Gram"`, `"Kilogram"`, `"Piece"`, etc. |
| `ingredientCogs` | number | Calculated COGS for this ingredient (before breakage margin) |

---

## Fetching Ingredient COGS by Quantity Sold

Returns COGS broken down per ingredient for a recipe, scaled by the number of units sold. This is useful for understanding total ingredient costs after a sales period.

### Endpoint

```bash
GET https://api.barbrain.com/v2-live/fetchRecipeIngredientCogs
```

### Query Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `companyInternalId` | string | Yes | The company-defined internal identifier for the recipe |
| `quantitySold` | integer | Yes | The number of recipe units sold. Must be a positive integer |

### Example Request

**Get ingredient COGS for 150 Mojitos sold:**

```bash
curl -X GET "https://api.barbrain.com/v2-live/fetchRecipeIngredientCogs?companyInternalId=MOJITO-001&quantitySold=150" \
  -H "Authorization: Bearer a1b2c3d4e5f6..."
```

### Success Response (200 OK)

```json
{
  "requestDate": "2026-02-27T10:30:00Z",
  "companyId": "comp-123",
  "companyName": "Tapped Bar Munich",
  "recipe": {
    "recipeId": "rec-456",
    "recipeName": "Classic Mojito",
    "companyInternalId": "MOJITO-001",
    "category": "Drink",
    "cogsPerUnit": 2.42,
    "currency": "EUR"
  },
  "quantitySold": 150,
  "totalCogs": 363.00,
  "ingredients": [
    {
      "productId": "prod-rum-1",
      "productName": "White Rum Premium",
      "quantityPerUnit": 60,
      "totalQuantityUsed": 9000,
      "unit": "Ml",
      "ingredientCogsPerRecipe": 1.50,
      "totalIngredientCogs": 225.00
    },
    {
      "productId": "prod-lime-1",
      "productName": "Fresh Lime Juice",
      "quantityPerUnit": 30,
      "totalQuantityUsed": 4500,
      "unit": "Ml",
      "ingredientCogsPerRecipe": 0.24,
      "totalIngredientCogs": 36.00
    },
    {
      "productId": "prod-sugar-1",
      "productName": "Simple Syrup",
      "quantityPerUnit": 20,
      "totalQuantityUsed": 3000,
      "unit": "Ml",
      "ingredientCogsPerRecipe": 0.06,
      "totalIngredientCogs": 9.00
    },
    {
      "productId": "prod-soda-1",
      "productName": "Soda Water",
      "quantityPerUnit": 90,
      "totalQuantityUsed": 13500,
      "unit": "Ml",
      "ingredientCogsPerRecipe": 0.18,
      "totalIngredientCogs": 27.00
    },
    {
      "productId": "prod-mint-1",
      "productName": "Fresh Mint",
      "quantityPerUnit": 8,
      "totalQuantityUsed": 1200,
      "unit": "Gram",
      "ingredientCogsPerRecipe": 0.33,
      "totalIngredientCogs": 49.50
    },
    {
      "productId": null,
      "productName": "House Syrup Base",
      "quantityPerUnit": 10,
      "totalQuantityUsed": 1500,
      "unit": "Ml",
      "ingredientCogsPerRecipe": 0.04,
      "totalIngredientCogs": 6.00
    }
  ]
}
```

> **Note:** `totalCogs` (363.00) equals `cogsPerUnit` (2.42) * `quantitySold` (150). The `cogsPerUnit` already includes the 3% breakage margin, so the sum of `totalIngredientCogs` across all ingredients (352.50) will be slightly less than `totalCogs`.

### Response Fields

#### Root Level

| Field | Type | Description |
| --- | --- | --- |
| `requestDate` | string | ISO 8601 timestamp of when the request was made |
| `companyId` | string | The company ID associated with the API token |
| `companyName` | string | Human-readable company name |
| `recipe` | object | Summary of the recipe (see below) |
| `quantitySold` | integer | The quantity sold value provided in the request |
| `totalCogs` | number | Total COGS for the quantity sold (`cogsPerUnit * quantitySold`), **including breakage margin** |
| `ingredients` | array | List of ingredient objects with scaled cost breakdown |

#### Recipe Summary Object

| Field | Type | Description |
| --- | --- | --- |
| `recipeId` | string | Unique identifier for this recipe in BarBrain |
| `recipeName` | string | Human-readable recipe name |
| `companyInternalId` | string | The company-defined internal identifier |
| `category` | string | Recipe category: `"Food"`, `"Drink"`, or `"Bundle"` |
| `cogsPerUnit` | number | COGS for a single serving of the recipe, **including 3% breakage margin** |
| `currency` | string | ISO 4217 currency code |

#### Scaled Ingredient Object

| Field | Type | Description |
| --- | --- | --- |
| `productId` | string? | Product identifier in BarBrain. **Null for pre-products** |
| `productName` | string | Product name, or source recipe name for pre-products |
| `quantityPerUnit` | number | Amount of this ingredient per single recipe serving |
| `totalQuantityUsed` | number | Total amount used (`quantityPerUnit * quantitySold`) |
| `unit` | string | Unit of measurement: `"Ml"`, `"Litre"`, `"Gram"`, `"Kilogram"`, `"Piece"`, etc. |
| `ingredientCogsPerRecipe` | number | COGS for this ingredient in a single serving (before breakage) |
| `totalIngredientCogs` | number | Total COGS for this ingredient across all units sold (`ingredientCogsPerRecipe * quantitySold`) |

---

## Batch Fetching Ingredient COGS

Returns COGS broken down per ingredient for **multiple recipes** in a single request. Each recipe is paired with its own `quantitySold`. This is the recommended approach for end-of-day or end-of-shift cost analysis when you have sales data for multiple menu items.

### Endpoint

```bash
POST https://api.barbrain.com/v2-live/fetchBatchRecipeIngredientCogs
```

### Request Body

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `items` | array | Yes | List of recipe + quantity pairs to calculate. Maximum 100 items per request |
| `items[].companyInternalId` | string | Yes | The company-defined internal identifier for the recipe |
| `items[].quantitySold` | integer | Yes | The number of recipe units sold. Must be a positive integer |

### Example Request

**Get ingredient COGS for all cocktails sold in a shift:**

```bash
curl -X POST "https://api.barbrain.com/v2-live/fetchBatchRecipeIngredientCogs" \
  -H "Authorization: Bearer a1b2c3d4e5f6..." \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "companyInternalId": "MOJITO-001", "quantitySold": 45 },
      { "companyInternalId": "MARGARITA-002", "quantitySold": 30 },
      { "companyInternalId": "OLD-FASHIONED-003", "quantitySold": 22 },
      { "companyInternalId": "ESPRESSO-MARTINI-004", "quantitySold": 38 }
    ]
  }'
```

### Success Response (200 OK)

```json
{
  "requestDate": "2026-02-27T23:30:00Z",
  "companyId": "comp-123",
  "companyName": "Tapped Bar Munich",
  "totalItems": 4,
  "totalCogs": 387.92,
  "currency": "EUR",
  "results": [
    {
      "recipe": {
        "recipeId": "rec-456",
        "recipeName": "Classic Mojito",
        "companyInternalId": "MOJITO-001",
        "category": "Drink",
        "cogsPerUnit": 2.42,
        "currency": "EUR"
      },
      "quantitySold": 45,
      "totalCogs": 108.90,
      "ingredients": [
        {
          "productId": "prod-rum-1",
          "productName": "White Rum Premium",
          "quantityPerUnit": 60,
          "totalQuantityUsed": 2700,
          "unit": "Ml",
          "ingredientCogsPerRecipe": 1.50,
          "totalIngredientCogs": 67.50
        },
        {
          "productId": "prod-lime-1",
          "productName": "Fresh Lime Juice",
          "quantityPerUnit": 30,
          "totalQuantityUsed": 1350,
          "unit": "Ml",
          "ingredientCogsPerRecipe": 0.24,
          "totalIngredientCogs": 10.80
        }
      ]
    },
    {
      "recipe": {
        "recipeId": "rec-789",
        "recipeName": "Classic Margarita",
        "companyInternalId": "MARGARITA-002",
        "category": "Drink",
        "cogsPerUnit": 2.78,
        "currency": "EUR"
      },
      "quantitySold": 30,
      "totalCogs": 83.40,
      "ingredients": [
        {
          "productId": "prod-tequila-1",
          "productName": "Tequila Blanco",
          "quantityPerUnit": 50,
          "totalQuantityUsed": 1500,
          "unit": "Ml",
          "ingredientCogsPerRecipe": 1.85,
          "totalIngredientCogs": 55.50
        },
        {
          "productId": "prod-triplesec-1",
          "productName": "Triple Sec",
          "quantityPerUnit": 25,
          "totalQuantityUsed": 750,
          "unit": "Ml",
          "ingredientCogsPerRecipe": 0.52,
          "totalIngredientCogs": 15.60
        }
      ]
    }
  ],
  "errors": []
}
```

### Partial Success — Some Recipes Not Found or Not Calculable

The batch endpoint uses a **best-effort** approach: it returns results for all recipes it can resolve, and reports errors for those it cannot. The HTTP status is still **200 OK** as long as at least one recipe succeeds.

```json
{
  "requestDate": "2026-02-27T23:30:00Z",
  "companyId": "comp-123",
  "companyName": "Tapped Bar Munich",
  "totalItems": 2,
  "totalCogs": 108.90,
  "currency": "EUR",
  "results": [
    {
      "recipe": {
        "recipeId": "rec-456",
        "recipeName": "Classic Mojito",
        "companyInternalId": "MOJITO-001",
        "category": "Drink",
        "cogsPerUnit": 2.42,
        "currency": "EUR"
      },
      "quantitySold": 45,
      "totalCogs": 108.90,
      "ingredients": [ ]
    }
  ],
  "errors": [
    {
      "companyInternalId": "DELETED-RECIPE-999",
      "quantitySold": 10,
      "errorCode": 404,
      "message": "Recipe not found for companyInternalId: DELETED-RECIPE-999"
    },
    {
      "companyInternalId": "INCOMPLETE-RECIPE-888",
      "quantitySold": 5,
      "errorCode": 422,
      "message": "Cannot calculate COGS for recipe: INCOMPLETE-RECIPE-888. Some ingredients are missing pricing data."
    }
  ]
}
```

### Response Fields

#### Root Level

| Field | Type | Description |
| --- | --- | --- |
| `requestDate` | string | ISO 8601 timestamp of when the request was made |
| `companyId` | string | The company ID associated with the API token |
| `companyName` | string | Human-readable company name |
| `totalItems` | integer | Number of successfully resolved recipes in `results` |
| `totalCogs` | number | Grand total COGS across **all** successful results |
| `currency` | string | ISO 4217 currency code |
| `results` | array | List of per-recipe results (same structure as single endpoint response) |
| `errors` | array | List of recipes that could not be resolved or calculated |

#### Result Item

Each item in `results` has the same structure as the single `fetchRecipeIngredientCogs` response body (recipe summary + quantitySold + totalCogs + ingredients). See [Scaled Ingredient Object](#scaled-ingredient-object) for field details.

#### Error Item

| Field | Type | Description |
| --- | --- | --- |
| `companyInternalId` | string | The internal ID that failed |
| `quantitySold` | integer | The quantity that was requested |
| `errorCode` | integer | `404` if recipe not found, `422` if COGS not calculable |
| `message` | string | Human-readable error description |

### Limits

| Constraint | Value |
| --- | --- |
| Maximum items per request | 100 |
| Duplicate `companyInternalId` values | Allowed (each entry is processed independently) |

---

## Error Responses

### 401 Unauthorized

Returned when the API token is invalid, expired, or revoked.

```json
{
  "error": {
    "errorCode": 401,
    "message": "Invalid or expired API token"
  }
}
```

**Possible causes:**

- Token is expired (check expiration date on User's API Token Dashboard)
- Token has been revoked
- Token string is malformed or incorrect
- Missing `Authorization` header

### 400 Bad Request

Returned when required parameters are missing or invalid.

```json
{
  "error": {
    "errorCode": 400,
    "message": "Missing required parameter: companyInternalId"
  }
}
```

```json
{
  "error": {
    "errorCode": 400,
    "message": "quantitySold must be a positive integer"
  }
}
```

**For the batch endpoint:**

```json
{
  "error": {
    "errorCode": 400,
    "message": "Request body must contain an 'items' array with 1-100 entries"
  }
}
```

### 404 Not Found

Returned when no recipe matches the provided `companyInternalId` for the company associated with the token (single endpoints only — the batch endpoint reports 404s in the `errors` array).

```json
{
  "error": {
    "errorCode": 404,
    "message": "Recipe not found for companyInternalId: MOJITO-999"
  }
}
```

**Possible causes:**

- The `companyInternalId` does not match any recipe in your company
- The recipe has been deleted
- The company associated with the token no longer exists

### 422 Unprocessable Entity

Returned when a recipe is found but COGS cannot be calculated because one or more ingredients are missing pricing data (single endpoints only — the batch endpoint reports 422s in the `errors` array).

```json
{
  "error": {
    "errorCode": 422,
    "message": "Cannot calculate COGS for recipe: MOJITO-001. Some ingredients are missing pricing data."
  }
}
```

**Possible causes:**

- An ingredient's product has no wholesale price set
- A pre-product ingredient's source recipe has no yield configured
- Unit conversion is not possible between the recipe unit and the product unit (e.g., mixing volume and weight units)

---

## Code Examples

### Python

```python
import requests

API_TOKEN = "your_api_token_here"
BASE_URL = "https://api.barbrain.com/v2-live"
HEADERS = {"Authorization": f"Bearer {API_TOKEN}"}


def fetch_recipe_cogs(company_internal_id):
    """Fetch total COGS for a single recipe."""
    response = requests.get(
        f"{BASE_URL}/fetchRecipeCogs",
        headers=HEADERS,
        params={"companyInternalId": company_internal_id}
    )
    response.raise_for_status()
    return response.json()


def fetch_recipe_ingredient_cogs(company_internal_id, quantity_sold):
    """Fetch COGS per ingredient, scaled by quantity sold."""
    response = requests.get(
        f"{BASE_URL}/fetchRecipeIngredientCogs",
        headers=HEADERS,
        params={
            "companyInternalId": company_internal_id,
            "quantitySold": quantity_sold
        }
    )
    response.raise_for_status()
    return response.json()


def fetch_batch_recipe_ingredient_cogs(items):
    """Fetch COGS per ingredient for multiple recipes at once.

    Args:
        items: List of dicts with 'companyInternalId' and 'quantitySold'
    """
    response = requests.post(
        f"{BASE_URL}/fetchBatchRecipeIngredientCogs",
        headers={**HEADERS, "Content-Type": "application/json"},
        json={"items": items}
    )
    response.raise_for_status()
    return response.json()


# Example 1: Get COGS for a single recipe
data = fetch_recipe_cogs("MOJITO-001")
recipe = data["recipe"]
print(f"{recipe['recipeName']}: {recipe['totalCogs']} {recipe['currency']} per serving")

# Example 2: Get ingredient COGS for 150 units sold
data = fetch_recipe_ingredient_cogs("MOJITO-001", 150)
print(f"\nTotal COGS for {data['quantitySold']} units: {data['totalCogs']} {data['recipe']['currency']}")
for ing in data["ingredients"]:
    print(f"  - {ing['productName']}: {ing['totalIngredientCogs']:.2f} ({ing['totalQuantityUsed']} {ing['unit']} used)")

# Example 3: End-of-day batch analysis
tonight_sales = [
    {"companyInternalId": "MOJITO-001", "quantitySold": 45},
    {"companyInternalId": "MARGARITA-002", "quantitySold": 30},
    {"companyInternalId": "OLD-FASHIONED-003", "quantitySold": 22},
    {"companyInternalId": "ESPRESSO-MARTINI-004", "quantitySold": 38},
]

data = fetch_batch_recipe_ingredient_cogs(tonight_sales)
print(f"\nTonight's total COGS: {data['totalCogs']} {data['currency']}")
for result in data["results"]:
    print(f"  - {result['recipe']['recipeName']}: {result['totalCogs']:.2f} ({result['quantitySold']} sold)")

if data["errors"]:
    print("\nWarnings:")
    for err in data["errors"]:
        print(f"  - {err['companyInternalId']}: {err['message']}")
```

### JavaScript/Node.js

```javascript
const axios = require('axios');

const API_TOKEN = 'your_api_token_here';
const BASE_URL = 'https://api.barbrain.com/v2-live';
const headers = { 'Authorization': `Bearer ${API_TOKEN}` };

async function fetchRecipeCogs(companyInternalId) {
  const response = await axios.get(`${BASE_URL}/fetchRecipeCogs`, {
    headers,
    params: { companyInternalId }
  });
  return response.data;
}

async function fetchRecipeIngredientCogs(companyInternalId, quantitySold) {
  const response = await axios.get(`${BASE_URL}/fetchRecipeIngredientCogs`, {
    headers,
    params: { companyInternalId, quantitySold }
  });
  return response.data;
}

async function fetchBatchRecipeIngredientCogs(items) {
  const response = await axios.post(`${BASE_URL}/fetchBatchRecipeIngredientCogs`,
    { items },
    { headers: { ...headers, 'Content-Type': 'application/json' } }
  );
  return response.data;
}

// Example: End-of-day batch analysis
const tonightSales = [
  { companyInternalId: 'MOJITO-001', quantitySold: 45 },
  { companyInternalId: 'MARGARITA-002', quantitySold: 30 },
  { companyInternalId: 'OLD-FASHIONED-003', quantitySold: 22 },
  { companyInternalId: 'ESPRESSO-MARTINI-004', quantitySold: 38 }
];

fetchBatchRecipeIngredientCogs(tonightSales)
  .then(data => {
    console.log(`Tonight's total COGS: ${data.totalCogs} ${data.currency}`);
    data.results.forEach(r => {
      console.log(`  - ${r.recipe.recipeName}: ${r.totalCogs.toFixed(2)} (${r.quantitySold} sold)`);
    });
    if (data.errors.length > 0) {
      console.log('\nWarnings:');
      data.errors.forEach(e => console.log(`  - ${e.companyInternalId}: ${e.message}`));
    }
  })
  .catch(err => console.error('Error:', err.message));
```

### Bash

```bash
#!/bin/bash

API_TOKEN="your_api_token_here"

# Fetch recipe COGS (single)
curl -s -X GET \
  "https://api.barbrain.com/v2-live/fetchRecipeCogs?companyInternalId=MOJITO-001" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq .

# Fetch ingredient COGS for 150 units sold (single)
curl -s -X GET \
  "https://api.barbrain.com/v2-live/fetchRecipeIngredientCogs?companyInternalId=MOJITO-001&quantitySold=150" \
  -H "Authorization: Bearer ${API_TOKEN}" | jq .

# Batch: all cocktails sold tonight
curl -s -X POST \
  "https://api.barbrain.com/v2-live/fetchBatchRecipeIngredientCogs" \
  -H "Authorization: Bearer ${API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      { "companyInternalId": "MOJITO-001", "quantitySold": 45 },
      { "companyInternalId": "MARGARITA-002", "quantitySold": 30 },
      { "companyInternalId": "OLD-FASHIONED-003", "quantitySold": 22 },
      { "companyInternalId": "ESPRESSO-MARTINI-004", "quantitySold": 38 }
    ]
  }' | jq .
```

---

## Best Practices

### Using `companyInternalId`

- Assign consistent internal IDs to recipes in BarBrain that match your POS or external system codes
- Internal IDs are scoped to your company — they must be unique within your company but don't need to be globally unique
- If a recipe's internal ID changes in BarBrain, update your integration accordingly

### COGS Accuracy

- COGS values are calculated based on **current ingredient purchase prices** in BarBrain
- A **3% breakage margin** is automatically included in `totalCogs` and `cogsPerUnit`
- If ingredient prices are updated, the COGS returned by the API will reflect the new prices immediately
- For historical cost analysis, store COGS snapshots at the time of sale rather than re-fetching retroactively

### Choosing the Right Endpoint

| Scenario | Endpoint |
| --- | --- |
| Display COGS on a menu dashboard | `GET /fetchRecipeCogs` |
| Cost analysis for a single item's sales | `GET /fetchRecipeIngredientCogs` |
| End-of-day/shift cost report for all items sold | `POST /fetchBatchRecipeIngredientCogs` |

### Batch Endpoint Tips

- Use the batch endpoint instead of looping single calls — it's faster and counts as one rate-limited request
- Handle partial failures: always check the `errors` array even on 200 responses
- Keep batch size under 100 items; split larger requests into multiple batches

### Error Handling

- Validate `companyInternalId` values before making API calls
- Handle 404 responses gracefully — a missing recipe may indicate an ID mismatch between your systems
- Handle 422 responses by checking that all ingredients in the recipe have pricing data in BarBrain
- Implement retry logic with exponential backoff for transient errors

---

## Support

For issues with the Recipe COGS API:

- Verify the `companyInternalId` matches a recipe in your BarBrain account
- Check token expiration and status
- Review error response messages
- Contact BarBrain support with your `requestDate` for troubleshooting
