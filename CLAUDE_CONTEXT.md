# CLAUDE_CONTEXT.md

## Objetivo do Projeto

**livraone-invoice** é um módulo de faturamento standalone construído em FlutterFlow + Custom Code (Dart). Faz parte do ecossistema LivraOne e será integrado futuramente via SSO centralizado (LivraOne Gateway/BFF + Keycloak).

Objetivo imediato: entregar MVP funcional de emissão de invoices para organizações, com arquitetura pronta para integração futura sem retrabalho.

---

## Fase Atual

**Phase 1: Mock Auth + Mock API**

- Autenticação simulada via `AuthAdapter` com tokens fake
- Chamadas de API simuladas via `ApiClient` com dados mock
- Nenhuma integração real com backend ainda
- UI funcional para validação de fluxos

---

## Regras de Arquitetura

### FlutterFlow Custom Actions/Functions

Todo código Dart customizado deve seguir o padrão FlutterFlow:
- Custom Actions para lógica de negócio
- Custom Functions para transformações de dados
- Custom Widgets apenas quando componentes nativos não atendem

### ApiClient Obrigatório

Todas as chamadas HTTP devem passar pelo `ApiClient`:

```dart
// lib/custom_code/api_client.dart
class ApiClient {
  static Future<ApiResponse> get(String endpoint, {Map<String, String>? headers});
  static Future<ApiResponse> post(String endpoint, {Map<String, dynamic>? body, Map<String, String>? headers});
  static Future<ApiResponse> put(String endpoint, {Map<String, dynamic>? body, Map<String, String>? headers});
  static Future<ApiResponse> delete(String endpoint, {Map<String, String>? headers});
}
```

**Nunca** fazer chamadas HTTP diretas (http.get, dio, etc.) fora do ApiClient.

### AuthAdapter Obrigatório

Toda lógica de autenticação deve passar pelo `AuthAdapter`:

```dart
// lib/custom_code/auth_adapter.dart
abstract class AuthAdapter {
  Future<AuthResult> login(String email, String password);
  Future<void> logout();
  Future<String?> getAccessToken();
  Future<UserSession?> getCurrentSession();
  bool isAuthenticated();
}

// Phase 1: MockAuthAdapter implements AuthAdapter
// Phase 2+: KeycloakAuthAdapter implements AuthAdapter
```

### Headers Padrão

Toda requisição autenticada deve incluir:

| Header | Descrição | Exemplo |
|--------|-----------|---------|
| `Authorization` | Bearer token JWT | `Bearer eyJhbG...` |
| `X-App-Id` | Identificador da aplicação | `livraone-invoice` |
| `X-Org-Id` | ID da organização do usuário | `org_abc123` |
| `Content-Type` | Tipo de conteúdo | `application/json` |

```dart
Map<String, String> getAuthHeaders(String token, String orgId) {
  return {
    'Authorization': 'Bearer $token',
    'X-App-Id': 'livraone-invoice',
    'X-Org-Id': orgId,
    'Content-Type': 'application/json',
  };
}
```

---

## Proibições

### Não assumir Firebase/Supabase
- Não usar `FirebaseAuth`, `FirebaseFirestore`, `SupabaseClient` diretamente
- Não criar dependências hardcoded com esses providers
- Toda integração de auth/data deve passar pelos adapters

### Não mexer em UI sem pedido explícito
- Alterações visuais apenas quando solicitadas
- Foco em lógica de negócio e integração
- UI é responsabilidade do design system FlutterFlow

### Não introduzir state management complexo
- Não usar Bloc, Riverpod, GetX ou similares
- Usar o state management nativo do FlutterFlow (AppState, PageState)
- Manter simplicidade para compatibilidade FlutterFlow

### Outras proibições
- Não criar arquivos fora da estrutura FlutterFlow padrão
- Não modificar `pubspec.yaml` sem necessidade documentada
- Não implementar cache local sem especificação
- Não armazenar tokens em SharedPreferences/localStorage (usar secure storage quando necessário)

---

## Estrutura de Pastas (Custom Code)

```
lib/
├── custom_code/
│   ├── actions/           # Custom Actions
│   │   ├── auth_actions.dart
│   │   ├── invoice_actions.dart
│   │   └── customer_actions.dart
│   ├── functions/         # Custom Functions
│   │   ├── formatters.dart
│   │   └── validators.dart
│   ├── models/            # Data Models
│   │   ├── user_session.dart
│   │   ├── invoice.dart
│   │   └── customer.dart
│   ├── adapters/          # Adapters (Auth, Storage)
│   │   ├── auth_adapter.dart
│   │   └── mock_auth_adapter.dart
│   └── api/               # API Layer
│       ├── api_client.dart
│       └── mock_api_client.dart
```

---

## Integração Futura (SSO LivraOne)

Arquitetura preparada para:
- **Gateway/BFF**: Proxy central que roteia para microserviços
- **Keycloak**: Identity Provider para SSO
- **Token Exchange**: Troca de tokens entre apps do ecossistema

Quando SSO for implementado:
1. Substituir `MockAuthAdapter` por `KeycloakAuthAdapter`
2. Substituir `MockApiClient` por `LiveApiClient`
3. Configurar redirect URIs no Keycloak
4. Nenhuma mudança em lógica de negócio ou UI necessária

---

## Checklist para PRs

- [ ] Usa `ApiClient` para todas as chamadas HTTP?
- [ ] Usa `AuthAdapter` para toda lógica de auth?
- [ ] Inclui headers padrão (X-App-Id, X-Org-Id, Authorization)?
- [ ] Não introduz dependência direta de Firebase/Supabase?
- [ ] Não altera UI sem solicitação explícita?
- [ ] Não adiciona state management externo?
- [ ] Código compatível com FlutterFlow export?
