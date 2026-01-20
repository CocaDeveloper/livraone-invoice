Crie 3 arquivos Markdown para governança do repo "livraone-invoice" (FlutterFlow + Custom Code), visando reduzir retrabalho e manter compatibilidade com SSO futuro do LivraOne.

Entregue exatamente 3 arquivos completos (conteúdo final), em Markdown:

1) CLAUDE_CONTEXT.md
- Objetivo do projeto
- Regras de arquitetura (FlutterFlow custom actions/functions, ApiClient obrigatório, AuthAdapter obrigatório)
- Fase atual: Phase 1 (mock auth + mock api)
- Proibições: não assumir Firebase/Supabase, não mexer em UI sem pedido, não introduzir state management complexo
- Padrões: headers X-App-Id, X-Org-Id, Authorization

2) API_CONTRACT.md
- Contratos JSON mínimos para:
  GET /v1/me
  GET /v1/invoices
  POST /v1/invoices
  GET /v1/customers
  POST /v1/customers
- Campos obrigatórios: orgId, roles, entitlements

3) SCOPE.md
- O que entra no MVP Invoice (telas e features)
- O que fica fora (pagamentos, email real, PDF avançado, multi-currency, contabilidade)
- Roadmap curto (Phase 1/2/3) com gates PASS/FAIL

Regras:
- Texto direto, sem explicação extra.
- Alinhado com futuro SSO via LivraOne Gateway/BFF + Keycloak, mas sem implementar agora.
