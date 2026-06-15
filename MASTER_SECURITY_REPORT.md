# 🛡️ VALEN POS — Master Security Report

> Consolidação da maratona SecOps (auditoria → exploração → remediação) do VALEN
> POS Market Edition. Stack: backend Elixir/Phoenix, frontend C#/.NET 8 Avalonia,
> offline-first, multi-tenant. Astraz Studio · 2026-06-15.

---

## 1. Status final das vulnerabilidades

| ID | Vulnerabilidade | Classe (OWASP) | Status | Mitigação |
|----|-----------------|----------------|--------|-----------|
| VULN-1 | Mass assignment do `status` da venda | A04/A08 | ✅ | Servidor impõe `status="completed"` (sobrescreve payload) |
| VULN-2 | Cross-tenant em item de venda (IDOR) | A01 | ✅ | Query de ownership: todo `product_id` do lote tem que ser do tenant |
| VULN-3 | Cross-tenant em `stock_movements:push` | A01 | ✅ | `product_owned_by_tenant?` antes de qualquer escrita |
| VULN-4 | Segredo hardcoded/compartilhado | A02/A05 | ✅ | `config/runtime.exs` lê `SECRET_KEY_BASE`/`DATABASE_URL` de env (prod) |
| VULN-5 | Spoof de `user_id` (operador) | A01 | ✅ | `user_id` imposto pelo contexto autenticado, nunca do payload |
| VULN-6 | Login sem rate-limit (brute force) | A07 | ✅ | Hammer + ETS, 5/min por IP → 429; XFF-aware atrás de proxy |
| VULN-7 | Enumeração de tenant/conta | A07 | ✅ | Resposta única `invalid_credentials` p/ todos os casos |
| VULN-8 | `certificate_password` em texto plano | A02 | ✅ | Cloak AES-256-GCM (`ValenPos.Encrypted.Binary`), cifrado em repouso |
| VULN-9 | Device token TTL 1 ano sem rotação | A07 | ⏳ pendente | Próxima rodada (TTL menor + refresh/revogação) |

**8 de 9 fechadas e cobertas por teste de regressão.** VULN-9 é a única pendência
(decisão de arquitetura — rotação de token de device).

### Itens de manutenção/infra (RELATORIO_MELHORIAS)
| Item | Status |
|------|--------|
| #1 EnsureCreated destrutivo (C#) → EF Migrations | ✅ |
| #5 `sync_queue` crescimento infinito → purge 30d | ✅ |
| #6 DLQ punindo offline → transporte × negócio | ✅ |
| #14 binários no git → `.gitignore` raiz | ✅ |

---

## 2. Como a arquitetura se comporta hoje contra ataques

### Cross-Tenant / IDOR
Todo escopo (`tenant_id`, `branch_id`, `company_id`) vem do **token autenticado da
socket/JWT**, nunca do payload. Ingestão de vendas e de movimentos de estoque roda
uma **query interceptadora de ownership**: qualquer `product_id` que não pertença
ao tenant derruba a operação inteira (rejeição, sem escrita). Leituras (produtos,
vendas) são filtradas por `tenant_id` do token → sem vazamento entre clientes do SaaS.

### Mass Assignment
Campos sensíveis (`status`, `user_id`, `tenant_id`, `branch_id`, `company_id`,
`device_id`) são **sobrescritos no servidor** antes do changeset. O cliente pode
enviar o que quiser no JSON — é descartado. Catálogo idem (`ensure_tenant`).

### Injeção (SQLi)
100% via Ecto parametrizado; `Repo.query!` usa binds posicionais. Sem
interpolação de input em SQL. Superfície de SQLi: nenhuma identificada.

### Forjamento de Tokens
JWT **HS256 self-contained** com `secret_key_base`: `verify` recomputa o HMAC
(não lê `alg` do header → imune a alg-confusion) e compara com
`Plug.Crypto.secure_compare` (tempo constante). Em produção o segredo vem só de
env (VULN-4), distinto por ambiente → token de dev não vale em prod. Device tokens
(Phoenix.Token) checam `is_active` no connect (revogação).

### Brute-Force / Credential Stuffing
Rate-limit por IP (Hammer/ETS) nos dois endpoints de login: **5 tentativas/min →
429**. Atrás de LB/CDN usa `X-Forwarded-For` para barrar o IP real. Combinado com
erro de login unificado (VULN-7), o atacante nem enumera contas nem martela senhas.

### Dados sensíveis em repouso
Senha do certificado A1 (NFC-e) cifrada com **AES-256-GCM** (Cloak). Dump do
Postgres expõe só ciphertext (`bytea`). Chave via `CLOAK_ENCRYPTION_KEY` em prod.

---

## 3. Status dos builds e testes

### Backend (Elixir/Phoenix)
```
mix compile --warnings-as-errors   → limpo (0 avisos)
mix test                           → 58 testes, 0 falhas
```
Cobertura de segurança como regressão:
- `test/valen_pos/security_poc_test.exs` — cross-tenant + mass assignment + spoof (todos rejeitados).
- `test/valen_pos_web/controllers/auth_controller_test.exs` — anti-enumeração + rate-limit 6→429.
- `test/valen_pos/encryption_test.exs` — `certificate_password` cifrado em repouso.

### Frontend (C#/.NET 8)
```
dotnet build ValenPos.Desktop.sln -c Debug   → 0 erros / 0 avisos
```
EF Migrations (`InitialCreate`) + `Database.Migrate()` no boot · SyncEngine com
DLQ resiliente (transporte × negócio) · purge da `sync_queue`.

---

## 4. Pendências para o próximo ciclo
- **VULN-9** — rotação/TTL menor + revogação ativa do device token.
- Unificar rotas admin (`/api/v1/products|sales|devices`) no JWT (hoje `AuthenticateTenant` usa Phoenix.Token admin).
- Rotação de chave Cloak (suporte a múltiplos ciphers/retire de chave antiga).
- Backfill cifrando `certificate_password` se houver dados legados em produção.
- Confiar em `X-Forwarded-For` apenas com proxy confiável configurado (`:trust_proxy_header`).

> **Resultado:** 8/9 vulnerabilidades fechadas, build 100% verde nos dois
> ecossistemas, postura de produção para Cross-Tenant, Mass Assignment, Injeção,
> Forjamento de Token e Brute-Force.
