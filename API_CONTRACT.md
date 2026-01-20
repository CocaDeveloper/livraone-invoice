# API_CONTRACT.md

## 1. Overview

### 1.1 Phases

| Phase | Description | Auth | Data |
|-------|-------------|------|------|
| **Phase 1** | Mock Foundation | Mock tokens (local) | Mock data (local) |
| **Phase 2** | Real Integration | SSO via LivraOne Gateway | Live API |
| **Phase 3** | Production | SSO + MFA | Production API |

### 1.2 Principles

| Principle | Description |
|-----------|-------------|
| **SSO Central** | Authentication via LivraOne Gateway/BFF with opaque or JWT tokens. No direct Firebase/Supabase dependency. |
| **Multi-tenant** | All data is scoped by `orgId`. Organization ID is derived from the access token; `X-Org-Id` header is optional and validated against token claims. |
| **Performance** | List endpoints return lightweight payloads (no nested collections). Detail endpoints return complete objects. |
| **i18n** | Support for `en-US`, `pt-BR`, `es-ES`. All formatting (currency, dates, numbers) is client-side. API returns raw values only. |
| **Idempotency** | Mutating operations support `X-Request-Id` for idempotency and traceability. |

---

## 2. Base URL

| Environment | URL |
|-------------|-----|
| Phase 1 (Mock) | N/A (local mock data) |
| Development | `https://api.dev.livraone.com` |
| Staging | `https://api.staging.livraone.com` |
| Production | `https://api.livraone.com` |

API Version: All endpoints are prefixed with `/v1`.

---

## 3. Standard Headers

### 3.1 Required Headers

| Header | Type | Description | Example |
|--------|------|-------------|---------|
| `Authorization` | string | Bearer token from SSO | `Bearer eyJhbGciOiJSUzI1NiIs...` |
| `X-App-Id` | string | Application identifier | `livraone-invoice` |
| `Content-Type` | string | Request content type | `application/json` |

### 3.2 Optional Headers

| Header | Type | Description | Example |
|--------|------|-------------|---------|
| `X-Org-Id` | string | Organization override (validated against token) | `org_abc123` |
| `X-Request-Id` | string | Client-generated request ID for traceability | `req_550e8400-e29b-41d4-a716-446655440000` |
| `Accept-Language` | string | Preferred locale for error messages | `en-US`, `pt-BR`, `es-ES` |

---

## 4. Standard Error Response

All error responses follow this structure:

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields failed validation",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Email format is invalid"
      },
      {
        "field": "items",
        "code": "MIN_LENGTH",
        "message": "At least one item is required"
      }
    ]
  }
}
```

### Error Codes

| Code | Description |
|------|-------------|
| `UNAUTHORIZED` | Token missing, invalid, or expired |
| `FORBIDDEN` | Valid token but insufficient permissions |
| `NOT_FOUND` | Resource does not exist |
| `VALIDATION_ERROR` | Request payload validation failed |
| `CONFLICT` | Resource already exists (duplicate) |
| `RATE_LIMITED` | Too many requests |
| `INTERNAL_ERROR` | Server-side error |

---

## 5. Auth & Identity

### GET /v1/me

Returns the authenticated user's profile, active organization, available organizations, roles, permissions, entitlements, and locale preference.

#### Request

```http
GET /v1/me
Authorization: Bearer <token>
X-App-Id: livraone-invoice
Accept-Language: en-US
```

#### Response 200

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "id": "user_abc123",
    "email": "john.doe@company.com",
    "name": "John Doe",
    "avatarUrl": "https://cdn.livraone.com/avatars/user_abc123.png",
    "locale": "en-US",
    "activeOrg": {
      "id": "org_xyz789",
      "name": "Acme Corporation",
      "plan": "professional",
      "roles": ["admin"],
      "permissions": [
        "invoice:read",
        "invoice:write",
        "invoice:delete",
        "customer:read",
        "customer:write",
        "customer:delete",
        "report:read"
      ]
    },
    "orgs": [
      {
        "id": "org_xyz789",
        "name": "Acme Corporation",
        "plan": "professional",
        "roles": ["admin"]
      },
      {
        "id": "org_def456",
        "name": "Beta Consulting",
        "plan": "starter",
        "roles": ["member"]
      }
    ],
    "entitlements": {
      "maxInvoicesPerMonth": 500,
      "maxCustomers": 1000,
      "canExportPdf": true,
      "canManageCustomers": true,
      "canViewReports": true,
      "canInviteMembers": true
    },
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2025-06-20T14:22:00Z"
  }
}
```

### Roles

| Role | Description |
|------|-------------|
| `owner` | Organization owner. Full access including billing and deletion. |
| `admin` | Administrator. Can manage users, settings, and all resources. |
| `member` | Standard member. Access based on assigned permissions. |

### Permissions

| Permission | Description |
|------------|-------------|
| `invoice:read` | View invoices |
| `invoice:write` | Create and edit invoices |
| `invoice:delete` | Delete invoices |
| `invoice:send` | Send invoices to customers |
| `customer:read` | View customers |
| `customer:write` | Create and edit customers |
| `customer:delete` | Delete customers |
| `report:read` | View reports and analytics |
| `settings:read` | View organization settings |
| `settings:write` | Modify organization settings |
| `member:read` | View organization members |
| `member:write` | Invite and manage members |

---

## 6. Customers

### 6.1 Customer Object

```json
{
  "id": "cust_abc123",
  "orgId": "org_xyz789",
  "name": "Acme Industries LLC",
  "email": "billing@acmeindustries.com",
  "phone": "+1-555-123-4567",
  "ein": "12-3456789",
  "address": {
    "line1": "123 Main Street",
    "line2": "Suite 400",
    "city": "Austin",
    "state": "TX",
    "zipCode": "78701",
    "country": "US"
  },
  "notes": "Net 30 payment terms preferred",
  "createdAt": "2024-03-10T08:00:00Z",
  "updatedAt": "2025-05-15T12:30:00Z"
}
```

### 6.2 GET /v1/customers

Returns a paginated list of customers for the organization.

#### Request

```http
GET /v1/customers?page=1&limit=20&search=acme
Authorization: Bearer <token>
X-App-Id: livraone-invoice
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number (1-indexed) |
| `limit` | integer | 20 | Items per page (max 100) |
| `search` | string | - | Search by name, email, or EIN |
| `sortBy` | string | `createdAt` | Sort field: `name`, `email`, `createdAt`, `updatedAt` |
| `sortOrder` | string | `desc` | Sort direction: `asc`, `desc` |

#### Response 200

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "data": [
    {
      "id": "cust_abc123",
      "orgId": "org_xyz789",
      "name": "Acme Industries LLC",
      "email": "billing@acmeindustries.com",
      "phone": "+1-555-123-4567",
      "ein": "12-3456789",
      "address": {
        "line1": "123 Main Street",
        "line2": "Suite 400",
        "city": "Austin",
        "state": "TX",
        "zipCode": "78701",
        "country": "US"
      },
      "createdAt": "2024-03-10T08:00:00Z",
      "updatedAt": "2025-05-15T12:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### 6.3 POST /v1/customers

Creates a new customer.

#### Request

```http
POST /v1/customers
Authorization: Bearer <token>
X-App-Id: livraone-invoice
Content-Type: application/json
X-Request-Id: req_550e8400-e29b-41d4-a716-446655440000
```

```json
{
  "name": "New Customer Inc",
  "email": "contact@newcustomer.com",
  "phone": "+1-555-987-6543",
  "ein": "98-7654321",
  "address": {
    "line1": "456 Oak Avenue",
    "line2": null,
    "city": "Denver",
    "state": "CO",
    "zipCode": "80202",
    "country": "US"
  },
  "notes": "Referred by John Smith"
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Company or individual name |
| `email` | string | Valid email address |

#### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `phone` | string | Phone number (E.164 format recommended) |
| `ein` | string | Employer Identification Number (XX-XXXXXXX) |
| `address` | object | US address object |
| `address.line1` | string | Street address |
| `address.line2` | string | Apartment, suite, unit, etc. |
| `address.city` | string | City name |
| `address.state` | string | Two-letter state code (e.g., TX, CA) |
| `address.zipCode` | string | ZIP code (5 or 9 digits) |
| `address.country` | string | ISO 3166-1 alpha-2 country code |
| `notes` | string | Internal notes |

#### Response 201

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "id": "cust_def456",
    "orgId": "org_xyz789",
    "name": "New Customer Inc",
    "email": "contact@newcustomer.com",
    "phone": "+1-555-987-6543",
    "ein": "98-7654321",
    "address": {
      "line1": "456 Oak Avenue",
      "line2": null,
      "city": "Denver",
      "state": "CO",
      "zipCode": "80202",
      "country": "US"
    },
    "notes": "Referred by John Smith",
    "createdAt": "2025-06-20T14:00:00Z",
    "updatedAt": "2025-06-20T14:00:00Z"
  }
}
```

---

## 7. Invoices

### 7.1 Invoice Status Enum

| Status | Description | Allowed Transitions |
|--------|-------------|---------------------|
| `draft` | Invoice is being prepared | `sent`, `cancelled` |
| `sent` | Invoice has been sent to customer | `paid`, `overdue`, `cancelled` |
| `paid` | Invoice has been paid in full | (terminal) |
| `overdue` | Invoice is past due date | `paid`, `cancelled` |
| `cancelled` | Invoice has been cancelled | (terminal) |

### 7.2 Tax Model (US Sales Tax)

```json
{
  "taxType": "sales_tax",
  "taxRate": 8.25,
  "taxAmount": 82.5,
  "taxJurisdiction": "TX",
  "taxExempt": false,
  "taxExemptReason": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `taxType` | string | Tax type: `sales_tax`, `use_tax`, `none` |
| `taxRate` | number | Tax rate as percentage (e.g., 8.25 for 8.25%) |
| `taxAmount` | number | Calculated tax amount (raw, not formatted) |
| `taxJurisdiction` | string | State or jurisdiction code |
| `taxExempt` | boolean | Whether customer is tax exempt |
| `taxExemptReason` | string | Reason for exemption (if applicable) |

### 7.3 Invoice List Item (Lightweight)

Used in list endpoints. Does NOT include `items[]` array.

```json
{
  "id": "inv_001",
  "orgId": "org_xyz789",
  "number": "INV-2025-0001",
  "customerId": "cust_abc123",
  "customerName": "Acme Industries LLC",
  "status": "sent",
  "currency": "USD",
  "subtotal": 1000,
  "taxAmount": 82.5,
  "total": 1082.5,
  "itemCount": 3,
  "issueDate": "2025-06-01",
  "dueDate": "2025-06-30",
  "paidAt": null,
  "createdAt": "2025-06-01T10:00:00Z",
  "updatedAt": "2025-06-01T10:00:00Z"
}
```

### 7.4 Invoice Detail (Full Object)

Used in detail endpoints. Includes complete `items[]` array.

```json
{
  "id": "inv_001",
  "orgId": "org_xyz789",
  "number": "INV-2025-0001",
  "customerId": "cust_abc123",
  "customerName": "Acme Industries LLC",
  "customerEmail": "billing@acmeindustries.com",
  "customerAddress": {
    "line1": "123 Main Street",
    "line2": "Suite 400",
    "city": "Austin",
    "state": "TX",
    "zipCode": "78701",
    "country": "US"
  },
  "status": "sent",
  "currency": "USD",
  "items": [
    {
      "id": "item_001",
      "description": "Consulting Services - June 2025",
      "quantity": 40,
      "unitPrice": 150,
      "amount": 6000
    },
    {
      "id": "item_002",
      "description": "Software License (Annual)",
      "quantity": 1,
      "unitPrice": 2400,
      "amount": 2400
    },
    {
      "id": "item_003",
      "description": "Support Package - Premium",
      "quantity": 1,
      "unitPrice": 600,
      "amount": 600
    }
  ],
  "subtotal": 9000,
  "tax": {
    "taxType": "sales_tax",
    "taxRate": 8.25,
    "taxAmount": 742.5,
    "taxJurisdiction": "TX",
    "taxExempt": false,
    "taxExemptReason": null
  },
  "total": 9742.5,
  "notes": "Payment due within 30 days. Thank you for your business.",
  "terms": "Net 30",
  "issueDate": "2025-06-01",
  "dueDate": "2025-06-30",
  "sentAt": "2025-06-01T14:30:00Z",
  "paidAt": null,
  "cancelledAt": null,
  "createdBy": "user_abc123",
  "createdAt": "2025-06-01T10:00:00Z",
  "updatedAt": "2025-06-01T14:30:00Z"
}
```

### 7.5 GET /v1/invoices

Returns a paginated list of invoices (lightweight, without items).

#### Request

```http
GET /v1/invoices?page=1&limit=20&status=sent
Authorization: Bearer <token>
X-App-Id: livraone-invoice
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number (1-indexed) |
| `limit` | integer | 20 | Items per page (max 100) |
| `status` | string | - | Filter by status: `draft`, `sent`, `paid`, `overdue`, `cancelled` |
| `customerId` | string | - | Filter by customer ID |
| `fromDate` | string | - | Filter by issue date >= (ISO 8601: YYYY-MM-DD) |
| `toDate` | string | - | Filter by issue date <= (ISO 8601: YYYY-MM-DD) |
| `search` | string | - | Search by invoice number or customer name |
| `sortBy` | string | `createdAt` | Sort field: `number`, `total`, `issueDate`, `dueDate`, `createdAt` |
| `sortOrder` | string | `desc` | Sort direction: `asc`, `desc` |

#### Response 200

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "data": [
    {
      "id": "inv_001",
      "orgId": "org_xyz789",
      "number": "INV-2025-0001",
      "customerId": "cust_abc123",
      "customerName": "Acme Industries LLC",
      "status": "sent",
      "currency": "USD",
      "subtotal": 9000,
      "taxAmount": 742.5,
      "total": 9742.5,
      "itemCount": 3,
      "issueDate": "2025-06-01",
      "dueDate": "2025-06-30",
      "paidAt": null,
      "createdAt": "2025-06-01T10:00:00Z",
      "updatedAt": "2025-06-01T14:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 127,
    "totalPages": 7,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### 7.6 POST /v1/invoices

Creates a new invoice.

#### Request

```http
POST /v1/invoices
Authorization: Bearer <token>
X-App-Id: livraone-invoice
Content-Type: application/json
X-Request-Id: req_550e8400-e29b-41d4-a716-446655440000
```

```json
{
  "customerId": "cust_abc123",
  "issueDate": "2025-06-01",
  "dueDate": "2025-06-30",
  "items": [
    {
      "description": "Consulting Services - June 2025",
      "quantity": 40,
      "unitPrice": 150
    },
    {
      "description": "Software License (Annual)",
      "quantity": 1,
      "unitPrice": 2400
    }
  ],
  "tax": {
    "taxType": "sales_tax",
    "taxRate": 8.25,
    "taxJurisdiction": "TX",
    "taxExempt": false
  },
  "notes": "Payment due within 30 days.",
  "terms": "Net 30"
}
```

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `customerId` | string | Customer ID |
| `issueDate` | string | Issue date (ISO 8601: YYYY-MM-DD) |
| `dueDate` | string | Due date (ISO 8601: YYYY-MM-DD) |
| `items` | array | Line items (minimum 1) |
| `items[].description` | string | Item description |
| `items[].quantity` | number | Quantity (raw number) |
| `items[].unitPrice` | number | Unit price (raw number) |

#### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `tax` | object | Tax configuration |
| `tax.taxType` | string | Tax type: `sales_tax`, `use_tax`, `none` |
| `tax.taxRate` | number | Tax rate as percentage |
| `tax.taxJurisdiction` | string | State or jurisdiction code |
| `tax.taxExempt` | boolean | Whether tax exempt |
| `tax.taxExemptReason` | string | Exemption reason |
| `notes` | string | Notes visible to customer |
| `terms` | string | Payment terms |

#### Response 201

Returns the full Invoice Detail object with generated `id`, `number`, calculated `subtotal`, `taxAmount`, and `total`.

### 7.7 GET /v1/invoices/{id}

Returns the complete invoice detail including all items.

#### Request

```http
GET /v1/invoices/inv_001
Authorization: Bearer <token>
X-App-Id: livraone-invoice
```

#### Response 200

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "data": {
    "id": "inv_001",
    "orgId": "org_xyz789",
    "number": "INV-2025-0001",
    "customerId": "cust_abc123",
    "customerName": "Acme Industries LLC",
    "customerEmail": "billing@acmeindustries.com",
    "customerAddress": {
      "line1": "123 Main Street",
      "line2": "Suite 400",
      "city": "Austin",
      "state": "TX",
      "zipCode": "78701",
      "country": "US"
    },
    "status": "sent",
    "currency": "USD",
    "items": [
      {
        "id": "item_001",
        "description": "Consulting Services - June 2025",
        "quantity": 40,
        "unitPrice": 150,
        "amount": 6000
      },
      {
        "id": "item_002",
        "description": "Software License (Annual)",
        "quantity": 1,
        "unitPrice": 2400,
        "amount": 2400
      }
    ],
    "subtotal": 8400,
    "tax": {
      "taxType": "sales_tax",
      "taxRate": 8.25,
      "taxAmount": 693,
      "taxJurisdiction": "TX",
      "taxExempt": false,
      "taxExemptReason": null
    },
    "total": 9093,
    "notes": "Payment due within 30 days.",
    "terms": "Net 30",
    "issueDate": "2025-06-01",
    "dueDate": "2025-06-30",
    "sentAt": "2025-06-01T14:30:00Z",
    "paidAt": null,
    "cancelledAt": null,
    "createdBy": "user_abc123",
    "createdAt": "2025-06-01T10:00:00Z",
    "updatedAt": "2025-06-01T14:30:00Z"
  }
}
```

#### Response 404

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "error": {
    "code": "NOT_FOUND",
    "message": "Invoice not found"
  }
}
```

---

## 8. Localization (i18n)

### 8.1 Accept-Language Header

The API accepts the `Accept-Language` header to localize error messages and system-generated content.

| Locale | Description |
|--------|-------------|
| `en-US` | English (United States) - Default |
| `pt-BR` | Portuguese (Brazil) |
| `es-ES` | Spanish (Spain) |

#### Example

```http
GET /v1/invoices
Accept-Language: pt-BR
```

### 8.2 Localized Error Messages

```json
{
  "requestId": "req_550e8400-e29b-41d4-a716-446655440000",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Um ou mais campos falharam na validacao",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Formato de email invalido"
      }
    ]
  }
}
```

### 8.3 Client-Side Formatting

All numeric values and dates are returned as raw data. Formatting is the responsibility of the client application.

| Data Type | API Format | Client Formatting Example (en-US) | Client Formatting Example (pt-BR) |
|-----------|------------|-----------------------------------|-----------------------------------|
| Currency | `9742.5` | `$9,742.50` | `US$ 9.742,50` |
| Percentage | `8.25` | `8.25%` | `8,25%` |
| Date | `2025-06-01` | `June 1, 2025` | `1 de junho de 2025` |
| DateTime | `2025-06-01T14:30:00Z` | `Jun 1, 2025, 2:30 PM` | `1 jun 2025 14:30` |
| Quantity | `40` | `40` | `40` |

---

## 9. HTTP Status Codes

| Code | Name | Description |
|------|------|-------------|
| `200` | OK | Request succeeded (GET, PUT, PATCH) |
| `201` | Created | Resource created successfully (POST) |
| `204` | No Content | Request succeeded with no response body (DELETE) |
| `400` | Bad Request | Invalid request syntax or parameters |
| `401` | Unauthorized | Missing, invalid, or expired authentication token |
| `403` | Forbidden | Valid authentication but insufficient permissions |
| `404` | Not Found | Resource does not exist or is not accessible |
| `409` | Conflict | Resource already exists (duplicate) |
| `422` | Unprocessable Entity | Request is well-formed but contains semantic errors |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Unexpected server-side error |
| `503` | Service Unavailable | Service temporarily unavailable (maintenance) |

---

## 10. Phase 1 Mock Data

### 10.1 Mock /v1/me

```json
{
  "requestId": "mock_req_001",
  "data": {
    "id": "user_mock_001",
    "email": "demo@livraone.local",
    "name": "Demo User",
    "avatarUrl": null,
    "locale": "en-US",
    "activeOrg": {
      "id": "org_mock_001",
      "name": "Demo Organization",
      "plan": "professional",
      "roles": ["owner"],
      "permissions": [
        "invoice:read",
        "invoice:write",
        "invoice:delete",
        "invoice:send",
        "customer:read",
        "customer:write",
        "customer:delete",
        "report:read",
        "settings:read",
        "settings:write",
        "member:read",
        "member:write"
      ]
    },
    "orgs": [
      {
        "id": "org_mock_001",
        "name": "Demo Organization",
        "plan": "professional",
        "roles": ["owner"]
      }
    ],
    "entitlements": {
      "maxInvoicesPerMonth": 999,
      "maxCustomers": 9999,
      "canExportPdf": true,
      "canManageCustomers": true,
      "canViewReports": true,
      "canInviteMembers": true
    },
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2025-01-01T00:00:00Z"
  }
}
```

### 10.2 Mock Invoice List

```json
{
  "requestId": "mock_req_002",
  "data": [
    {
      "id": "inv_mock_001",
      "orgId": "org_mock_001",
      "number": "INV-2025-0001",
      "customerId": "cust_mock_001",
      "customerName": "TechCorp Solutions",
      "status": "paid",
      "currency": "USD",
      "subtotal": 5000,
      "taxAmount": 412.5,
      "total": 5412.5,
      "itemCount": 2,
      "issueDate": "2025-05-01",
      "dueDate": "2025-05-31",
      "paidAt": "2025-05-28T16:45:00Z",
      "createdAt": "2025-05-01T09:00:00Z",
      "updatedAt": "2025-05-28T16:45:00Z"
    },
    {
      "id": "inv_mock_002",
      "orgId": "org_mock_001",
      "number": "INV-2025-0002",
      "customerId": "cust_mock_002",
      "customerName": "Global Enterprises Inc",
      "status": "sent",
      "currency": "USD",
      "subtotal": 12500,
      "taxAmount": 1031.25,
      "total": 13531.25,
      "itemCount": 4,
      "issueDate": "2025-06-01",
      "dueDate": "2025-06-30",
      "paidAt": null,
      "createdAt": "2025-06-01T10:00:00Z",
      "updatedAt": "2025-06-01T14:30:00Z"
    },
    {
      "id": "inv_mock_003",
      "orgId": "org_mock_001",
      "number": "INV-2025-0003",
      "customerId": "cust_mock_003",
      "customerName": "StartupXYZ LLC",
      "status": "draft",
      "currency": "USD",
      "subtotal": 3200,
      "taxAmount": 264,
      "total": 3464,
      "itemCount": 1,
      "issueDate": "2025-06-15",
      "dueDate": "2025-07-15",
      "paidAt": null,
      "createdAt": "2025-06-15T08:30:00Z",
      "updatedAt": "2025-06-15T08:30:00Z"
    },
    {
      "id": "inv_mock_004",
      "orgId": "org_mock_001",
      "number": "INV-2025-0004",
      "customerId": "cust_mock_001",
      "customerName": "TechCorp Solutions",
      "status": "overdue",
      "currency": "USD",
      "subtotal": 8750,
      "taxAmount": 721.88,
      "total": 9471.88,
      "itemCount": 3,
      "issueDate": "2025-04-01",
      "dueDate": "2025-04-30",
      "paidAt": null,
      "createdAt": "2025-04-01T11:00:00Z",
      "updatedAt": "2025-05-01T00:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 4,
    "totalPages": 1,
    "hasNext": false,
    "hasPrev": false
  }
}
```

### 10.3 Mock Invoice Detail

```json
{
  "requestId": "mock_req_003",
  "data": {
    "id": "inv_mock_002",
    "orgId": "org_mock_001",
    "number": "INV-2025-0002",
    "customerId": "cust_mock_002",
    "customerName": "Global Enterprises Inc",
    "customerEmail": "accounts@globalenterprises.com",
    "customerAddress": {
      "line1": "500 Corporate Boulevard",
      "line2": "Floor 12",
      "city": "San Francisco",
      "state": "CA",
      "zipCode": "94105",
      "country": "US"
    },
    "status": "sent",
    "currency": "USD",
    "items": [
      {
        "id": "item_mock_001",
        "description": "Enterprise Software License",
        "quantity": 5,
        "unitPrice": 1200,
        "amount": 6000
      },
      {
        "id": "item_mock_002",
        "description": "Implementation Services",
        "quantity": 20,
        "unitPrice": 200,
        "amount": 4000
      },
      {
        "id": "item_mock_003",
        "description": "Training Sessions (4 hours)",
        "quantity": 4,
        "unitPrice": 500,
        "amount": 2000
      },
      {
        "id": "item_mock_004",
        "description": "Support Package - Premium (Annual)",
        "quantity": 1,
        "unitPrice": 500,
        "amount": 500
      }
    ],
    "subtotal": 12500,
    "tax": {
      "taxType": "sales_tax",
      "taxRate": 8.25,
      "taxAmount": 1031.25,
      "taxJurisdiction": "CA",
      "taxExempt": false,
      "taxExemptReason": null
    },
    "total": 13531.25,
    "notes": "Thank you for choosing our services. Payment is due within 30 days.",
    "terms": "Net 30",
    "issueDate": "2025-06-01",
    "dueDate": "2025-06-30",
    "sentAt": "2025-06-01T14:30:00Z",
    "paidAt": null,
    "cancelledAt": null,
    "createdBy": "user_mock_001",
    "createdAt": "2025-06-01T10:00:00Z",
    "updatedAt": "2025-06-01T14:30:00Z"
  }
}
```

### 10.4 Mock Customers

```json
{
  "requestId": "mock_req_004",
  "data": [
    {
      "id": "cust_mock_001",
      "orgId": "org_mock_001",
      "name": "TechCorp Solutions",
      "email": "billing@techcorp.com",
      "phone": "+1-555-100-2000",
      "ein": "45-1234567",
      "address": {
        "line1": "100 Innovation Drive",
        "line2": "Building A",
        "city": "Austin",
        "state": "TX",
        "zipCode": "78701",
        "country": "US"
      },
      "notes": "Preferred customer. Net 15 terms.",
      "createdAt": "2024-06-15T08:00:00Z",
      "updatedAt": "2025-03-10T12:30:00Z"
    },
    {
      "id": "cust_mock_002",
      "orgId": "org_mock_001",
      "name": "Global Enterprises Inc",
      "email": "accounts@globalenterprises.com",
      "phone": "+1-555-200-3000",
      "ein": "78-9012345",
      "address": {
        "line1": "500 Corporate Boulevard",
        "line2": "Floor 12",
        "city": "San Francisco",
        "state": "CA",
        "zipCode": "94105",
        "country": "US"
      },
      "notes": null,
      "createdAt": "2024-09-20T10:00:00Z",
      "updatedAt": "2024-09-20T10:00:00Z"
    },
    {
      "id": "cust_mock_003",
      "orgId": "org_mock_001",
      "name": "StartupXYZ LLC",
      "email": "hello@startupxyz.io",
      "phone": "+1-555-300-4000",
      "ein": "23-4567890",
      "address": {
        "line1": "789 Startup Lane",
        "line2": null,
        "city": "Denver",
        "state": "CO",
        "zipCode": "80202",
        "country": "US"
      },
      "notes": "Early-stage startup. Monthly billing preferred.",
      "createdAt": "2025-01-10T14:00:00Z",
      "updatedAt": "2025-01-10T14:00:00Z"
    },
    {
      "id": "cust_mock_004",
      "orgId": "org_mock_001",
      "name": "MidWest Manufacturing Co",
      "email": "purchasing@midwestmfg.com",
      "phone": "+1-555-400-5000",
      "ein": "56-7890123",
      "address": {
        "line1": "2500 Industrial Parkway",
        "line2": "Unit 15",
        "city": "Chicago",
        "state": "IL",
        "zipCode": "60601",
        "country": "US"
      },
      "notes": "Tax exempt - Manufacturer exemption certificate on file.",
      "createdAt": "2024-11-05T09:00:00Z",
      "updatedAt": "2025-02-14T11:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 4,
    "totalPages": 1,
    "hasNext": false,
    "hasPrev": false
  }
}
```
