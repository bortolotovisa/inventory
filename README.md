# WMS Chapas — Sistema de Gerenciamento de Estoque de Chapas

Especificação funcional e técnica completa de um WMS (Warehouse Management System) para o estoque de chapas (MDF, MDP, compensado, fórmica) de uma **fábrica de móveis planejados**, operado via **iPad acoplado à empilhadeira**, com foco em **simplicidade extrema para o operador**: poucos cliques, botões grandes, fluxo guiado e leitura de etiquetas.

## Conceitos-chave

| Conceito | Definição |
|---|---|
| **Pallet** | Unidade de estocagem. Todo pallet tem **ID único** e **etiqueta QR/DataMatrix**. Contém N chapas de um único material/lote. |
| **Endereço** | Posição física única no armazém (`RUA-COLUNA-NÍVEL`), representada na **planta baixa digital** com coordenadas x,y. |
| **Work Order (WO)** | Ordem de produção vinda do PCP/ERP. **Toda saída de material — inclusive pedido extra — é obrigatoriamente vinculada a uma WO.** |
| **Movimentação** | Todo evento de estoque é registrado em um **ledger imutável** (`stock_movements`), base da auditoria e da IA. |

## Processos cobertos

1. **Recebimento** — conferência contra pedido de compra, criação de pallets, impressão de etiquetas.
2. **Endereçamento automático** — o sistema sugere a melhor posição (slotting inteligente); o operador confirma com dois scans.
3. **Movimentações** — transferência de pallet entre endereços, sempre guiada e rastreada.
4. **Retirada para produção** — scan da WO → lista de materiais → mapa com pallets destacados → scan de confirmação → baixa automática.
5. **Devolução** — sobras retornam ao estoque com quantidade atualizada e re-endereçamento.
6. **Inventário** — contagem cíclica dirigida e inventário geral, com ajustes auditados.
7. **Auditoria** — trilha completa de quem/quando/onde/quanto para todo evento.

## Índice da especificação

| Documento | Conteúdo |
|---|---|
| [01 — Visão Geral](docs/01-visao-geral.md) | Contexto, personas, princípios de UX, regras de negócio, escopo |
| [02 — Arquitetura](docs/02-arquitetura.md) | Arquitetura de solução, componentes, offline-first, integrações, eventos |
| [03 — Planta Baixa e Endereçamento](docs/03-planta-e-enderecamento.md) | Modelo de localização, planta digital, algoritmo de endereçamento automático |
| [04 — Modelo de Dados](docs/04-modelo-de-dados.md) | Diagrama ER, DDL PostgreSQL, ledger de movimentações |
| [05 — APIs](docs/05-apis.md) | Contratos REST + WebSocket, exemplos de payload, códigos de erro |
| [06 — Telas e Fluxos](docs/06-telas-e-fluxos.md) | Design system de chão de fábrica, wireframes, navegação, fluxos guiados |
| [07 — Inteligência Artificial](docs/07-inteligencia-artificial.md) | Previsão de falta, otimização de rotas, anomalias, slotting, dashboards inteligentes |
| [08 — NFRs, Segurança e Roadmap](docs/08-nfr-seguranca-roadmap.md) | Requisitos não funcionais, segurança/LGPD, plano de fases |

## Stack de referência

- **Frontend operador:** PWA React + TypeScript, offline-first, otimizada para iPad (Safari), scanner por câmera ou Bluetooth.
- **Backend:** NestJS (Node/TypeScript), PostgreSQL 16, Redis (filas/cache), WebSocket para mapa em tempo real.
- **Serviço de IA:** Python + FastAPI (previsão, anomalias, slotting, rotas) + LLM (Claude API) para dashboards conversacionais.
- **Etiquetas:** impressora térmica ZPL (Zebra) via print-server.

> A stack é uma recomendação de referência: os contratos de API e o modelo de dados são independentes de tecnologia.
