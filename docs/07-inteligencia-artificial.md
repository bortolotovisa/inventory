# 07 — Funcionalidades de Inteligência Artificial

Princípios: (1) **IA nunca no caminho crítico** — toda função tem fallback determinístico e a operação continua se o serviço cair; (2) **toda sugestão é explicável** — o operador/supervisor vê o porquê em uma frase; (3) **feedback humano vira dado de treino** — overrides, exceções e falsos positivos são registrados e realimentam os modelos.

O ledger `stock_movements` é o dataset perfeito: cada linha é um evento rotulado com material, quantidade, posições, WO, operador e timestamp.

## 1. Previsão de falta de material (stockout forecast)

**Problema:** a produção para quando falta chapa; comprar demais empata capital em material que risca e empena.

**Solução:** job diário calcula, por material, a **cobertura em dias** e a **data provável de ruptura**, combinando:

- **Demanda firme:** WOs liberadas e programadas (BOM já conhecido) — não é previsão, é soma.
- **Demanda estatística:** consumo histórico do ledger (sazonalidade semanal/mensal, tendência) — modelo de série temporal (LightGBM com features de calendário; Prophet como baseline). Inclui a distribuição dos **pedidos extras** por material (o "consumo invisível" de erro de corte).
- **Oferta futura:** POs abertas com `expected_at` + `lead_time_days` do fornecedor + confiabilidade de prazo do fornecedor (atraso médio observado nos recebimentos).

**Saída** (`ai_stockout_forecasts` + alertas): "MDF Carvalho 18mm rompe em **9 dias** (confiança 0,87). Lead time do fornecedor: 12 dias — **pedido deveria ter saído há 3 dias**. Sugestão: 60 chapas." O comprador recebe a lista priorizada por criticidade × valor.

**Fallback:** `min_stock_units` do cadastro (ponto de pedido fixo).

**Métrica de qualidade:** % de rupturas reais que tinham alerta ≥ 7 dias antes; taxa de falso alarme.

## 2. Otimização de rotas de picking e guarda

**Problema:** a ordem das coletas define quantos metros a empilhadeira roda.

**Solução:**

- Grafo de corredores + matriz de distâncias pré-computada (doc 03 §2).
- Sequenciamento das linhas da tarefa = TSP pequeno (5–20 pontos): resolvido por heurística **nearest-neighbor + 2-opt** (milissegundos, ótimo o suficiente); origem = posição atual da empilhadeira, destino = `PROD-SECC`.
- **Re-otimização em tempo real:** exceção numa linha (acesso bloqueado, pallet ausente) → re-sequencia o restante na hora.
- **Batching inteligente (fase 2):** agrupar 2–3 WOs compatíveis numa única volta quando a empilhadeira comporta; sugerir "aproveite a volta" — guarda pendente cujo destino está no caminho do picking (interleaving).

**Fallback:** ordenação por rua/coluna (serpentina).

**Métrica:** metros/linha de picking antes × depois (o próprio ledger + timestamps mede).

## 3. Detecção de anomalias

**Problema:** desvios, erros sistemáticos e "jeitinhos" só aparecem no inventário anual — tarde demais.

**Detectores** (job horário/diário → `ai_anomaly_alerts`, com severidade e explicação):

| Detector | Técnica | Exemplo de alerta |
|---|---|---|
| **Shrinkage por material** | Controle estatístico (EWMA) sobre divergências de contagem | "Branco TX 15mm: 4ª divergência negativa seguida em contagens — perda acumulada 9 chapas/30 dias" |
| **Movimento atípico** | Isolation Forest sobre features do movimento (hora, tipo, qty vs. típico do material, operador, posição) | "Saída de 14 chapas às 22:47 — 3σ acima do padrão do material e fora do turno do operador" |
| **Abuso de pedido extra** | Regras + quantis por WO/solicitante/motivo | "WOs do projetista X consomem 2,3× mais extras por motivo ERRO_CORTE que a mediana" |
| **Endereço instável** | Frequência de exceções `PALLET_MISSING` por endereço | "C-11 teve 5 'pallet não está aqui' em 2 semanas — etiqueta trocada?" |
| **Sequência impossível** | Validação de invariantes no ledger | "Pallet escaneado em picking 40 s após guarda a 80 m de distância" |

**Ação integrada:** anomalias de estoque geram automaticamente **contagem cíclica dirigida** (doc 01 §5.7) no endereço/material suspeito — o sistema investiga a si mesmo. Falso positivo marcado pelo supervisor reduz o peso do detector (threshold adaptativo).

## 4. Sugestão de endereçamento (slotting inteligente)

**Duas camadas sobre o motor de regras do doc 03:**

1. **Re-ranqueamento online (putaway):** o modelo ajusta o score dos candidatos elegíveis usando: previsão de demanda do material (§1) — material que vai sair muito esta semana ganha posição nobre mesmo sendo classe B histórica; afinidade de co-ocorrência (materiais que saem juntos nas mesmas WOs ficam próximos — mineração de itemsets no ledger); e taxa de override dos operadores (sugestões sistematicamente ignoradas perdem score).
2. **Re-slotting periódico (batch semanal):** simula o custo total de picking do mix atual de WOs contra layouts alternativos e propõe as N trocas de maior ganho: "Mover Carvalho 18 de D-02 para B-01: −18% de distância no mix atual (economia estimada 1,4 km/semana)". Aprovadas pelo supervisor, viram tarefas de transferência em horário ocioso.

**Fallback:** motor de regras puro (doc 03 §3).

## 5. Conferência assistida por visão computacional (fase 3, opcional)

- Foto do pallet no recebimento/devolução → modelo conta chapas pela lateral do pacote e estima divergência da quantidade declarada (chapas têm espessura conhecida: altura da pilha ÷ espessura ≈ contagem).
- Foto de avaria classificada automaticamente (canto batido, superfície riscada, empenamento) para padronizar ocorrências com fornecedores.
- Executa no próprio iPad (Core ML) para funcionar offline.

## 6. Dashboards inteligentes e assistente conversacional

**Dashboards com narrativa:** além dos KPIs (acurácia de inventário, giro por material, metros/picking, tempo doca→endereço, extras por projeto, aging de estoque), a IA gera um **resumo diário em linguagem natural**:

> "Terça, 15/07: 14 WOs separadas (média 11). Destaque negativo: tempo médio de guarda subiu 32% — 6 pallets esperaram >2h na doca entre 10h e 12h (pico de recebimento + só 1 empilhadeira ativa). Risco: Fórmica Preta 0.8mm rompe em 5 dias e não há PO aberta. Extras do dia: 7 chapas (R$ 1.240), 70% por ERRO_CORTE no projeto Souza."

**Assistente de perguntas (`POST /ai/ask`):** supervisor pergunta em português — "quanto MDF branco 18 saiu por semana no último trimestre?", "quais projetos mais estouram material?" — pipeline **LLM (Claude API) → SQL sobre views curadas e read-only** → resposta com tabela/gráfico + o SQL exibido para conferência. Guard-rails: apenas `SELECT` em views permitidas (allow-list), timeout, limite de linhas, sem dados pessoais além de nomes de usuário internos.

## 7. Dados, treino e operação dos modelos

| Aspecto | Abordagem |
|---|---|
| Feature store | Views materializadas sobre o ledger (consumo diário por material, distâncias percorridas, divergências por endereço) — recomputadas por job |
| Cold start | Sistema nasce 100% em regras determinísticas; modelos entram quando houver ≥ 8–12 semanas de ledger |
| Retreino | Semanal (forecast, slotting), diário (anomalias — thresholds), sob demanda após mudança de layout |
| Avaliação | Backtesting contra o próprio ledger (o que o modelo teria previsto vs. o que houve); painel de métricas por modelo |
| Explicabilidade | Toda sugestão persiste `features`/`reasons` legíveis (contrato dos docs 04 §7 e 05 §4) |
| Privacidade | Modelos usam operador como feature apenas para anomalia/auditoria interna; relatórios agregados não expõem ranking público de operadores (evitar gamificação punitiva) — política definida com o RH |
