# EcommCloudRunFunctionWithCICD

A Google Cloud Run HTTP function for processing e-commerce order events. This service validates, enriches, and stores order data from an e-commerce platform.

## Overview

This Cloud Run function handles `order.created` events from an e-commerce system. It provides:

- **Request validation** - Ensures all required fields are present and properly formatted
- **Payload enrichment** - Adds processing metadata (unique ID, timestamp)
- **Data persistence** - Simulates saving orders to a database
- **Structured logging** - All operations are logged for debugging and monitoring
- **Error handling** - Returns appropriate HTTP status codes and error messages

## Project Structure

```
EcommCloudRunFunctionWithCICD/
├── main.py                 # Cloud Run HTTP entry point
├── requirements.txt        # Python dependencies
├── README.md              # This file
├── test_curl_event.txt    # Example curl request
└── utils/
    ├── __init__.py
    └── order_utils.py     # Validation, enrichment, and DB utilities
```

## Dependencies

- `functions-framework==3.*` - Google Cloud Functions framework for Python
- `Flask` - Web framework for handling HTTP requests

Install dependencies:
```bash
pip install -r requirements.txt
```

## API Reference

### Endpoint: POST /

**Description:** Process an order event

**Request Headers:**
- `Content-Type: application/json`
- `Authorization: bearer <gcloud-identity-token>` (for GCP authenticated requests)

**Request Body:**
```json
{
  "order_id": "ORD-20250426-1001",
  "customer_id": "CUST-100234",
  "order_date": "2025-04-26T22:30:45Z",
  "source": "web_portal",
  "items": [
    {
      "sku": "WM-12345",
      "name": "Wireless Ergonomic Mouse",
      "qty": 2,
      "unit_price": 25.99
    }
  ],
  "shipping_address": {
    "line1": "123 Tech Park Rd",
    "line2": "Suite 400",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94107",
    "country": "USA"
  },
  "payment_method": "credit_card",
  "total_amount": 131.97
}
```

**Required Fields:**
- `order_id` (string) - Unique order identifier
- `customer_id` (string) - Customer identifier
- `items` (array) - List of items in the order (non-empty)
  - `sku` (string) - Stock keeping unit
  - `name` (string) - Item name
  - `qty` (integer) - Quantity (must be > 0)
  - `unit_price` (number) - Price per unit (must be >= 0)
- `order_date` (string) - ISO 8601 timestamp
- `shipping_address` (object) - Delivery address
  - `line1` (string) - Primary address line
  - `city` (string)
  - `state` (string)
  - `postal_code` (string)
  - `country` (string)
- `payment_method` (string) - Payment type (e.g., "credit_card")
- `total_amount` (number) - Order total (must match sum of item prices)

**Success Response (200):**
```json
{
  "status": "processed",
  "order_id": "ORD-20250426-1001",
  "processing_id": "a1b2c3d4e5f6g7h8",
  "processed_at": "2025-04-26T23:15:30.123456Z",
  "items_count": 2,
  "total_amount": 131.97,
  "payment_method": "credit_card",
  "shipping_address": {
    "line1": "123 Tech Park Rd",
    "line2": "Suite 400",
    "city": "San Francisco",
    "state": "CA",
    "postal_code": "94107",
    "country": "USA"
  },
  "message": "Order received and stored."
}
```

**Error Responses:**

- **400 Bad Request** - Invalid JSON or missing/malformed fields
  ```json
  {"error": "Missing fields: ['items', 'order_id']"}
  ```

- **405 Method Not Allowed** - Non-POST request
  ```json
  {"error": "Use POST"}
  ```

- **500 Internal Server Error** - Database save failure
  ```json
  {"error": "Internal error"}
  ```

## Validation Rules

The service validates:

1. **Required fields** - All specified fields must be present
2. **Item validation** - Must have at least one item with valid `qty` (positive int) and `unit_price` (non-negative number)
3. **Address validation** - All required address fields must be present
4. **Total amount** - Must match the sum of all item quantities × unit prices (with floating-point tolerance)

## Usage

### Local Development

1. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Run the function locally:
   ```bash
   functions-framework --target=order_event --debug --port=8080
   ```

3. Test with curl:
   ```bash
   curl -X POST http://localhost:8080 \
     -H "Content-Type: application/json" \
     -d @test_curl_event.txt
   ```

### Deployment to Google Cloud Run

1. Set your GCP project:
   ```bash
   gcloud config set project YOUR_PROJECT_ID
   ```

2. Deploy the function:
   ```bash
   gcloud functions deploy order_event \
     --runtime python311 \
     --trigger-http \
     --allow-unauthenticated
   ```

3. Get the function URL:
   ```bash
   gcloud functions describe order_event --gen2 --format='value(serviceConfig.uri)'
   ```

## Testing

Use the provided `test_curl_event.txt` file:

```bash
curl -X POST https://your-cloud-run-url \
  -H "Authorization: bearer $(gcloud auth print-identity-token)" \
  -H "Content-Type: application/json" \
  -d @test_curl_event.txt
```

## Logging

The function uses Python's standard `logging` module. Logs include:

- Request received (method, path)
- Validation results
- Payload enrichment details
- Database operation status
- Successful order processing

Logs are output to Cloud Run's standard output and accessible via Cloud Logging.

## CI/CD

This project supports continuous integration and deployment via Cloud Build. Configuration details can be found in the project's build files.

## Notes

- The `simulate_db_save()` function is a placeholder for actual database operations (Cloud SQL, Firestore, etc.)
- Timestamps are in UTC (ISO 8601 format with 'Z' suffix)
- Processing IDs are generated as UUID hex values
- Total amount is validated with floating-point precision tolerance