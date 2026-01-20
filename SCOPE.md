# SCOPE.md

## MVP Invoice - Escopo Definido

### O que ENTRA no MVP

#### Telas

| Tela | Descrição | Prioridade |
|------|-----------|------------|
| Login | Tela de autenticação (mock em Phase 1) | P0 |
| Dashboard | Visão geral com métricas básicas | P0 |
| Lista de Invoices | Listagem com filtros e busca | P0 |
| Criar Invoice | Formulário de nova invoice | P0 |
| Visualizar Invoice | Detalhes da invoice | P0 |
| Editar Invoice | Edição de invoice (apenas status draft) | P1 |
| Lista de Clientes | Listagem de clientes | P0 |
| Criar Cliente | Formulário de novo cliente | P0 |
| Visualizar Cliente | Detalhes do cliente | P1 |
| Editar Cliente | Edição de cliente | P1 |

#### Features

| Feature | Descrição | Phase |
|---------|-----------|-------|
| Autenticação Mock | Login/logout simulado | 1 |
| CRUD Invoices | Criar, listar, visualizar, editar invoices | 1 |
| CRUD Clientes | Criar, listar, visualizar, editar clientes | 1 |
| Status Workflow | draft → sent → paid/cancelled | 1 |
| Cálculo Automático | Subtotal, impostos, total | 1 |
| Validação de Campos | Campos obrigatórios, formatos | 1 |
| Busca e Filtros | Por status, cliente, data | 1 |
| Paginação | Listagens paginadas | 1 |
| PDF Básico | Exportar invoice como PDF simples | 2 |
| Autenticação Real | Integração com Keycloak via AuthAdapter | 2 |
| API Real | Conexão com backend LivraOne | 2 |

---

### O que FICA FORA do MVP

| Feature | Motivo | Quando |
|---------|--------|--------|
| Processamento de Pagamentos | Requer integração com gateways (Stripe, PagSeguro) | Post-MVP |
| Envio de Email Real | Requer serviço de email configurado | Post-MVP |
| PDF Avançado | Templates customizados, logo, etc. | Post-MVP |
| Multi-currency | Complexidade de câmbio e formatação | Post-MVP |
| Integração Contábil | Requer definição de padrões fiscais | Post-MVP |
| Relatórios Avançados | Gráficos, exportação Excel, etc. | Post-MVP |
| Notificações Push | Requer infraestrutura de notificações | Post-MVP |
| Multi-idioma | i18n completo | Post-MVP |
| Recorrência | Invoices recorrentes automáticas | Post-MVP |
| Aprovações/Workflow | Fluxo de aprovação multi-nível | Post-MVP |
| Anexos | Upload de arquivos nas invoices | Post-MVP |
| Histórico/Auditoria | Log detalhado de alterações | Post-MVP |
| API Pública | Endpoints para integrações externas | Post-MVP |

---

## Roadmap de Fases

### Phase 1: Mock Foundation

**Objetivo:** Validar fluxos de UI e lógica de negócio com dados simulados.

**Entregas:**
- [ ] AuthAdapter + MockAuthAdapter funcionando
- [ ] ApiClient + MockApiClient funcionando
- [ ] Todas as telas P0 implementadas
- [ ] CRUD completo de invoices (mock)
- [ ] CRUD completo de clientes (mock)
- [ ] Validações de formulários
- [ ] Status workflow implementado

**Gate PASS/FAIL:**

| Critério | PASS | FAIL |
|----------|------|------|
| Login mock funciona | ✓ Usuário consegue logar com credenciais mock | ✗ Erro ou tela quebrada |
| Criar invoice | ✓ Invoice criada aparece na lista | ✗ Dados perdidos ou erro |
| Listar invoices | ✓ Lista carrega com paginação | ✗ Lista vazia ou erro |
| Criar cliente | ✓ Cliente criado aparece na lista | ✗ Dados perdidos ou erro |
| Workflow status | ✓ Status muda corretamente | ✗ Status inconsistente |
| Headers corretos | ✓ Todas requisições têm X-App-Id, X-Org-Id | ✗ Headers faltando |
| Sem Firebase/Supabase | ✓ Zero imports diretos | ✗ Qualquer import direto |

**Resultado:** PASS se todos os critérios atendidos. Qualquer FAIL bloqueia Phase 2.

---

### Phase 2: Real Integration

**Objetivo:** Conectar com backend real e Keycloak.

**Pré-requisitos:**
- Phase 1 PASS
- Backend API disponível
- Keycloak configurado com client_id para livraone-invoice

**Entregas:**
- [ ] KeycloakAuthAdapter implementado
- [ ] LiveApiClient implementado
- [ ] Configuração de ambiente (dev/staging/prod)
- [ ] Tratamento de erros reais
- [ ] Refresh token automático
- [ ] PDF básico funcional

**Gate PASS/FAIL:**

| Critério | PASS | FAIL |
|----------|------|------|
| Login real | ✓ Autenticação via Keycloak funciona | ✗ Erro de auth |
| Token refresh | ✓ Token renova automaticamente | ✗ Usuário deslogado inesperadamente |
| API real | ✓ Dados persistem no backend | ✗ Dados perdidos |
| Org isolation | ✓ Usuário só vê dados da sua org | ✗ Vazamento de dados |
| PDF export | ✓ PDF gerado corretamente | ✗ PDF corrompido ou vazio |
| Error handling | ✓ Erros exibidos adequadamente | ✗ Tela branca ou crash |

**Resultado:** PASS se todos os critérios atendidos. Qualquer FAIL bloqueia Phase 3.

---

### Phase 3: Production Hardening

**Objetivo:** Preparar para produção com qualidade e segurança.

**Pré-requisitos:**
- Phase 2 PASS
- Ambiente de produção disponível
- Processo de deploy definido

**Entregas:**
- [ ] Testes E2E principais fluxos
- [ ] Logging e monitoramento
- [ ] Rate limiting configurado
- [ ] Secure storage para tokens
- [ ] Telas P1 finalizadas
- [ ] Performance otimizada
- [ ] Documentação de deploy

**Gate PASS/FAIL:**

| Critério | PASS | FAIL |
|----------|------|------|
| Testes E2E | ✓ Cobertura dos fluxos críticos | ✗ Sem testes ou falhando |
| Performance | ✓ Listagens < 2s | ✗ Lentidão perceptível |
| Segurança | ✓ Tokens em secure storage | ✗ Tokens expostos |
| Erros rastreáveis | ✓ Logs com correlation_id | ✗ Erros não rastreáveis |
| Deploy automatizado | ✓ Pipeline CI/CD funcional | ✗ Deploy manual |

**Resultado:** PASS libera para produção. FAIL requer correções.

---

## Critérios de Aceitação Globais

### Compatibilidade SSO

Toda implementação deve:
1. Usar `AuthAdapter` para auth (nunca direto)
2. Usar `ApiClient` para HTTP (nunca direto)
3. Incluir headers `X-App-Id` e `X-Org-Id` em toda requisição
4. Não assumir provider específico (Firebase, Supabase, etc.)
5. Permitir troca de adapter sem mudança em lógica de negócio

### Qualidade de Código

- Zero warnings do Flutter analyzer
- Nomes descritivos (sem abreviações obscuras)
- Funções com responsabilidade única
- Comentários apenas quando lógica não é óbvia

### FlutterFlow Compliance

- Custom code compatível com export FlutterFlow
- Não usar packages incompatíveis
- Seguir estrutura de pastas padrão
- Documentar qualquer dependência adicionada
