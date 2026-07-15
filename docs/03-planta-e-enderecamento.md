# 03 — Planta Baixa Digital e Endereçamento

## 1. Modelo de localização

Hierarquia física:

```
Armazém (warehouse)
└── Zona (zone)            ex.: ESTOCAGEM-CHAPAS, PICKING-RAPIDO, DOCA, QUARENTENA, DEVOLUCAO
    └── Rua (aisle)        ex.: A, B, C
        └── Coluna (bay)   ex.: 01..12
            └── Nível (level)  ex.: 0 (chão), 1, 2   — porta-pallets ou pilha
```

**Código de endereço** (impresso na etiqueta da posição, formato humano + QR):

```
A-03-N0        rua A, coluna 3, chão
C-11-N2        rua C, coluna 11, nível 2
DOCA-01        endereço virtual de doca
QUAR-01        quarentena
GARFO-EMP01    endereço virtual "no garfo da empilhadeira 01"
PROD-SECC      endereço virtual "entregue na seccionadora"
```

Regras:

- Cada endereço físico tem **etiqueta única** afixada na longarina/coluna (QR + texto grande + cor da rua).
- Endereços têm atributos de restrição: `max_peso_kg`, `max_altura_mm`, `capacidade_pallets` (default 1), `aceita_material_classe` (ex.: nível 2 não aceita pallets acima de 800 kg).
- Endereços **virtuais** (doca, garfo, produção, quarentena) participam do mesmo modelo — todo pallet está sempre em exatamente um endereço, o que mantém o mapa 100% verdadeiro.

## 2. Planta baixa digital

A planta é um documento JSON versionado (`layout_versions`), editado no painel admin (editor visual arrastar-e-soltar sobre uma imagem de fundo opcional — a planta CAD da fábrica exportada em PNG/SVG).

```jsonc
{
  "version": 12,
  "warehouse_id": "wh-main",
  "canvas": { "width_m": 48.0, "height_m": 30.0 },
  "obstacles": [                       // paredes, pilares, máquinas — usados no cálculo de rota
    { "id": "pilar-1", "polygon": [[10,4],[10.6,4],[10.6,4.6],[10,4.6]] }
  ],
  "corridors": [                       // grafo de circulação da empilhadeira
    { "id": "corr-A", "polyline": [[2,3],[2,27]], "width_m": 3.5, "bidirectional": true }
  ],
  "zones": [
    { "code": "ESTOCAGEM", "polygon": [[4,2],[44,2],[44,28],[4,28]], "color": "#4A6FA5" }
  ],
  "locations": [
    {
      "code": "A-03-N0",
      "zone": "ESTOCAGEM",
      "x_m": 6.2, "y_m": 8.4,          // ponto de acesso frontal da posição
      "level": 0,
      "capacity_pallets": 1,
      "max_weight_kg": 1500,
      "max_height_mm": 2200
    }
  ],
  "pois": [
    { "code": "DOCA-01", "x_m": 2.0, "y_m": 2.0, "type": "dock" },
    { "code": "PROD-SECC", "x_m": 46.0, "y_m": 15.0, "type": "production_drop" }
  ]
}
```

- O **render no iPad** é SVG gerado a partir desse JSON: ruas rotuladas, posição do alvo pulsando, rota desenhada sobre os corredores, pallets da tarefa atual destacados com numeração da sequência (①②③).
- **Distâncias entre posições** são pré-computadas (matriz por grafo de corredores, Dijkstra) a cada publicação de versão de layout — usadas pelo endereçamento, pela rota de picking e pela IA. Nada é calculado "em linha reta através de parede".
- Alterar o layout gera nova versão; endereços extintos com pallet dentro bloqueiam a publicação até serem esvaziados (tarefas de transferência geradas automaticamente).

## 3. Endereçamento automático (putaway) — algoritmo base

Quando um pallet precisa de endereço (recebimento, devolução, re-slotting), a API Core executa o **motor de regras determinístico** abaixo. A IA (doc 07) apenas **re-ranqueia** os candidatos aprovados — nunca inventa posição inválida.

### 3.1 Filtros de elegibilidade (hard constraints)

1. Endereço `ATIVO` e com capacidade livre.
2. `peso_pallet ≤ max_peso_kg` e `altura_pallet ≤ max_altura_mm`.
3. Zona compatível com o status do material (avariado → só quarentena; saldo quebrado → preferir zona `PICKING-RAPIDO` se habilitada).
4. Sem reserva de posição pendente (outra tarefa de guarda em andamento para lá).

### 3.2 Ranqueamento (soft constraints — pesos parametrizáveis)

| Critério | Peso default | Racional |
|---|---|---|
| **Giro do material (classe ABC)** | 35% | Classe A perto do ponto de consumo (`PROD-SECC`); classe C nos fundos/níveis altos |
| **Proximidade de pallets do mesmo material** | 25% | Agrupar facilita contagem e picking multi-pallet |
| **Distância da doca (custo da guarda agora)** | 15% | Não atravessar o galpão à toa |
| **FIFO físico** | 15% | Lote mais novo atrás/acima do mais antigo do mesmo material, para a retirada natural respeitar FIFO |
| **Nível baixo para pallets pesados/cheios** | 10% | Segurança e velocidade |

Empate → menor código de endereço (determinismo total; dois cálculos seguidos dão a mesma resposta).

### 3.3 Comportamento na exceção

- Operador escaneia um endereço diferente do sugerido: se passa nos filtros hard, **aceita e registra override** (a IA aprende com esses desvios — se todo mundo ignora a sugestão da rua D, algo está errado no modelo ou na rua).
- Nenhum endereço elegível → pallet vai para zona `OVERFLOW` e o supervisor recebe alerta de capacidade.

## 4. Etiquetas

### 4.1 Pallet

```
┌─────────────────────────────────────┐
│  ██████████        PLT-004217       │
│  ██ QR   ██   MDF BRANCO TX 18mm    │
│  ██████████   2750×1850  ·  42 chp  │
│               Lote F2026-0711       │
│               Receb. 11/07/2026     │
│  ▍▍▍▍▍▍▍ (barra lateral cor do material p/ conferência visual)
└─────────────────────────────────────┘
```

- QR codifica apenas `PLT-004217` (dados vivem no sistema — etiqueta nunca fica desatualizada).
- Papel adesivo destrutível ou etiqueta plástica com abraçadeira (chapas não aceitam adesivo direto); 2 vias por pallet (frente e lateral) para leitura com pallet no rack.

### 4.2 Endereço

- QR + código gigante (leitura humana a 5 m) + faixa de cor por rua. Material: poliéster laminado, afixada na estrutura, nunca no chão (empilhadeira apaga).

## 5. Reservas e ocupação

- `location_occupancy` é projeção derivada do ledger: por endereço, pallets presentes + reservas de entrada (guarda em rota) + reservas de saída (picking alocado).
- O mapa em tempo real pinta: 🟩 livre · 🟦 ocupado · 🟨 reservado (entrada ou saída em andamento) · 🟥 bloqueado/manutenção · 🟪 quarentena.
