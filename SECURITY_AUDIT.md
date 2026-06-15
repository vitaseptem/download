# 🛡️ VALEN POS — Auditoria de Segurança (Pentest Autorizado)

> Escopo: código-fonte do projeto VALEN POS (Elixir/Phoenix + C#/.NET 8). Pentest
> autorizado pelo dono no próprio repositório. Data: 2026-06-15.
> Metodologia: análise estática (white-box) + **prova de conceito executável**
> (`test/valen_pos/security_poc_test.exs`, 3 testes — todos confirmam exploit).

| Sev | ID | Vulnerabilidade | Prova |
|-----|----|-----------------|-------|
| 🔴 Alta | VULN-1 | Mass assignment: cliente controla `status` da venda | PoC ✅ |
| 🔴 Alta | VULN-2 | Cross-tenant: item de venda referencia produto de outro tenant | PoC ✅ |
| 🔴 Alta | VULN-3 | Cross-tenant via `stock_movements:push` (sem validar dono do produto) | PoC ✅ |
| 🔴 Crít | VULN-4 | `secret_key_base` hardcoded e reutilizado p/ JWT + Phoenix.Token | estático |
| 🟠 Alta | VULN-5 | `user_id` da venda forjável (spoof de atribuição do operador) | estático |
| 🟠 Alta | VULN-6 | Login sem rate-limit/lockout (brute force) | estático |
| 🟡 Méd | VULN-7 | Enumeração de tenant via mensagens de erro distintas | estático |
| 🟡 Méd | VULN-8 | `certificate_password` (NFC-e) em texto plano | estático |
| 🟡 Méd | VULN-9 | Token de device TTL 1 ano, sem rotação | estático |

---

## 🔴 VULN-1 — Mass assignment do `status` da venda
**OWASP A08 / A04.** `Sale.sync_changeset` faz `cast(attrs, [... :status ...])` e
`persist_sale` (`lib/valen_pos/sales.ex:63-69`) sobrescreve `tenant_id/company_id/
branch_id/cash_register_id/device_id` — **mas não `status` nem `user_id`**.

**Exploit:** device autenticado envia `"status" => "canceled"` no payload. Servidor persiste.
```
PoC: VULN-1 -> persisted.status == "canceled"  (forjado pelo cliente) ✅
```
**Impacto:** caixa marca vendas como `canceled`/estado arbitrário → fraude em
relatório/fechamento, sonegação, burla de conferência.

**Fix:** impor `status` no servidor (`Map.put("status", "completed")`) ou
restringir via `validate_inclusion` + remover do `cast` do cliente.

---

## 🔴 VULN-2 — Cross-tenant: produto de outro tenant em item de venda
**OWASP A01 (Broken Access Control / IDOR).** `SaleItem.changeset`
(`lib/valen_pos/sales/sale_item.ex:44`) só faz `foreign_key_constraint(:product_id)`
— **não valida** que o produto pertence ao tenant do socket.

**Exploit:** device do **tenant A** envia item com `product_id` de produto do **tenant B**.
A venda é aceita e cria/decrementa saldo do produto de B **sob a filial de A**.
```
PoC: VULN-2 -> stock(produtoB, branchA) == -9.000  (poluição cross-tenant) ✅
```
**Impacto:** corrupção de dados entre clientes do SaaS, vazamento de existência de
SKUs de terceiros, inconsistência de inventário.

**Fix:** validar no `process_offline_sales_chunk` que todo `product_id` dos itens
pertence ao `ctx.tenant_id` (query `where p.id in ^ids and p.tenant_id == ^tenant`),
rejeitando a venda senão.

---

## 🔴 VULN-3 — Cross-tenant via `stock_movements:push`
**OWASP A01.** `Inventory.process_offline_movements_chunk`
(`lib/valen_pos/inventory.ex`) usa `raw["product_id"]` sem checar dono.

**Exploit:** device do tenant A faz `ENTRY` de 999 em produto do tenant B.
```
PoC: VULN-3 -> movimento aceito p/ produto de outro tenant ✅
```
**Impacto:** inflar/zerar estoque alheio, ledger de auditoria poluído.

**Fix:** mesma validação de ownership do VULN-2 antes de `apply_movement_multi`.

---

## 🔴 VULN-4 — Segredo hardcoded reutilizado (token forgery)
**OWASP A02 / A05.** `config/dev.exs` e `config/test.exs` têm `secret_key_base`
fixo no repo. **Não existe `config/prod.exs`.** Esse mesmo segredo assina:
device token (Phoenix.Token), admin token e **JWT** (`ValenPos.JWT` usa
`Endpoint.config(:secret_key_base)`).

**Impacto:** se prod subir com o segredo de dev (ou ele vazar), atacante **forja
qualquer token** — vira qualquer device/tenant, qualquer operador, qualquer
`branch_id`/`roles`/`plan_id` no JWT. Comprometimento total do multi-tenant.

**Fix:** `config/runtime.exs` lendo `SECRET_KEY_BASE` de env (gerado por
`mix phx.gen.secret`); nunca commitar; segredo distinto por ambiente. Considerar
segredo separado p/ JWT.

---

## 🟠 VULN-5 — `user_id` da venda forjável
`sync_changeset` casta `user_id` e o servidor não o impõe. Device pode atribuir a
venda a **qualquer usuário existente** (inclusive de outro tenant, pois não há
`assoc_constraint` de tenant). FK só exige que o id exista.
**Impacto:** spoof de "quem vendeu" → fura trilha/comissão/responsabilização.
**Fix:** derivar `user_id` do contexto autenticado (claim do JWT do operador) e
sobrescrever no servidor.

## 🟠 VULN-6 — Login sem rate-limit
`/api/v1/auth/login` e `/api/v1/login` só usam bcrypt. Sem throttle por IP/usuário
nem lockout. **Brute force / credential stuffing** viáveis.
**Fix:** Hammer/PlugAttack — limitar tentativas por IP+username, backoff progressivo.

## 🟡 VULN-7 — Enumeração de tenant
`auth_controller` responde `tenant_unavailable` vs `invalid_credentials`
(`:24-28`). Atacante distingue tenant existente de inexistente.
**Fix:** mensagem única `invalid_credentials` para ambos os casos.

## 🟡 VULN-8 — `certificate_password` em texto plano
`companies.certificate_password` (certificado A1 NFC-e) sem cifragem em repouso.
Dump de DB expõe credencial fiscal.
**Fix:** cifrar coluna (Cloak.Ecto) ou cofre externo (Vault/KMS).

## 🟡 VULN-9 — Device token de 1 ano sem rotação
`ValenPos.Auth` emite token com `max_age` 1 ano. Revogação só por `is_active` no
connect. Token roubado vale meses.
**Fix:** TTL menor + refresh, e/ou lista de revogação consultada no `handle_in`.

---

## ✅ Pontos fortes (o que resistiu)
- **Sem SQL injection:** todo acesso via Ecto parametrizado; `Repo.query!` usa binds.
- **Sem alg-confusion no JWT:** `ValenPos.JWT.verify` recomputa HMAC fixo (não lê
  `alg` do header) e usa `Plug.Crypto.secure_compare` (constante).
- **Escopo de leitura isolado:** `tenant_id` sempre do token nos GET (products/sales),
  sem IDOR de leitura. `tenant_id` de catálogo imposto no servidor (`ensure_tenant`).
- **Idempotência forte:** `correlation_id` + `sync_receipts` evitam replay duplicado.
- **Senha:** bcrypt + `no_user_verify` (mitiga timing).

---

## 🎯 Plano de remediação (ordem)
1. **VULN-4** (segredo/prod) — base de toda a confiança em tokens.
2. **VULN-2/3** (ownership de produto) — uma validação resolve ambos.
3. **VULN-1/5** (impor `status`/`user_id` no servidor).
4. **VULN-6/7** (rate-limit + erro único de login).
5. **VULN-8/9** (cifrar cert + rotação de token).

> PoC reproduzível: `mix test test/valen_pos/security_poc_test.exs` (3/3 confirmam).
> Ao corrigir, inverter as asserções → vira suíte de regressão de segurança.
