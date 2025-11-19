# ZSend Wash API Documentation

## Overview

The ZSend Wash API provides a RESTful interface for managing private cryptocurrency wash operations. The API allows you to create wash operations, check their status, get quotes, and manage API keys.

**Base URL:** `https://api.zsend.app/api/v1`

**API Version:** v1

**Content-Type:** `application/json`

---

## Table of Contents

1. [Authentication](#authentication)
2. [Endpoints](#endpoints)
   - [Health Check](#health-check)
   - [Get Statistics](#get-statistics)
   - [Get Quote](#get-quote)
   - [Create Wash](#create-wash)
   - [Get Wash Status](#get-wash-status)
   - [List Washes](#list-washes)
3. [Error Handling](#error-handling)
4. [Code Examples](#code-examples)
5. [Rate Limiting](#rate-limiting)

---

## Authentication

All wash operation endpoints require API key authentication. Include your API key in the request header:

```
X-API-Key: your-api-key-here
```

**Note:** API keys should be kept secure and never exposed in client-side code or public repositories.

---

## Endpoints

### Health Check

Check if the API is running and healthy.

**Endpoint:** `GET /api/v1/health`

**Authentication:** Not required

**Response:**
```json
{
  "success": true,
  "status": "healthy",
  "timestamp": "2024-01-15T10:30:00.000000"
}
```

**Status Code:** `200 OK`

---

### Get Statistics

Get public statistics for the wash service. This endpoint does not require authentication and is designed for public display on websites.

**Endpoint:** `GET /api/v1/stats`

**Authentication:** Not required

**Example Request:**
```bash
curl -X GET "https://api.zsend.app/api/v1/stats"
```

**Response:**
```json
{
  "success": true,
  "stats": {
    "volume": {
      "current": 21220.27,
      "total": 2270432.23
    },
    "swaps": {
      "current": 12376,
      "total": 834514
    },
    "period": {
      "start": "2024-01-08T10:30:00.000000",
      "end": "2024-01-15T10:30:00.000000",
      "days": 7
    },
    "last_updated": "2024-01-15T10:30:00.000000"
  }
}
```

**Response Fields:**
| Field | Type | Description |
|-------|------|-------------|
| volume.current | float | Total volume for the current period (last 7 days) |
| volume.total | float | Total volume (all time) |
| swaps.current | integer | Number of completed swaps in the current period (last 7 days) |
| swaps.total | integer | Total number of completed swaps (all time) |
| period.start | string | Start date of the current period (ISO format) |
| period.end | string | End date of the current period (ISO format) |
| period.days | integer | Number of days in the current period |
| last_updated | string | Timestamp when statistics were last updated (ISO format) |

**Status Codes:**
- `200 OK` - Statistics retrieved successfully
- `500 Internal Server Error` - Server error

---

### Get Quote

Get a quote for a wash operation. The quote includes the fee breakdown and expected amount received.

**Endpoint:** `GET /api/v1/wash/quote`

**Authentication:** Required

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| amount | float | Yes | Amount to wash |

**Example Request:**
```bash
curl -X GET "https://api.zsend.app/api/v1/wash/quote?amount=1.5" \
  -H "X-API-Key: your-api-key-here"
```

**Response:**
```json
{
  "success": true,
  "quote": {
    "amount_in": 1.5,
    "fee_percent": 5.0,
    "fee_amount": 0.075,
    "amount_received": 1.425,
    "currency": "SOL",  # Currency identifier
    "quote_timestamp": "2024-01-15T10:30:00.000000"
  }
}
```

**Status Codes:**
- `200 OK` - Quote generated successfully
- `400 Bad Request` - Missing or invalid amount parameter
- `401 Unauthorized` - Invalid or missing API key
- `500 Internal Server Error` - Server error

**Error Response Example:**
```json
{
  "success": false,
  "error": "Missing parameter",
  "message": "amount parameter is required"
}
```

---

### Create Wash

Create a new wash operation. This will create a wash record in "awaiting" status, waiting for the deposit.

**Endpoint:** `POST /api/v1/wash/create`

**Authentication:** Required

**Request Body:**
```json
{
    "destination_wallet": "YourDestinationAddressHere",
    "amount": "1.5"
}
```

**Request Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| destination_wallet | string | Yes | Wallet address that will receive the final output |
| amount | string/float | Yes | Amount to wash (must be positive) |

**Note:** A unique deposit wallet (`washer_wallet`) is automatically generated for each wash operation and returned in the response. You must send the SOL to this generated address to initiate the wash.

**Example Request:**
```bash
curl -X POST "https://api.zsend.app/api/v1/wash/create" \
  -H "X-API-Key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "destination_wallet": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "amount": "1.5"
  }'
```

**Response:**
```json
{
  "success": true,
  "wash": {
    "wash_id": 123,
    "exchange_id": "ABC123XYZ",
    "washer_wallet": "Gd9CufCg1sdgoyTmrC5WJBJedtUSfRuyof6gNDnkxmD5",
    "destination_wallet": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "amount": "1.5",
    "status": "awaiting",
    "created": "2024-01-15T10:30:00.000000"
  }
}
```

**Response Fields:**
- `washer_wallet`: The deposit address where you must send your SOL. This is automatically generated for each wash.
- `exchange_id`: Unique identifier for this wash operation.

**Status Codes:**
- `201 Created` - Wash created successfully
- `400 Bad Request` - Invalid request data
- `401 Unauthorized` - Invalid or missing API key
- `500 Internal Server Error` - Server error

**Error Response Examples:**

Missing destination_wallet:
```json
{
  "success": false,
  "error": "Missing parameter",
  "message": "destination_wallet is required"
}
```

Invalid amount:
```json
{
  "success": false,
  "error": "Invalid amount",
  "message": "amount must be a positive number"
}
```

---

### Get Wash Status

Get the current status and details of a wash operation.

**Endpoint:** `GET /api/v1/wash/<wash_id>/status`

**Authentication:** Required

**URL Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| wash_id | integer | The ID of the wash operation |

**Example Request:**
```bash
curl -X GET "https://api.zsend.app/api/v1/wash/123/status" \
  -H "X-API-Key: your-api-key-here"
```

**Response:**
```json
{
  "success": true,
  "wash": {
    "wash_id": 123,
    "status": "washing",
    "amount": "1.5",
    "amount_received": "1.5",
    "amount_out": "0",
    "deposit_hash": "5j7s8K9mN2pQ4rT6vW8xY0zA1bC3dE5fG7hI9jK1lM3nO5pQ7rS9tU1vW3xY5z",
    "destination_wallet": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "washer_wallet": "9yLYuh3DX98e98UYJTqcE6kCjfUrB94UaSvKvpThtBvV",
    "chain": "sol",
    "wash_type": "one_to_one",
    "completed": false,
    "created": "2024-01-15T10:30:00.000000",
    "last_checked": "2024-01-15T10:35:00.000000",
    "exchanges": [
      {
        "exchange_type": "deposit",
        "exchange_status": "finished",
        "currency_in": "sol",
        "currency_out": "sol",
        "amount_in": "1.5",
        "amount_out": "0.045",
        "hash_in": "5j7s8K9mN2pQ4rT6vW8xY0zA1bC3dE5fG7hI9jK1lM3nO5pQ7rS9tU1vW3xY5z",
        "hash_out": "0x1234567890abcdef1234567890abcdef12345678",
        "created": "2024-01-15T10:30:00.000000",
        "last_checked": "2024-01-15T10:32:00.000000"
      }
    ]
  }
}
```

**Status Values:**
- `awaiting` - Waiting for deposit to washer wallet
- `confirming` - Deposit detected, waiting for confirmation
- `confirmed` - Deposit confirmed, preparing wash operation
- `washing` - Wash operation in progress
- `withdrawing` - Finalizing wash operation
- `completed` - Wash completed successfully
- `failed` - Wash failed

**Status Codes:**
- `200 OK` - Status retrieved successfully
- `401 Unauthorized` - Invalid or missing API key
- `404 Not Found` - Wash not found
- `500 Internal Server Error` - Server error

---

### List Washes

List washes associated with your API key. Supports pagination and filtering.

**Endpoint:** `GET /api/v1/wash/list`

**Authentication:** Required

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| status | string | No | Filter by status (awaiting, confirming, confirmed, washing, withdrawing, completed, failed) |
| limit | integer | No | Number of results per page (default: 50, max: 100) |
| offset | integer | No | Pagination offset (default: 0) |

**Example Request:**
```bash
curl -X GET "https://api.zsend.app/api/v1/wash/list?status=completed&limit=10&offset=0" \
  -H "X-API-Key: your-api-key-here"
```

**Response:**
```json
{
  "success": true,
  "washes": [
    {
      "wash_id": 123,
      "status": "completed",
      "amount": "1.5",
      "amount_received": "1.5",
      "destination_wallet": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
      "completed": true,
      "created": "2024-01-15T10:30:00.000000"
    },
    {
      "wash_id": 122,
      "status": "washing",
      "amount": "2.0",
      "amount_received": "2.0",
      "destination_wallet": "8yMZvi4EY09f09VZKUrdF7lDkgWVsC05VbTwLwqUiuCwX",
      "completed": false,
      "created": "2024-01-15T09:15:00.000000"
    }
  ],
  "pagination": {
    "total": 25,
    "limit": 10,
    "offset": 0,
    "has_more": true
  }
}
```

**Status Codes:**
- `200 OK` - List retrieved successfully
- `401 Unauthorized` - Invalid or missing API key
- `500 Internal Server Error` - Server error

---

## Error Handling

All error responses follow this format:

```json
{
  "success": false,
  "error": "Error Type",
  "message": "Human-readable error message"
}
```

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | OK - Request successful |
| 201 | Created - Resource created successfully |
| 400 | Bad Request - Invalid request parameters |
| 401 | Unauthorized - Invalid or missing API key |
| 404 | Not Found - Resource not found |
| 500 | Internal Server Error - Server error |

### Common Errors

**Missing API Key:**
```json
{
  "success": false,
  "error": "API key required",
  "message": "Please provide X-API-Key header"
}
```

**Invalid API Key:**
```json
{
  "success": false,
  "error": "Invalid API key",
  "message": "The provided API key is invalid or inactive"
}
```

**Database Error:**
```json
{
  "success": false,
  "error": "Database error",
  "message": "Unable to connect to database"
}
```

---

## Code Examples

### Python

```python
import requests

API_BASE_URL = "https://api.zsend.app/api/v1"
API_KEY = "your-api-key-here"

headers = {
    "X-API-Key": API_KEY,
    "Content-Type": "application/json"
}

# Get a quote
response = requests.get(
    f"{API_BASE_URL}/wash/quote",
    headers=headers,
    params={"amount": 1.5}
)
quote = response.json()
print(f"Amount received: {quote['quote']['amount_received']}")

# Create a wash
wash_data = {
    "destination_wallet": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "amount": "1.5"
}
response = requests.post(
    f"{API_BASE_URL}/wash/create",
    headers=headers,
    json=wash_data
)
wash = response.json()
wash_id = wash['wash']['wash_id']
print(f"Created wash: {wash_id}")

# Check status
response = requests.get(
    f"{API_BASE_URL}/wash/{wash_id}/status",
    headers=headers
)
status = response.json()
print(f"Status: {status['wash']['status']}")
```

### JavaScript/Node.js

```javascript
const axios = require('axios');

const API_BASE_URL = 'https://api.zsend.app/api/v1';
const API_KEY = 'your-api-key-here';

const headers = {
  'X-API-Key': API_KEY,
  'Content-Type': 'application/json'
};

// Get a quote
async function getQuote(amount) {
  const response = await axios.get(`${API_BASE_URL}/wash/quote`, {
    headers,
    params: { amount }
  });
  return response.data.quote;
}

// Create a wash
async function createWash(destinationWallet, amount) {
  const response = await axios.post(
    `${API_BASE_URL}/wash/create`,
    {
      destination_wallet: destinationWallet,
      amount: amount.toString()
    },
    { headers }
  );
  return response.data.wash;
}

// Check status
async function getWashStatus(washId) {
  const response = await axios.get(
    `${API_BASE_URL}/wash/${washId}/status`,
    { headers }
  );
  return response.data.wash;
}

// Example usage
(async () => {
  const quote = await getQuote(1.5);
  console.log(`Amount received: ${quote.amount_received}`);
  
  const wash = await createWash(
    '7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU',
    1.5
  );
  console.log(`Created wash: ${wash.wash_id}`);
  
  const status = await getWashStatus(wash.wash_id);
  console.log(`Status: ${status.status}`);
})();
```

### cURL

```bash
# Get quote
curl -X GET "https://api.zsend.app/api/v1/wash/quote?amount=1.5" \
  -H "X-API-Key: your-api-key-here"

# Create wash
curl -X POST "https://api.zsend.app/api/v1/wash/create" \
  -H "X-API-Key: your-api-key-here" \
  -H "Content-Type: application/json" \
  -d '{
    "destination_wallet": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "amount": "1.5"
  }'

# Check status
curl -X GET "https://api.zsend.app/api/v1/wash/123/status" \
  -H "X-API-Key: your-api-key-here"

# List washes
curl -X GET "https://api.zsend.app/api/v1/wash/list?status=completed&limit=10" \
  -H "X-API-Key: your-api-key-here"
```

### PHP

```php
<?php

$apiBaseUrl = "https://api.zsend.app/api/v1";
$apiKey = "your-api-key-here";

$headers = [
    "X-API-Key: $apiKey",
    "Content-Type: application/json"
];

// Get a quote
$ch = curl_init("$apiBaseUrl/wash/quote?amount=1.5");
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
$quote = json_decode($response, true);
echo "Amount received: " . $quote['quote']['amount_received'] . "\n";

// Create a wash
$washData = json_encode([
    "destination_wallet" => "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "amount" => "1.5"
]);

$ch = curl_init("$apiBaseUrl/wash/create");
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $washData);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
$wash = json_decode($response, true);
$washId = $wash['wash']['wash_id'];
echo "Created wash: $washId\n";

// Check status
$ch = curl_init("$apiBaseUrl/wash/$washId/status");
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$response = curl_exec($ch);
$status = json_decode($response, true);
echo "Status: " . $status['wash']['status'] . "\n";

curl_close($ch);
?>
```

---

## Rate Limiting

Rate limiting is configured per API key. The default rate limit is 100 requests per minute, but this can be customized when creating API keys.

If you exceed the rate limit, you will receive a `429 Too Many Requests` response (note: rate limiting implementation may need to be added to the API server).

---

## Wash Status Flow

Understanding the wash status flow:

1. **awaiting** - Wash created, waiting for deposit to washer wallet
2. **confirming** - Deposit detected, waiting for blockchain confirmation
3. **confirmed** - Deposit confirmed, preparing wash operation
4. **washing** - Wash operation in progress
5. **withdrawing** - Finalizing wash operation
6. **completed** - Wash completed successfully, funds sent to destination wallet
7. **failed** - Wash failed at some stage

---

## Support

For issues, questions, or feature requests, please contact your API administrator.

---

## Changelog

### v1.0.0 (2024-01-15)
- Initial API release
- Basic wash operations
- API key authentication
- Quote endpoint
- Status tracking

