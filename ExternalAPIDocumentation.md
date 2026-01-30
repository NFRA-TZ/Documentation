# NFRA DFTS External API Documentation

**Version:** 1.0  
**Last Updated:** January 30, 2026  
**Status:** Production

---

## Table of Contents

1. [API Overview](#api-overview)
2. [Authentication](#authentication)
3. [Request Rules & Validation](#request-rules--validation)
4. [Storage Reports](#storage-reports)
5. [Purchase Reports](#purchase-reports)
6. [Data Models](#data-models)
7. [Season & Date Logic](#season--date-logic)
8. [Error Handling](#error-handling)
9. [Security Best Practices](#security-best-practices)
10. [Versioning](#versioning)
11. [Support](#support)

---

## API Overview

The NFRA DFTS External API provides secure access to warehouse inventory and purchase reporting data. This API is designed for external system integrators, backend developers, and authorized data consumers who need real-time access to storage and procurement analytics.

### Base URL

```
https://{domain}/api/external
```

Replace `{domain}` with your actual DFTS deployment domain.

### Key Features

- **Multi-tier Reporting**: Warehouse → Zone → Cumulative inventory aggregation
- **Purchase Analytics**: Track purchases by zone with product-level breakdown
- **Flexible Querying**: Query by season, quarter, specific date, or current status
- **Comprehensive Coverage**: All products and zones included (even with zero values)
- **Secure Access**: API Key-based authentication on all endpoints

---

## Authentication

All endpoints require API Key authentication. API keys are issued by the NFRA DFTS administrator and must be included with every request.

### How to Authenticate

Add the `X-API-KEY` header to every HTTP request:

```
X-API-KEY: {your_api_key_here}
```

### Example: cURL

```bash
curl -X POST https://{domain}/api/external/reports/warehouses \
  -H "X-API-KEY: your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{"season":"2025/2026"}'
```

### Example: Python Requests

```python
import requests

headers = {
    "X-API-KEY": "your_api_key_here",
    "Content-Type": "application/json"
}

url = "https://{domain}/api/external/reports/warehouses"
payload = {"season": "2025/2026"}

response = requests.post(url, json=payload, headers=headers)
print(response.json())
```

### Example: JavaScript Fetch

```javascript
const apiKey = "your_api_key_here";
const url = "https://{domain}/api/external/reports/warehouses";

const payload = { season: "2025/2026" };

fetch(url, {
  method: "POST",
  headers: {
    "X-API-KEY": apiKey,
    "Content-Type": "application/json"
  },
  body: JSON.stringify(payload)
})
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error("Error:", error));
```

### Authentication Failure Responses

**401 Unauthorized** - API key is missing or invalid:

```json
{
  "status": 401,
  "message": "Unauthorized",
  "detail": "API key is missing or invalid"
}
```

**403 Forbidden** - API key is valid but access to this resource is denied:

```json
{
  "status": 403,
  "message": "Forbidden",
  "detail": "Your API key does not have permission to access this resource"
}
```

### How to Obtain an API Key

1. Contact the NFRA DFTS administrator
2. Provide your organization name and intended use case
3. The administrator will issue a unique API key
4. Store the key securely (use environment variables, secrets management tools)
5. Never commit API keys to version control systems

---

## Request Rules & Validation

All API requests follow standard conventions and validation rules.

### Common Headers

| Header | Value | Required |
|--------|-------|----------|
| `X-API-KEY` | Your API key string | Yes |
| `Content-Type` | `application/json` | Yes (POST requests) |

### Request Format

All POST requests accept JSON bodies. Use `Content-Type: application/json` header.

### Parameter Validation Rules

| Parameter | Format | Example | Notes |
|-----------|--------|---------|-------|
| `season` | YYYY/YYYY | `2025/2026` | Represents July 1 - June 30 fiscal year |
| `quarter` | Q1, Q2, Q3, Q4 | `Q1` | Optional; if provided, filters to quarter dates |
| `upToDate` | YYYY-MM-DD | `2025-10-15` | Optional; overrides quarter if provided |
| `warehouseIds` | Integer array | `[1, 2, 3]` | Optional; filters to specified warehouses |
| `zoneIds` | Integer array | `[1, 2, 3]` | Optional; filters to specified zones |

### Validation Error Response

```json
{
  "status": 400,
  "message": "Bad Request",
  "detail": "Season must be in format 'YYYY/YYYY' (e.g., '2025/2026')"
}
```

---

## Storage Reports

Storage reports provide inventory data aggregated at multiple levels: warehouse, zone, and system-wide cumulative.

### 1.1 Generate Warehouse Report

**Endpoint:** `POST /reports/warehouses`

**Description:** Returns inventory data grouped by warehouse with cumulative weights and all products per warehouse.

**Request Body:**

```json
{
  "season": "2025/2026",
  "quarter": "Q1",
  "warehouseIds": [1, 2, 3]
}
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY (e.g., "2025/2026") |
| `quarter` | string | No | Quarter: Q1, Q2, Q3, or Q4 |
| `upToDate` | string | No | Specific date in YYYY-MM-DD format (overrides quarter) |
| `warehouseIds` | array | No | Filter to specific warehouse IDs |

**Response Example (200 OK):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "reportingPeriod": "Q1 (Up to September 30)",
  "upToDate": "2025-09-30T00:00:00Z",
  "warehouses": [
    {
      "warehouseId": 1,
      "warehouseName": "Central Storage",
      "centerId": 1,
      "centerName": "Main Center",
      "zoneId": 1,
      "zoneName": "Zone A",
      "products": [
        {
          "productId": 101,
          "productName": "Maize",
          "cumulativeWeightKg": 5000.50,
          "totalBags": 250
        },
        {
          "productId": 102,
          "productName": "Wheat",
          "cumulativeWeightKg": 3200.75,
          "totalBags": 160
        }
      ],
      "totalCumulativeWeightKg": 8201.25,
      "totalBags": 410
    }
  ],
  "grandTotalWeightKg": 8201.25,
  "grandTotalBags": 410,
  "generatedAt": "2026-01-30T14:30:00Z"
}
```

**Response Fields:**

- `season`: Season identifier (string)
- `seasonPeriod`: Human-readable season date range (string)
- `reportingPeriod`: Reporting period description (string)
- `upToDate`: Cutoff date for the report (ISO 8601 datetime)
- `warehouses`: Array of warehouse summaries
- `grandTotalWeightKg`: Total inventory weight in kilograms across all warehouses (double, 2 decimals)
- `grandTotalBags`: Total bags across all warehouses (integer)
- `generatedAt`: Report generation timestamp (ISO 8601 datetime)

**Status Codes:**

| Code | Meaning | Notes |
|------|---------|-------|
| 200 | Success | Report generated successfully |
| 400 | Bad Request | Invalid season format or parameters |
| 401 | Unauthorized | Missing or invalid API key |
| 403 | Forbidden | Insufficient permissions |
| 500 | Internal Server Error | Server error during report generation |

---

### 1.2 Get Current Warehouse Status

**Endpoint:** `GET /reports/warehouses/current`

**Description:** Returns current warehouse inventory status (data up to today) without quarter filtering.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY |

**Request Example:**

```
GET /reports/warehouses/current?season=2025/2026
```

**Response Example (200 OK):**

Same structure as [1.1 Generate Warehouse Report](#11-generate-warehouse-report), with `reportingPeriod: "Current Status (as of YYYY-MM-DD)"`.

---

### 1.3 Get Quarterly Warehouse Report

**Endpoint:** `GET /reports/warehouses/quarterly`

**Description:** Returns warehouse inventory for a specific quarter.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY |
| `quarter` | string | Yes | Quarter: Q1, Q2, Q3, or Q4 |

**Request Example:**

```
GET /reports/warehouses/quarterly?season=2025/2026&quarter=Q2
```

**Quarter Cutoff Dates:**

| Quarter | End Date | Description |
|---------|----------|-------------|
| Q1 | September 30 | Jul 1 - Sep 30 |
| Q2 | December 31 | Jul 1 - Dec 31 |
| Q3 | March 31 | Jul 1 - Mar 31 |
| Q4 | June 30 | Jul 1 - Jun 30 (Full Year) |

---

### 1.4 Generate Zone Report

**Endpoint:** `POST /reports/zones`

**Description:** Returns inventory data grouped by zone with cumulative weights and all products per zone.

**Request Body:**

```json
{
  "season": "2025/2026",
  "quarter": "Q1",
  "zoneIds": [1, 2, 3]
}
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY |
| `quarter` | string | No | Quarter: Q1, Q2, Q3, or Q4 |
| `upToDate` | string | No | Specific date in YYYY-MM-DD format |
| `zoneIds` | array | No | Filter to specific zone IDs |

**Response Example (200 OK):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "reportingPeriod": "Q1 (Up to September 30)",
  "upToDate": "2025-09-30T00:00:00Z",
  "zones": [
    {
      "zoneId": 1,
      "zoneName": "Zone A",
      "products": [
        {
          "productId": 101,
          "productName": "Maize",
          "cumulativeWeightKg": 5000.50,
          "totalBags": 250
        }
      ],
      "totalCumulativeWeightKg": 5000.50,
      "totalBags": 250
    }
  ],
  "grandTotalWeightKg": 5000.50,
  "grandTotalBags": 250,
  "generatedAt": "2026-01-30T14:30:00Z"
}
```

---

### 1.5 Get Current Zone Status

**Endpoint:** `GET /reports/zones/current`

**Description:** Returns current zone inventory status.

**Query Parameters:**

| Parameter | Type | Required |
|-----------|------|----------|
| `season` | string | Yes |

**Request Example:**

```
GET /reports/zones/current?season=2025/2026
```

---

### 1.6 Get Quarterly Zone Report

**Endpoint:** `GET /reports/zones/quarterly`

**Description:** Returns zone inventory for a specific quarter.

**Query Parameters:**

| Parameter | Type | Required |
|-----------|------|----------|
| `season` | string | Yes |
| `quarter` | string | Yes |

**Request Example:**

```
GET /reports/zones/quarterly?season=2025/2026&quarter=Q3
```

---

### 1.7 Generate Cumulative Storage Report

**Endpoint:** `POST /reports/cumulative`

**Description:** Returns system-wide inventory totals across ALL zones and warehouses, with all products included (even zero values).

**Request Body:**

```json
{
  "season": "2025/2026",
  "quarter": "Q1"
}
```

**Response Example (200 OK):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "reportingPeriod": "Q1 (Up to September 30)",
  "upToDate": "2025-09-30T00:00:00Z",
  "summary": {
    "products": [
      {
        "productId": 101,
        "productName": "Maize",
        "cumulativeWeightKg": 15000.00,
        "totalBags": 750
      },
      {
        "productId": 102,
        "productName": "Wheat",
        "cumulativeWeightKg": 8000.50,
        "totalBags": 400
      },
      {
        "productId": 103,
        "productName": "Rice",
        "cumulativeWeightKg": 0.00,
        "totalBags": 0
      }
    ],
    "totalCumulativeWeightKg": 23000.50,
    "totalBags": 1150
  },
  "generatedAt": "2026-01-30T14:30:00Z"
}
```

---

### 1.8 Get Current Cumulative Storage Status

**Endpoint:** `GET /reports/cumulative/current`

**Description:** Returns current cumulative inventory status across all zones.

**Query Parameters:**

| Parameter | Type | Required |
|-----------|------|----------|
| `season` | string | Yes |

**Request Example:**

```
GET /reports/cumulative/current?season=2025/2026
```

---

### 1.9 Get Quarterly Cumulative Storage Report

**Endpoint:** `GET /reports/cumulative/quarterly`

**Description:** Returns cumulative inventory for a specific quarter across all zones.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY |
| `quarter` | string | Yes | Quarter: Q1, Q2, Q3, or Q4 |

**Request Example:**

```
GET /reports/cumulative/quarterly?season=2025/2026&quarter=Q4
```

**Response Example (200 OK):**

Same structure as [1.7 Generate Cumulative Storage Report](#17-generate-cumulative-storage-report), with quarter-filtered data.

**Status Codes:**

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 500 | Internal Server Error |

---

## Purchase Reports

Purchase reports track goods received from suppliers, organized by zone or aggregated across all zones.

### 2.1 Purchase Report by Zone

**Endpoint:** `POST /reports/purchases`

**Description:** Returns purchases grouped by zone with product-level breakdown. Includes all zones and products, even those with zero purchases. Useful for tracking procurement by distribution zone.

**Request Body:**

```json
{
  "season": "2025/2026",
  "zoneIds": [1, 2, 3]
}
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY (e.g., "2025/2026") |
| `zoneIds` | array | No | Filter to specific zone IDs (leave empty for all zones) |

**Response Example (200 OK):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "upToDate": "2026-06-30T00:00:00Z",
  "zones": [
    {
      "zoneId": 1,
      "zoneName": "Zone A",
      "products": [
        {
          "productId": 101,
          "productName": "Maize",
          "totalNetWeightKg": 12500.00,
          "totalBags": 625
        },
        {
          "productId": 102,
          "productName": "Wheat",
          "totalNetWeightKg": 8000.75,
          "totalBags": 400
        }
      ],
      "totalNetWeightKg": 20500.75,
      "totalBags": 1025
    },
    {
      "zoneId": 2,
      "zoneName": "Zone B",
      "products": [
        {
          "productId": 101,
          "productName": "Maize",
          "totalNetWeightKg": 5000.00,
          "totalBags": 250
        },
        {
          "productId": 102,
          "productName": "Wheat",
          "totalNetWeightKg": 0.00,
          "totalBags": 0
        }
      ],
      "totalNetWeightKg": 5000.00,
      "totalBags": 250
    }
  ],
  "grandTotalNetWeightKg": 25500.75,
  "grandTotalBags": 1275,
  "generatedAt": "2026-01-30T14:31:12Z"
}
```

**Response Fields:**

- `season`: Season identifier (string)
- `seasonPeriod`: Human-readable season date range (string)
- `upToDate`: Season end date (ISO 8601 datetime)
- `zones`: Array of zone purchase summaries
  - `zoneId`: Zone identifier (integer)
  - `zoneName`: Zone name (string)
  - `products`: Array of products with purchase totals (all products included)
    - `productId`: Product identifier (integer)
    - `productName`: Product name (string)
    - `totalNetWeightKg`: Total weight in kilograms (double, 2 decimals)
    - `totalBags`: Total bags purchased (integer)
  - `totalNetWeightKg`: Total purchases for this zone in kilograms (double, 2 decimals)
  - `totalBags`: Total bags for this zone (integer)
- `grandTotalNetWeightKg`: Total purchases across all zones in kilograms (double, 2 decimals)
- `grandTotalBags`: Total bags across all zones (integer)
- `generatedAt`: Report generation timestamp (ISO 8601 datetime)

**Status Codes:**

| Code | Meaning | Notes |
|------|---------|-------|
| 200 | Success | Report generated successfully |
| 400 | Bad Request | Invalid season format or parameters |
| 401 | Unauthorized | Missing or invalid API key |
| 403 | Forbidden | Insufficient permissions |
| 500 | Internal Server Error | Server error during report generation |

---

### 2.2 Cumulative Purchase Report

**Endpoint:** `POST /reports/purchases/cumulative`

**Description:** Returns aggregated purchases across ALL zones. Shows total purchases by product for the entire season. Includes all products, even those with zero purchases.

**Request Body:**

```json
{
  "season": "2025/2026"
}
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `season` | string | Yes | Season in format YYYY/YYYY |

**Response Example (200 OK):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "upToDate": "2026-06-30T00:00:00Z",
  "summary": {
    "products": [
      {
        "productId": 101,
        "productName": "Maize",
        "totalNetWeightKg": 25500.00,
        "totalBags": 1275
      },
      {
        "productId": 102,
        "productName": "Wheat",
        "totalNetWeightKg": 15000.75,
        "totalBags": 750
      },
      {
        "productId": 103,
        "productName": "Rice",
        "totalNetWeightKg": 0.00,
        "totalBags": 0
      }
    ],
    "totalNetWeightKg": 40500.75,
    "totalBags": 2025
  },
  "generatedAt": "2026-01-30T14:31:45Z"
}
```

**Response Fields:**

- `season`: Season identifier (string)
- `seasonPeriod`: Human-readable season period (string)
- `upToDate`: Season end date (ISO 8601 datetime)
- `summary`: Summary of all purchases across all zones
  - `products`: Array of all products with totals
    - `productId`: Product identifier (integer)
    - `productName`: Product name (string)
    - `totalNetWeightKg`: Total weight in kilograms (double, 2 decimals)
    - `totalBags`: Total bags purchased (integer)
  - `totalNetWeightKg`: Total weight in kilograms across all zones (double, 2 decimals)
  - `totalBags`: Total bags across all zones (integer)
- `generatedAt`: Report generation timestamp (ISO 8601 datetime)

**Status Codes:**

| Code | Meaning | Notes |
|------|---------|-------|
| 200 | Success | Report generated successfully |
| 400 | Bad Request | Invalid season format or parameters |
| 401 | Unauthorized | Missing or invalid API key |
| 403 | Forbidden | Insufficient permissions |
| 500 | Internal Server Error | Server error during report generation |

---

## Data Models

### PurchaseProductSummaryDto

Represents a single product's purchase data within a zone or cumulative report.

```json
{
  "productId": 101,
  "productName": "Maize",
  "totalNetWeightKg": 12500.00,
  "totalBags": 625
}
```

| Field | Type | Description |
|-------|------|-------------|
| `productId` | integer | Unique product identifier |
| `productName` | string | Human-readable product name |
| `totalNetWeightKg` | double | Total weight in kilograms (2 decimal precision) |
| `totalBags` | integer | Total bags purchased (count) |

### CumulativePurchaseSummaryDto

Represents the cumulative summary of all purchases across all zones.

```json
{
  "products": [
    {
      "productId": 101,
      "productName": "Maize",
      "totalNetWeightKg": 25500.00,
      "totalBags": 1275
    }
  ],
  "totalNetWeightKg": 40500.75,
  "totalBags": 2025
}
```

| Field | Type | Description |
|-------|------|-------------|
| `products` | array | Array of product summaries with totals |
| `totalNetWeightKg` | double | Sum of all product weights (2 decimal precision) |
| `totalBags` | integer | Sum of all bags |

### CumulativePurchaseReportDto

Complete cumulative purchase report response.

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "upToDate": "2026-06-30T00:00:00Z",
  "summary": { ... },
  "generatedAt": "2026-01-30T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `season` | string | Season code (e.g., "2025/2026") |
| `seasonPeriod` | string | Human-readable season date range |
| `upToDate` | datetime | Report data cutoff date (ISO 8601) |
| `summary` | object | CumulativePurchaseSummaryDto |
| `generatedAt` | datetime | Timestamp when report was generated (ISO 8601) |

### PurchaseZoneSummaryDto

Represents purchase data for a single zone.

```json
{
  "zoneId": 1,
  "zoneName": "Zone A",
  "products": [ ... ],
  "totalNetWeightKg": 20500.75,
  "totalBags": 1025
}
```

| Field | Type | Description |
|-------|------|-------------|
| `zoneId` | integer | Unique zone identifier |
| `zoneName` | string | Human-readable zone name |
| `products` | array | Array of PurchaseProductSummaryDto |
| `totalNetWeightKg` | double | Total purchases for this zone (2 decimal precision) |
| `totalBags` | integer | Total bags for this zone |

### PurchaseReportDto

Complete purchase report response (by zone).

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "upToDate": "2026-06-30T00:00:00Z",
  "zones": [ ... ],
  "grandTotalNetWeightKg": 25500.75,
  "grandTotalBags": 1275,
  "generatedAt": "2026-01-30T14:30:00Z"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `season` | string | Season code |
| `seasonPeriod` | string | Human-readable season range |
| `upToDate` | datetime | Report cutoff date |
| `zones` | array | Array of PurchaseZoneSummaryDto |
| `grandTotalNetWeightKg` | double | Total purchases across all zones (2 decimals) |
| `grandTotalBags` | integer | Total bags across all zones |
| `generatedAt` | datetime | Timestamp when report was generated |

---

## Season & Date Logic

### Season Format

All seasons follow the **fiscal year** format from July 1 to June 30:

- **Season Code:** `YYYY/YYYY` (e.g., `2025/2026`)
- **Meaning:** July 1, 2025 – June 30, 2026
- **Storage:** Stored as string in database for data consistency

### Season Examples

| Season Code | Period |
|-------------|--------|
| `2024/2025` | July 1, 2024 – June 30, 2025 |
| `2025/2026` | July 1, 2025 – June 30, 2026 |
| `2026/2027` | July 1, 2026 – June 30, 2027 |

### Quarter Dates

Quarters are calculated from the season start date (July 1):

| Quarter | Cutoff Date | Period |
|---------|-------------|--------|
| Q1 | September 30 | July 1 – September 30 (3 months) |
| Q2 | December 31 | July 1 – December 31 (6 months) |
| Q3 | March 31 | July 1 – March 31 (9 months) |
| Q4 | June 30 | July 1 – June 30 (12 months, Full Year) |

### UpToDate Field

The `upToDate` field in responses indicates the **data cutoff date**:

- Shows the **end date** of the reporting period
- In **ISO 8601 format** with time set to 00:00:00Z
- Example: `2025-09-30T00:00:00Z` (for Q1)

### GeneratedAt Field

The `generatedAt` field shows when the report was **actually generated**:

- Uses **current UTC timestamp**
- In **ISO 8601 format**
- Example: `2026-01-30T14:35:22Z`

---

## Error Handling

### Standard HTTP Status Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request successful, data returned |
| 400 | Bad Request | Invalid request parameters or malformed JSON |
| 401 | Unauthorized | Missing or invalid API key |
| 403 | Forbidden | API key valid but lacks permission for resource |
| 500 | Internal Server Error | Server error during processing |

### Error Response Format

All error responses follow this standard format:

```json
{
  "status": 400,
  "message": "Bad Request",
  "detail": "Specific error message describing what went wrong"
}
```

### Common Error Scenarios

#### 400 - Invalid Season Format

```json
{
  "status": 400,
  "message": "Bad Request",
  "detail": "Season must be in format 'YYYY/YYYY' (e.g., '2025/2026')"
}
```

**Cause:** Season parameter not in correct format.  
**Fix:** Use format `YYYY/YYYY` (e.g., `2025/2026`).

#### 400 - Invalid Quarter

```json
{
  "status": 400,
  "message": "Bad Request",
  "detail": "Quarter must be Q1, Q2, Q3, or Q4"
}
```

**Cause:** Quarter parameter has invalid value.  
**Fix:** Use one of: `Q1`, `Q2`, `Q3`, or `Q4`.

#### 400 - Invalid Date Format

```json
{
  "status": 400,
  "message": "Bad Request",
  "detail": "UpToDate must be in format 'YYYY-MM-DD'"
}
```

**Cause:** Date parameter not in ISO 8601 date format.  
**Fix:** Use format `YYYY-MM-DD` (e.g., `2025-10-15`).

#### 401 - Missing API Key

```json
{
  "status": 401,
  "message": "Unauthorized",
  "detail": "API key is missing or invalid"
}
```

**Cause:** No `X-API-KEY` header in request.  
**Fix:** Add `X-API-KEY: {your_api_key}` header.

#### 401 - Invalid API Key

```json
{
  "status": 401,
  "message": "Unauthorized",
  "detail": "API key is missing or invalid"
}
```

**Cause:** API key value is incorrect or has been revoked.  
**Fix:** Verify API key; contact support if revoked.

#### 403 - Insufficient Permissions

```json
{
  "status": 403,
  "message": "Forbidden",
  "detail": "Your API key does not have permission to access this resource"
}
```

**Cause:** API key is valid but not authorized for this endpoint.  
**Fix:** Contact NFRA DFTS administrator to request access.

#### 500 - Server Error

```json
{
  "status": 500,
  "message": "Internal Server Error",
  "detail": "An unexpected error occurred while processing your request"
}
```

**Cause:** Server-side error (database connection, processing failure, etc.).  
**Fix:** Retry the request. If issue persists, contact support.

---

## Security Best Practices

### API Key Protection

1. **Never Expose in Client Code**
   - Do NOT hardcode API keys in frontend JavaScript, mobile apps, or any client-side code
   - Always use API keys only on secure backend servers

2. **Use Environment Variables**
   - Store API keys in environment variables or secrets management systems
   - Example (.env file):
     ```
     NFRA_DFTS_API_KEY=your_api_key_here
     ```

3. **Rotate Keys Regularly**
   - Request key rotation from NFRA DFTS administrator every 6-12 months
   - Revoke and replace compromised keys immediately

4. **Secure Storage**
   - Use password managers or secrets vaults (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault)
   - Never commit secrets to version control systems (Git, GitHub, etc.)

### API Request Security

1. **Always Use HTTPS**
   - All API requests must use `https://` (never `http://`)
   - Encryption protects API keys and data in transit

2. **Verify SSL Certificates**
   - Validate SSL/TLS certificates on the server
   - Enable certificate pinning in mobile applications for extra security

3. **Request Rate Limiting** (Recommended)
   - The API may implement rate limiting to prevent abuse
   - Implement exponential backoff for rate-limited responses (429 status code)
   - Do not hammer the API with rapid requests

4. **Optional: IP Whitelisting**
   - Contact NFRA DFTS administrator to request IP whitelisting
   - Only requests from whitelisted IP addresses will be accepted
   - Useful for backend-to-backend integration

### Data Handling

1. **Secure Data in Transit**
   - All API requests/responses are encrypted over HTTPS
   - Do not log API keys or sensitive data

2. **Secure Data at Rest**
   - Store API keys securely in your application
   - Use encrypted configuration files
   - Restrict database access to authorized personnel only

3. **Audit Logging**
   - Log API requests for audit and debugging purposes
   - Do not log API keys, only request endpoints and parameters
   - Retain logs for minimum 30 days

4. **Access Control**
   - Limit who has access to API keys in your organization
   - Use role-based access control (RBAC) for sensitive data
   - Implement principle of least privilege

---

## Versioning

### Current Version

**API Version:** `v1`  
**Release Date:** January 30, 2026  
**Status:** Stable, Production-Ready

### Versioning Strategy

The NFRA DFTS API uses semantic versioning with the following approach:

- **Major Version (v1, v2):** Breaking changes to endpoints, data models, or authentication
- **Minor Version (1.1, 1.2):** New features or non-breaking enhancements
- **Patch Version (1.0.1):** Bug fixes, performance improvements

### Accessing Specific Versions

All endpoints are currently on `/api/external` (v1):

```
https://{domain}/api/external/reports/warehouses
```

Future major versions will use version prefixes:

```
https://{domain}/api/v2/reports/warehouses
```

### Deprecation Policy

- When a feature is deprecated, we will announce it with a **6-month notice**
- Deprecated endpoints remain functional during the deprecation period
- A deprecation header will be included: `Deprecation: true` or `Sunset: <date>`
- Migration guides will be provided for new implementations

### Version Support

| Version | Status | Support Until | Notes |
|---------|--------|---------------|-------|
| v1 (current) | Active | TBD | Production-ready |

---

## Support

### Getting Help

If you encounter issues or have questions about the API:

### Technical Support

**Email:** `api-support@nfra.example.com`  
**Response Time:** Business hours (24-48 hours)  
**Available:** Monday – Friday, 9:00 AM – 5:00 PM UTC

### Report Issues

When reporting an issue, include:

1. **API Key (masked):** First 8 characters only (e.g., `abc12345...`)
2. **Endpoint:** Which endpoint you were calling
3. **Request:** Full request body (excluding sensitive data)
4. **Error Response:** Exact error message received
5. **Timestamp:** When the error occurred (include timezone)
6. **Reproducibility:** Steps to reproduce the issue

### API Governance

**API Owner:** NFRA DFTS Team  
**Documentation Maintainer:** Technical Documentation Team  
**Security Contact:** `security@nfra.example.com`

### Feedback & Feature Requests

We welcome feedback and feature requests:

- **Email:** `api-feedback@nfra.example.com`
- **Submit:** Use the feedback form in your NFRA DFTS admin portal

---

## Appendix: Complete Request/Response Examples

### Example 1: Get Current Warehouse Status

**Request:**

```bash
curl -X GET \
  "https://dfts.nfra.org/api/external/reports/warehouses/current?season=2025/2026" \
  -H "X-API-KEY: your_api_key_here" \
  -H "Content-Type: application/json"
```

**Response (200):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "reportingPeriod": "Current Status (as of 2026-01-30)",
  "upToDate": "2026-01-30T00:00:00Z",
  "warehouses": [
    {
      "warehouseId": 1,
      "warehouseName": "Central Storage",
      "centerId": 1,
      "centerName": "Main Center",
      "zoneId": 1,
      "zoneName": "Zone A",
      "products": [
        {
          "productId": 101,
          "productName": "Maize",
          "cumulativeWeightKg": 5000.50,
          "totalBags": 250
        }
      ],
      "totalCumulativeWeightKg": 5000.50,
      "totalBags": 250
    }
  ],
  "grandTotalWeightKg": 5000.50,
  "grandTotalBags": 250,
  "generatedAt": "2026-01-30T14:30:45Z"
}
```

### Example 2: Get Zone Purchases by Season

**Request:**

```bash
curl -X POST \
  "https://dfts.nfra.org/api/external/reports/purchases" \
  -H "X-API-KEY: your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "season": "2025/2026"
  }'
```

**Response (200):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "upToDate": "2026-06-30T00:00:00Z",
  "zones": [
    {
      "zoneId": 1,
      "zoneName": "Zone A",
      "products": [
        {
          "productId": 101,
          "productName": "Maize",
          "totalNetWeightKg": 12500.00,
          "totalBags": 625
        },
        {
          "productId": 102,
          "productName": "Wheat",
          "totalNetWeightKg": 8000.75,
          "totalBags": 400
        }
      ],
      "totalNetWeightKg": 20500.75,
      "totalBags": 1025
    }
  ],
  "grandTotalNetWeightKg": 20500.75,
  "grandTotalBags": 1025,
  "generatedAt": "2026-01-30T14:31:12Z"
}
```

### Example 3: Get Cumulative Purchases

**Request:**

```bash
curl -X POST \
  "https://dfts.nfra.org/api/external/reports/purchases/cumulative" \
  -H "X-API-KEY: your_api_key_here" \
  -H "Content-Type: application/json" \
  -d '{
    "season": "2025/2026"
  }'
```

**Response (200):**

```json
{
  "season": "2025/2026",
  "seasonPeriod": "July 1, 2025 - June 30, 2026",
  "upToDate": "2026-06-30T00:00:00Z",
  "summary": {
    "products": [
      {
        "productId": 101,
        "productName": "Maize",
        "totalNetWeightKg": 25500.00,
        "totalBags": 1275
      },
      {
        "productId": 102,
        "productName": "Wheat",
        "totalNetWeightKg": 15000.75,
        "totalBags": 750
      }
    ],
    "totalNetWeightKg": 40500.75,
    "totalBags": 2025
  },
  "generatedAt": "2026-01-30T14:31:45Z"
}
```

---

## Document Metadata

| Property | Value |
|----------|-------|
| **Title** | NFRA DFTS External API Documentation |
| **Version** | 1.0 |
| **Date** | January 30, 2026 |
| **Status** | Production |
| **Language** | English |
| **Format** | Markdown |

---

**End of Documentation**
