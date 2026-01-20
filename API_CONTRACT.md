# API_CONTRACT.md

## Visão Geral

Contratos JSON para integração do livraone-invoice com backend (mock em Phase 1, real em Phase 2+).

**Base URL:**
- Phase 1 (Mock): N/A (dados locais)
- Phase 2+: `https://api.livraone.com` ou via Gateway/BFF

**Headers obrigatórios em todas as requisições autenticadas:**

```
Authorization: Bearer <access_token>
X-App-Id: livraone-invoice
X-Org-Id: <org_id>
Content-Type: application/json
```

---

## GET /v1/me

Retorna dados do usuário autenticado e sua organização.

### Request

```http
GET /v1/me
Authorization: Bearer <token>
X-App-Id: livraone-invoice
```

### Response 200

```json
{
  "id": "user_abc123",
  "email": "usuario@empresa.com",
  "name": "João Silva",
  "orgId": "org_xyz789",
  "orgName": "Empresa LTDA",
  "roles": ["admin", "invoice:write", "invoice:read"],
  "entitlements": {
    "maxInvoicesPerMonth": 100,
    "canExportPdf": true,
    "canManageCustomers": true
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-06-20T14:22:00Z"
}
```

### Campos Obrigatórios

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | string | ID único do usuário |
| `email` | string | Email do usuário |
| `orgId` | string | ID da organização (multi-tenant) |
| `roles` | string[] | Roles do usuário na organização |
| `entitlements` | object | Limites e permissões do plano |

### Response 401

```json
{
  "error": "unauthorized",
  "message": "Token inválido ou expirado"
}
```

---

## GET /v1/invoices

Lista invoices da organização com paginação.

### Request

```http
GET /v1/invoices?page=1&limit=20&status=draft
Authorization: Bearer <token>
X-App-Id: livraone-invoice
X-Org-Id: <org_id>
```

### Query Parameters

| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `page` | int | 1 | Página atual |
| `limit` | int | 20 | Itens por página (max 100) |
| `status` | string | - | Filtro: draft, sent, paid, cancelled |
| `customerId` | string | - | Filtro por cliente |
| `fromDate` | string | - | Data inicial (ISO 8601) |
| `toDate` | string | - | Data final (ISO 8601) |

### Response 200

```json
{
  "data": [
    {
      "id": "inv_001",
      "orgId": "org_xyz789",
      "number": "INV-2024-0001",
      "customerId": "cust_abc",
      "customerName": "Cliente Exemplo",
      "status": "draft",
      "currency": "BRL",
      "subtotal": 1000.00,
      "taxAmount": 100.00,
      "total": 1100.00,
      "issueDate": "2024-06-01",
      "dueDate": "2024-06-30",
      "items": [
        {
          "id": "item_001",
          "description": "Serviço de consultoria",
          "quantity": 10,
          "unitPrice": 100.00,
          "amount": 1000.00
        }
      ],
      "notes": "Pagamento via PIX",
      "createdAt": "2024-06-01T10:00:00Z",
      "updatedAt": "2024-06-01T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 45,
    "totalPages": 3
  }
}
```

### Campos Obrigatórios por Invoice

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | string | ID único da invoice |
| `orgId` | string | ID da organização |
| `number` | string | Número sequencial |
| `customerId` | string | ID do cliente |
| `status` | string | Estado atual |
| `total` | number | Valor total |
| `issueDate` | string | Data de emissão |
| `dueDate` | string | Data de vencimento |

---

## POST /v1/invoices

Cria nova invoice.

### Request

```http
POST /v1/invoices
Authorization: Bearer <token>
X-App-Id: livraone-invoice
X-Org-Id: <org_id>
Content-Type: application/json
```

```json
{
  "customerId": "cust_abc",
  "issueDate": "2024-06-01",
  "dueDate": "2024-06-30",
  "items": [
    {
      "description": "Serviço de consultoria",
      "quantity": 10,
      "unitPrice": 100.00
    }
  ],
  "notes": "Pagamento via PIX",
  "taxRate": 10.0
}
```

### Campos Obrigatórios no Request

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `customerId` | string | ID do cliente |
| `issueDate` | string | Data de emissão (YYYY-MM-DD) |
| `dueDate` | string | Data de vencimento (YYYY-MM-DD) |
| `items` | array | Lista de itens (min 1) |
| `items[].description` | string | Descrição do item |
| `items[].quantity` | number | Quantidade |
| `items[].unitPrice` | number | Preço unitário |

### Response 201

```json
{
  "id": "inv_002",
  "orgId": "org_xyz789",
  "number": "INV-2024-0002",
  "customerId": "cust_abc",
  "customerName": "Cliente Exemplo",
  "status": "draft",
  "currency": "BRL",
  "subtotal": 1000.00,
  "taxAmount": 100.00,
  "total": 1100.00,
  "issueDate": "2024-06-01",
  "dueDate": "2024-06-30",
  "items": [
    {
      "id": "item_001",
      "description": "Serviço de consultoria",
      "quantity": 10,
      "unitPrice": 100.00,
      "amount": 1000.00
    }
  ],
  "notes": "Pagamento via PIX",
  "createdAt": "2024-06-01T10:00:00Z",
  "updatedAt": "2024-06-01T10:00:00Z"
}
```

### Response 400

```json
{
  "error": "validation_error",
  "message": "Dados inválidos",
  "details": [
    {"field": "customerId", "message": "Cliente não encontrado"},
    {"field": "items", "message": "Mínimo de 1 item obrigatório"}
  ]
}
```

---

## GET /v1/customers

Lista clientes da organização.

### Request

```http
GET /v1/customers?page=1&limit=20&search=empresa
Authorization: Bearer <token>
X-App-Id: livraone-invoice
X-Org-Id: <org_id>
```

### Query Parameters

| Param | Tipo | Default | Descrição |
|-------|------|---------|-----------|
| `page` | int | 1 | Página atual |
| `limit` | int | 20 | Itens por página |
| `search` | string | - | Busca por nome/email/documento |

### Response 200

```json
{
  "data": [
    {
      "id": "cust_abc",
      "orgId": "org_xyz789",
      "name": "Cliente Exemplo LTDA",
      "email": "contato@cliente.com",
      "phone": "+5511999999999",
      "document": "12.345.678/0001-90",
      "documentType": "CNPJ",
      "address": {
        "street": "Rua Exemplo, 123",
        "city": "São Paulo",
        "state": "SP",
        "zipCode": "01234-567",
        "country": "BR"
      },
      "createdAt": "2024-01-10T08:00:00Z",
      "updatedAt": "2024-05-15T12:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 12,
    "totalPages": 1
  }
}
```

### Campos Obrigatórios por Customer

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | string | ID único do cliente |
| `orgId` | string | ID da organização |
| `name` | string | Nome/Razão social |
| `email` | string | Email de contato |

---

## POST /v1/customers

Cria novo cliente.

### Request

```http
POST /v1/customers
Authorization: Bearer <token>
X-App-Id: livraone-invoice
X-Org-Id: <org_id>
Content-Type: application/json
```

```json
{
  "name": "Novo Cliente LTDA",
  "email": "contato@novocliente.com",
  "phone": "+5511888888888",
  "document": "98.765.432/0001-10",
  "documentType": "CNPJ",
  "address": {
    "street": "Av. Nova, 456",
    "city": "Rio de Janeiro",
    "state": "RJ",
    "zipCode": "20000-000",
    "country": "BR"
  }
}
```

### Campos Obrigatórios no Request

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `name` | string | Nome/Razão social |
| `email` | string | Email válido |

### Response 201

```json
{
  "id": "cust_xyz",
  "orgId": "org_xyz789",
  "name": "Novo Cliente LTDA",
  "email": "contato@novocliente.com",
  "phone": "+5511888888888",
  "document": "98.765.432/0001-10",
  "documentType": "CNPJ",
  "address": {
    "street": "Av. Nova, 456",
    "city": "Rio de Janeiro",
    "state": "RJ",
    "zipCode": "20000-000",
    "country": "BR"
  },
  "createdAt": "2024-06-20T14:00:00Z",
  "updatedAt": "2024-06-20T14:00:00Z"
}
```

### Response 409

```json
{
  "error": "conflict",
  "message": "Cliente com este documento já existe"
}
```

---

## Códigos de Status HTTP

| Código | Descrição |
|--------|-----------|
| 200 | Sucesso (GET, PUT) |
| 201 | Criado (POST) |
| 204 | Sem conteúdo (DELETE) |
| 400 | Erro de validação |
| 401 | Não autenticado |
| 403 | Sem permissão |
| 404 | Não encontrado |
| 409 | Conflito (duplicata) |
| 422 | Entidade não processável |
| 500 | Erro interno |

---

## Mock Data (Phase 1)

Para Phase 1, usar dados mock estáticos:

```dart
// lib/custom_code/api/mock_data.dart
final mockUser = {
  'id': 'user_mock_001',
  'email': 'dev@livraone.local',
  'name': 'Desenvolvedor Mock',
  'orgId': 'org_mock_001',
  'orgName': 'Organização Teste',
  'roles': ['admin', 'invoice:write', 'invoice:read', 'customer:write', 'customer:read'],
  'entitlements': {
    'maxInvoicesPerMonth': 999,
    'canExportPdf': true,
    'canManageCustomers': true,
  },
};
```
