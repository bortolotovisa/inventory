# 05 — APIs

Base: `https://wms.fabrica.com/api/v1`. JSON, UTF-8. Autenticação: `Authorization: Bearer <JWT>`.

## 1. Convenções

- **Idempotência**: toda mutação de estoque exige header `Idempotency-Key: <uuid>` (gerado no cliente antes de entrar na outbox). Repetição da mesma chave retorna a resposta original com `X-Idempotent-Replay: true`.
- **Erros** (RFC 9457 problem+json):

```json
{
  "type": "https://wms/errors/wrong-pallet",
  "title": "Pallet não pertence a esta ordem",
  "status": 409,
  "detail": "PLT-004217 é MDF Branco 15mm; a WO pede 18mm.",
  "action_hint": { "text": "O pallet correto está em C-02-N1", "location_code": "C-02-N1" },
  "trace_id": "01J..."
}
```

  `action_hint` é contrato de UX: a PWA sempre tem o que mostrar ao operador em vermelho/âmbar.
- **Paginação**: cursor (`?cursor=...&limit=50`), resposta com `next_cursor`.
- **Sync offline**: `GET /sync?since=<sync_token>&scopes=tasks,layout,materials,pallets` retorna deltas + novo token.

### Códigos de erro de domínio (principais)

| `type` | HTTP | Quando |
|---|---|---|
| `wrong-pallet` | 409 | Scan de pallet fora da alocação |
| `fifo-violation` | 409 | Pallet certo, lote errado (aceita override com `override_reason`) |
| `location-occupied` | 409 | Guarda em endereço cheio |
| `location-incompatible` | 422 | Peso/altura/zona incompatível |
| `wo-not-active` | 422 | WO cancelada/concluída |
| `wo-required` | 422 | Tentativa de saída sem WO (RN-04) |
| `insufficient-stock` | 409 | Reserva impossível |
| `approval-required` | 403 | Extra acima do limite sem aprovação |
| `stale-sync` | 409 | Cliente offline demais; força ressincronização completa |

## 2. Autenticação

| Método | Rota | Descrição |
|---|---|---|
| POST | `/auth/badge` | `{badge_code, pin, device_id}` → JWT curto (8 h) para operador |
| POST | `/auth/login` | `{email, password}` (+ TOTP p/ admin) → JWT web |
| POST | `/auth/refresh` | Renovação silenciosa |

## 3. Recebimento (inbound)

| Método | Rota | Descrição |
|---|---|---|
| GET | `/purchase-orders?status=OPEN&supplier=` | POs esperados na doca |
| POST | `/receipts` | Abre recebimento `{purchase_order_id?, nfe_key?}` |
| POST | `/receipts/{id}/items` | Confere item `{material_id, qty_received, qty_damaged, supplier_batch, issue_notes?, issue_photos?}` |
| POST | `/receipts/{id}/pallets` | Cria pallet(s) do item conferido — ver abaixo |
| POST | `/receipts/{id}/complete` | Fecha; divergências viram ocorrências p/ Compras |

**POST `/receipts/{id}/pallets`** — cria pallet e dispara etiqueta:

```json
// request
{ "receipt_item_id": "…", "qty_units": 42, "copies": 2, "printer": "DOCA-ZEBRA-1" }
// response 201
{
  "pallet": { "id": "…", "label_code": "PLT-004217", "status": "RECEIVED",
              "location": "DOCA-01", "material": { "sku": "MDF-BCO-TX-18-2750X1850" } },
  "putaway_task_id": "…",          // tarefa de guarda já criada
  "print_job_id": "…"
}
```

## 4. Endereçamento e movimentação

| Método | Rota | Descrição |
|---|---|---|
| GET | `/tasks?assigned_to=me&type=PUTAWAY,MOVE&status=PENDING` | Fila do operador (priorizada) |
| POST | `/pallets/{label}/putaway/suggest` | Calcula endereço sugerido (motor doc 03 + IA doc 07) |
| POST | `/pallets/{label}/putaway/confirm` | `{scanned_location_code, suggested_location_id, idempotency…}` — grava `GUARDA`; se `scanned ≠ suggested` e válido, registra override |
| POST | `/movements/transfer` | `{pallet_label, from_scan, to_scan, reason_code?}` — grava `TRANSFERENCIA` |
| POST | `/pallets/{label}/pickup` | Marca "no garfo" (`GARFO-EMPxx`) ao retirar do rack |

**POST `/pallets/PLT-004217/putaway/suggest`** — resposta:

```json
{
  "suggestion": {
    "location_id": "…", "location_code": "B-04-N0",
    "reasons": ["Material classe A — perto da seccionadora", "3 pallets do mesmo material na rua B"],
    "source": "AI",                      // AI | RULES (fallback)
    "route": { "polyline": [[2,3],[2,14],[7.5,14]], "distance_m": 27 }
  },
  "alternatives": [ { "location_code": "B-05-N1", "score": 0.91 } ]
}
```

## 5. Picking (retirada para produção)

| Método | Rota | Descrição |
|---|---|---|
| GET | `/work-orders/{wo_number}` | Resolve o QR da WO: itens, cobertura, status |
| POST | `/work-orders/{wo_number}/picking-tasks` | Gera/retoma tarefa de picking p/ o operador logado (aloca reservas, calcula rota) |
| GET | `/picking-tasks/{id}` | Tarefa com linhas ordenadas pela rota + geometria p/ mapa |
| POST | `/picking-tasks/{id}/lines/{lineId}/scan` | Valida scan do pallet (síncrono, também validado offline) |
| POST | `/picking-tasks/{id}/lines/{lineId}/confirm` | Confirma retirada → **movimento `SAIDA_PRODUCAO` + baixa automática** |
| POST | `/picking-tasks/{id}/lines/{lineId}/exception` | `{code: PALLET_MISSING\|QTY_SHORT\|ACCESS_BLOCKED, note?}` → realoca na hora |
| POST | `/picking-tasks/{id}/complete` | Resumo final; WO → `PICKED`/`PARTIAL` |

**GET `/picking-tasks/{id}`** — resposta (o contrato da tela de mapa):

```json
{
  "id": "…", "work_order": { "wo_number": "WO-8841", "project_ref": "Cozinha Fam. Souza" },
  "status": "IN_PROGRESS",
  "lines": [
    {
      "id": "L1", "order": 1, "status": "CONFIRMED",
      "pallet": { "label_code": "PLT-004102", "qty_units": 12 },
      "material": { "sku": "MDF-BCO-TX-18-2750X1850", "description": "MDF Branco TX 18mm" },
      "qty_planned": 4, "qty_confirmed": 4,
      "location": { "code": "B-02-N0", "x_m": 6.2, "y_m": 12.1 }
    },
    {
      "id": "L2", "order": 2, "status": "PENDING",
      "pallet": { "label_code": "PLT-004217", "qty_units": 42 },
      "material": { "sku": "MDF-CARV-18-2750X1850", "description": "MDF Carvalho 18mm" },
      "qty_planned": 6,
      "location": { "code": "C-07-N0", "x_m": 14.8, "y_m": 21.3 }
    }
  ],
  "route": { "polyline": [[2,3],[6.2,12.1],[14.8,21.3],[46,15]], "total_distance_m": 96,
             "source": "AI" }
}
```

**Confirmação (o momento da baixa automática):**

```json
// POST /picking-tasks/{id}/lines/L2/confirm
{ "scanned_label": "PLT-004217", "qty_confirmed": 6, "client_ts": "2026-07-15T14:02:11-03:00" }
// 200
{
  "movement_id": "…",
  "pallet_after": { "label_code": "PLT-004217", "qty_units": 36, "status": "STORED" },
  "wo_item_progress": { "picked": 6, "required": 6, "complete": true },
  "next_line": { "id": "L3", "location_code": "A-01-N0" }   // guia o próximo passo
}
```

## 6. Pedido extra e devolução

| Método | Rota | Descrição |
|---|---|---|
| POST | `/work-orders/{wo_number}/extra-requests` | `{material_id, qty, reason_code, reason_text?}` → 201 ou `approval-required` |
| POST | `/extra-requests/{id}/approve` \| `/reject` | Supervisor (web ou push no iPad do supervisor) |
| POST | `/work-orders/{wo_number}/returns` | `{pallet_label?, material_id?, qty, condition}` → movimento `DEVOLUCAO` + tarefa de guarda |

## 7. Inventário

| Método | Rota | Descrição |
|---|---|---|
| GET | `/count-sessions?status=OPEN&assigned_to=me` | Contagens do dia |
| POST | `/count-items/{id}/submit` | `{location_scan, pallet_scans: [{label, qty}]}` → `MATCH` ou divergência |
| POST | `/count-sessions/{id}/close` | Fecha (aprovações pendentes bloqueiam) |
| POST | `/count-items/{id}/approve-adjustment` | Supervisor aprova → movimento `AJUSTE_INVENTARIO` |

## 8. Mapa, consultas e auditoria

| Método | Rota | Descrição |
|---|---|---|
| GET | `/layout/current` | JSON da planta (doc 03 §2) + versão |
| GET | `/map/occupancy` | Estado por endereço (livre/ocupado/reservado/bloqueado) — delta via WebSocket |
| GET | `/pallets/{label}` | Ficha do pallet (o "quem é esse?" de qualquer scan avulso) |
| GET | `/pallets/{label}/timeline` | Linha do tempo (ledger) do pallet |
| GET | `/materials/{sku}/stock` | Saldo, reservas, pallets, cobertura em dias |
| GET | `/audit/movements?material=&wo=&user=&type=&from=&to=` | Consulta do ledger + `?format=csv` |

## 9. IA (servidos pelo serviço FastAPI, roteados sob o mesmo gateway)

| Método | Rota | Descrição |
|---|---|---|
| GET | `/ai/stockouts?horizon=30` | Previsões de ruptura por material |
| GET | `/ai/anomalies?status=OPEN` | Alertas de anomalia com explicação |
| POST | `/ai/slotting/replan` | Dispara estudo de re-slotting; resultado vira sugestões |
| POST | `/ai/routes/optimize` | Interno (API Core → IA): pontos → sequência ótima |
| POST | `/ai/ask` | Dashboard conversacional: `{question}` → `{answer, sql?, chart_spec?}` (doc 07 §6) |

## 10. WebSocket `/realtime`

Cliente assina canais após autenticar:

| Canal | Evento exemplo | Consumidor |
|---|---|---|
| `tasks.{user_id}` | `task.assigned`, `task.cancelled` | PWA operador |
| `map.{warehouse_id}` | `occupancy.changed {location_code, state}` | Mapa (iPad e painel) |
| `wo.{wo_id}` | `picking.progress {picked, total}` | Painel PCP |
| `alerts.supervisor` | `extra_request.pending`, `anomaly.critical` | Painel supervisor |

## 11. Webhooks de saída (ERP e outros)

`POST` assinado (HMAC-SHA256 no header `X-WMS-Signature`), retry exponencial 24 h:

- `receipt.completed`, `movement.created` (filtro configurável), `inventory.adjusted`, `stockout.predicted`, `wo.picked`.
