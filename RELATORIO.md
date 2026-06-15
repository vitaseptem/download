# 🛡️ VALEN POS — Relatório de Segurança · VULN-6 mitigada (Rate-Limit)

> Astraz Studio · 2026-06-15 · hardening de autenticação nível bancário.

## VULN-6 — Login sem rate-limit/lockout (brute force) → **FECHADA**

Proteção de rate-limiting nos endpoints de autenticação do backend Elixir com
**Hammer + backend ETS** (em memória, nativo do Erlang, sem serviço externo).

### O que foi feito
| Passo | Item | Detalhe |
|-------|------|---------|
| 1 | Dependência | `{:hammer, "~> 6.1"}` em `mix.exs` (`hammer 6.2.1` resolvido via `mix deps.get`) |
| 2 | Config | `config/config.exs`: `Hammer.Backend.ETS` (`expiry_ms: 60_000`, `cleanup_interval_ms: 60_000`); limite em `:valen_pos, :login_rate_limit` = `{60_000, 5}` |
| 3 | Plug | `ValenPosWeb.Plugs.RateLimit` — chave `"login:<ip>"`, `Hammer.check_rate/3`; excedeu → **429** `%{error: "too_many_requests", message: "Muitas tentativas. Tente novamente em um minuto."}`; **fail-open** se o backend falhar |
| 4 | Rotas | pipeline `:rate_limit_auth` aplicado a `POST /api/v1/auth/login` (PDV) e `POST /api/v1/login` (painel) |
| 5 | Testes | Cenário de ataque: 6 logins do mesmo IP → 5 normais (401), 6º **429**. IP único por teste no `ConnCase` p/ isolar o ETS global |
| 6 | Validação | `mix compile --warnings-as-errors` limpo · `mix test` verde |

### Regra de bloqueio
- **5 tentativas / minuto / IP**. 6ª em diante → `429 Too Many Requests` até a janela expirar.
- Limite parametrizável por env (`:login_rate_limit`), sem recompilar a lógica.
- Janela e limpeza ETS de 60s (memória estável).

### Comportamento seguro
- **Fail-open**: erro no backend do Hammer nunca derruba o login (pior caso: não limita; nunca nega indevidamente).
- Combina com VULN-7 (erro de login unificado) → atacante não enumera contas **e** é barrado por taxa.

---

## Status do build — 100% verde
```
mix compile --warnings-as-errors   → limpo (0 avisos)
mix test                           → 56 testes, 0 falhas
```
O teste de regressão `auth_controller_test.exs` ("VULN-6 — rate-limit") prova o 429 na 6ª tentativa.

---

## Postura de segurança consolidada (rodadas anteriores)
| ID | Vulnerabilidade | Status |
|----|-----------------|--------|
| VULN-1 | Mass assignment `status` da venda | ✅ imposto pelo servidor |
| VULN-2 | Cross-tenant em item de venda | ✅ validação de ownership |
| VULN-3 | Cross-tenant em stock_movements | ✅ validação de ownership |
| VULN-4 | Segredo hardcoded/compartilhado | ✅ `runtime.exs` + env |
| VULN-5 | Spoof de `user_id` (operador) | ✅ imposto pelo contexto |
| VULN-6 | Login sem rate-limit | ✅ **Hammer/ETS (esta rodada)** |
| VULN-7 | Enumeração de tenant | ✅ erro unificado |
| Item 1 | `EnsureCreated` destrutivo (C#) | ✅ EF Migrations |
| Item 5 | `sync_queue` infinita (C#) | ✅ purge 30d |
| Item 6 | DLQ punindo offline (C#) | ✅ transporte × negócio |
| Item 14 | Binários no git | ✅ `.gitignore` raiz |

## Pendências (próxima rodada)
- **VULN-8** cifrar `certificate_password` (Cloak/Vault).
- **VULN-9** rotação/TTL menor do device token.
- Unificar rotas admin no JWT (`AuthenticateTenant` ainda usa Phoenix.Token).
- Atrás de proxy/LB em prod: usar `X-Forwarded-For` no `RateLimit` (hoje usa `remote_ip` direto).

> Backend compila limpo e suíte 100% verde. VULN-6 fechada e coberta por teste.
