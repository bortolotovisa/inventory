# 08 — Requisitos Não Funcionais, Segurança e Roadmap

## 1. Requisitos não funcionais

| Categoria | Requisito |
|---|---|
| **Desempenho** | Scan → feedback visual: ≤ 200 ms (validação local); scan → confirmação do servidor: p95 ≤ 800 ms na LAN; renderização do mapa ≤ 1 s com 2.000 posições |
| **Disponibilidade** | Operação de chão de fábrica tolera queda total do servidor por até 30 min via modo offline (leituras da réplica local + outbox); RTO servidor 1 h, RPO 5 min |
| **Escala** | Dimensionado para 5.000 endereços, 10.000 pallets ativos, 50.000 movimentos/mês, 30 dispositivos simultâneos — uma instância modesta de cada serviço atende com folga; nada impede multi-armazém (modelo já é multi-warehouse) |
| **Compatibilidade** | iPad (Safari/PWA) geração ≥ 9ª; painel web: Chrome/Edge/Firefox atuais; impressoras ZPL II |
| **Usabilidade** | Tarefa de picking completa sem tocar em campo de texto; onboarding de operador < 1 h; WCAG AA no painel, AAA de contraste na PWA |
| **Manutenibilidade** | Cobertura de testes nos serviços de domínio ≥ 80%; contratos de API versionados (`/v1`); migrações de banco versionadas e reversíveis |
| **Observabilidade** | Trace de ponta a ponta por `trace_id`; métricas de negócio (tarefas/h, profundidade de outbox) e técnicas expostas em Prometheus; retenção de logs 90 dias |

## 2. Segurança

### 2.1 Autenticação e autorização

- Operadores: **crachá (QR) + PIN** — rápido com luva, sem senha memorizada; JWT de 8 h vinculado ao `device_id`; troca de operador não derruba o estado da tarefa (tarefas pertencem ao usuário, não ao dispositivo).
- Painel web: e-mail + senha + TOTP para `SUPERVISOR`/`ADMIN`.
- **RBAC** por papel (doc 04 §6): matriz ação × papel mantida no código (guards NestJS) e auditada; princípio do menor privilégio — operador não vê custos, comprador não movimenta estoque.
- Dispositivos registrados: iPad precisa de enrollment (QR de provisionamento do admin); dispositivo perdido é revogado no painel.

### 2.2 Proteção de dados e trilha

- TLS em tudo (inclusive LAN); segredos em vault/variáveis, nunca em código.
- Ledger append-only com regras no banco (doc 04 §4) + `audit_log` para ações administrativas; relógio do servidor como fonte de ordem.
- **LGPD:** dados pessoais mínimos (nome, e-mail, crachá); base legal = execução de contrato de trabalho/legítimo interesse; relatórios de produtividade individual acessíveis apenas a supervisor+RH; retenção e anonimização de ex-funcionários parametrizadas (crachá revogado, nome preservado no ledger por obrigação de auditoria — documentado no RIPD).
- Backups criptografados, restore testado trimestralmente.

### 2.3 Superfícies específicas

- Webhooks assinados (HMAC) nos dois sentidos da integração ERP; allow-list de IP opcional.
- `POST /ai/ask`: SQL gerado por LLM roda com usuário de banco read-only restrito a views curadas; allow-list de tabelas; sem interpolação de credenciais no prompt.
- Print server só aceita jobs da API (mTLS na LAN).

## 3. Estratégia de testes

| Camada | Abordagem |
|---|---|
| Domínio | Unit tests dos invariantes (RN-01..10): baixa exige WO, ledger imutável, capacidade de endereço, FIFO |
| API | Testes de contrato (OpenAPI) + testes de idempotência (replay da mesma `Idempotency-Key`) |
| Offline | Testes E2E simulando perda de rede no meio do picking (Playwright + service worker throttling): fila drena, nada duplica, conflito vira tarefa de correção |
| Carga | 30 dispositivos virtuais × pico de recebimento + picking simultâneos |
| Campo | Piloto com 1 empilhadeira e 1 rua real antes do go-live total (ver roadmap) |

## 4. Roadmap de implantação

### Fase 0 — Fundação (semanas 1–4)
- Modelo de dados + ledger + API Core (catalog, layout, inventory, iam).
- Editor de planta baixa + impressão de etiquetas.
- **Marco:** etiquetar fisicamente endereços e pallets do estoque atual (mutirão de carga inicial com o app de contagem).

### Fase 1 — MVP operacional (semanas 5–12)
- PWA operador: guarda, retirada por WO, movimentação, consulta avulsa — com offline-first desde o dia 1.
- Recebimento na doca; devolução; pedido extra com aprovação.
- Endereçamento automático **por regras**; rota por serpentina.
- Painel: torre de controle, WOs, auditoria, aprovações.
- Integração ERP fase 1 (import de PO/WO por CSV/API simples).
- **Marco de go-live:** duas semanas de operação paralela (WMS + processo antigo), depois corte.

### Fase 2 — Otimização (semanas 13–20)
- Contagem cíclica dirigida; inventário geral por zona.
- Rota de picking otimizada (TSP) + re-otimização por exceção; interleaving guarda/picking.
- Previsão de ruptura (após acumular ledger) + alertas a compras.
- Dashboards com narrativa diária; webhooks ERP completos (custeio por WO).

### Fase 3 — Inteligência plena (semanas 21+)
- Slotting com IA (re-ranking online + re-slotting semanal).
- Detecção de anomalias completa + contagens dirigidas por anomalia.
- Assistente conversacional (`/ai/ask`).
- Batching multi-WO; visão computacional de conferência (piloto).
- Avaliar RFID em portais de doca (o modelo de dados não muda).

## 5. Riscos e mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| Wi-Fi ruim no galpão | Operação trava | Offline-first (doc 02 §3) + site survey antes do go-live + APs direcionais nas ruas |
| Operador contorna o sistema ("depois eu lanço") | Estoque volta a mentir | UX mais rápida que o papel (2 scans), etiqueta obrigatória para a serra aceitar material (combinado com produção), auditoria de WO sem movimentos |
| Etiquetas de pallet danificadas (chapas raspam) | Scan falha | 2 vias por pallet, etiqueta plástica lateral, fallback "não consigo escanear" com busca por posição, reimpressão em 2 toques |
| Carga inicial malfeita | Sistema nasce mentindo | Mutirão com dupla contagem + 2 semanas de contagem cíclica agressiva pós-carga |
| ERP sem API decente | Integração emperra | Fase 1 aceita CSV/manual; contratos internos independem do ERP |
| Dependência da IA | Operação para se o serviço cair | Fallbacks determinísticos em 100% das funções de IA (doc 07) |

## 6. Glossário

| Termo | Significado |
|---|---|
| **WMS** | Warehouse Management System — sistema de gestão de armazém |
| **WO (Work Order)** | Ordem de produção/corte liberada pelo PCP |
| **PO (Purchase Order)** | Pedido de compra ao fornecedor |
| **Putaway / Guarda** | Levar pallet da doca ao endereço de estocagem |
| **Picking / Retirada** | Separar material do estoque para a produção |
| **Slotting** | Definição de onde cada material deve ficar no armazém |
| **FIFO** | First In, First Out — consumir primeiro o lote mais antigo |
| **Ledger** | Registro contábil imutável de todos os movimentos |
| **Outbox** | Fila local de eventos aguardando sincronização |
| **PCP** | Planejamento e Controle da Produção |
| **Seccionadora** | Máquina que corta as chapas conforme o plano de corte |
