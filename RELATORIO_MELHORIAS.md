# 🔎 VALEN POS — Market Edition · Relatório de Melhorias

> Análise técnica do projeto (backend Elixir/Phoenix + frontend C#/.NET 8 Avalonia).
> Data: 2026-06-15. Stack offline-first, hierarquia multi-tenant, motor Outbox/DLQ, UUIDv7.

---

## 🔴 CRÍTICO (corrigir antes de produção real)

### 1. C# usa `EnsureCreated()`, não migrations EF
- **Onde:** `clients/ValenPos.Desktop/src/ValenPos.Desktop.UI/App.axaml.cs:39`
- Schema local SQLite é criado uma vez. Ao atualizar o app (novas colunas: `sale_unit`, `Cost`, `stocks`, `sync_queue`...), `EnsureCreated` **não aplica diff** em banco existente → caixa antigo quebra no upgrade.
- **Fix:** trocar por `Database.Migrate()` + `dotnet ef migrations`. Sem isso, todo update de schema exige apagar o `.db` do cliente (perde fila offline não sincronizada).

### 2. RBAC declarado mas não aplicado
- JWT carrega `roles`, mas **nenhum handler/rota verifica papel**. `gerente`/`operador`/`supervisor` são decorativos. Qualquer usuário logado faz tudo.
- **Fix:** plug de autorização por role + checagem nos handlers sensíveis (cancelamento, ajuste de estoque, sangria).

### 3. Dois sistemas de token incompatíveis
- `/api/v1/auth/login` emite **JWT**; mas `AuthenticateTenant` (`plugs/authenticate_tenant.ex:26`) valida **Phoenix.Token admin** (`verify_admin_token`). O JWT do PDV **não acessa** `/api/v1/products|sales|devices`.
- Frontend `AuthTokenHandler` injeta o JWT como Bearer → essas rotas dariam 401. Unificar no JWT (plug verifica `ValenPos.JWT`) ou documentar a separação.

### 4. Sem `config/prod.exs`
- Backend não tem config de produção (secret, DB, host, SSL). `secret_key_base` hardcoded em dev/test. Deploy do backend impossível hoje.

---

## 🟠 ALTO

### 5. `sync_queue` cresce sem limite
- Itens `synced` (com `SyncedAt`) nunca são removidos. Após meses de vendas, tabela enorme no SQLite do caixa. Falta job de purge (ex.: deletar synced > 30 dias).

### 6. DLQ pune offline legítimo
- `SyncEngine` incrementa `attempts` em falha de conexão e manda p/ `failed_permanently` em 5. Queda de rede no meio de push repetida = venda **boa** vira DLQ e some do retry. Conflita com "offline-first".
- **Fix:** separar falha de **transporte** (backoff infinito) de **rejeição de negócio** (DLQ).

### 7. Contexto `Audit` morto
- `ValenPos.Audit` existe (PASSO 5) mas **zero call sites**. Nenhuma ação gera trilha.
- **Fix:** logar login, revogação de device, ajuste de inventário, cancelamento.

### 8. Login sem rate-limit/lockout
- Só bcrypt. Sem trava por tentativas/IP → brute force. Adicionar throttle (ex.: Hammer) em `/auth/login` e `/login`.

### 9. `certificate_password` em texto plano
- `companies.certificate_password` (cert NFC-e) sem cifragem. Dado sensível fiscal. Cifrar em repouso (Cloak/Vault).

---

## 🟡 MÉDIO

### 10. Cobertura de testes com buracos
- Só `sales_test` no nível de contexto. **Sem teste direto** de `Inventory` (apply_movement, inventory_count, idempotência de movimento), `Org`, `Accounts.authenticate_pos` (coberto só via controller), `ValenPos.UUIDv7` (versão/monotonia), `ValenPos.JWT` (expiração/tamper).
- C# **sem testes** (0 projetos de teste).

### 11. Cursor de catálogo ignora `stocks`
- `GetCatalogCursorAsync` usa `max(product.updated_at)`. Mudança só de saldo não avança o cursor → re-pull de produtos a cada ciclo. Cursor deveria ser `max(product.updated_at, stock.updated_at)`.

### 12. `inventory_counted` tem corrida read-modify-write
- `apply_inventory_count` lê saldo atual, calcula delta, aplica via `inc`. Duas contagens simultâneas podem divergir. Usar `set` absoluto no upsert em vez de `current + delta`.

### 13. JWT sem refresh/revogação
- TTL fixo 12h, sem renovação nem blacklist. Operador deslogado só na expiração.

### 14. Sem `.gitignore` na raiz
- `dist/` (98M), `_build/`, `deps/` soltos. Risco de commit acidental de binário/segredo se a raiz virar repo git.

---

## 🟢 BAIXO / OPS

| # | Item | Ação |
|---|------|------|
| 15 | Sem CI | `.github/workflows`: `mix test` + `dotnet build/test` + publish em tag |
| 16 | Binário no histórico git | Migrar `.zip` para **GitHub Releases** (assets fora do histórico) ou Git LFS |
| 17 | `CompositionRootExample.cs` | Código de exemplo morto — remover ou marcar |
| 18 | KPIs do `AdminViewModel` | Placeholders fixos — ligar a agregação real de vendas |
| 19 | Observabilidade do sync | Expor profundidade da fila / contagem DLQ na UI do operador |

---

## ⭐ Top 5 prioridade

1. **EF Migrations no C#** (#1) — sem isso, todo upgrade quebra caixa em campo.
2. **Unificar auth no JWT + aplicar RBAC** (#2, #3).
3. **`config/prod.exs` + cifrar segredos** (#4, #9).
4. **Purge da `sync_queue` + revisar DLQ vs offline** (#5, #6).
5. **Trilha de auditoria viva + rate-limit login** (#7, #8).

---

## ✅ Estado atual (o que já está sólido)

- Backend: `mix compile --warnings-as-errors` limpo; `mix test` = **51 testes, 0 falhas**.
- Frontend: `dotnet build -c Debug` = 0 erros / 0 avisos.
- Arquitetura UUIDv7 ponta-a-ponta, hierarquia tenant→company→branch→cash_register, estoque por filial, ledger de movimentos, idempotência absoluta por `correlation_id` (sync_receipts), Outbox/DLQ no cliente.
- Build de produção Windows x64 single-file gerado e publicado.

> Gerado por Claude Code — análise técnica do repositório VALEN POS.
