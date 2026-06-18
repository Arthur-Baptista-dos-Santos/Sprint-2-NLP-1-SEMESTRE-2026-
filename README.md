# Forzy Digital Twin - Sprint 2: PLN para Busca Semântica em Ativos Industriais

> **Disciplina:** Processamento de Linguagem Natural, Chatbots & Virtual Agents
> **Instituição:** FIAP · Faculdade de Informática e Administração Paulista · 2TIAPY · 2026
> **Equipe:** Arthur Baptista (RM 565346) · João Pedro (RM 561738) · Nelson Felix (RM 565603)

---

## Visão Executiva

Este projeto implementa a **camada de inteligência linguística** do Digital Twin do Projeto Forzy — um sistema de monitoramento preditivo de motores elétricos industriais. A Sprint 2 expande o corpus estruturado da Sprint 1 com três capacidades novas:

1. **Representação vetorial** — cada ativo industrial é convertido em um vetor semântico de 384 dimensões, capturando equivalências técnicas entre vocabulário de operador e terminologia normativa (IEC 60034, ISO 10816-3, ISA-5.1).

2. **Busca semântica em linguagem natural** — o operador digita *"motor com calor elevado na área de compressores"* e o sistema recupera os ativos relevantes, mesmo que a ficha técnica use *"temperatura_elevada"* e *"COMP"*.

3. **Enriquecimento textual automático** — o Digital Twin substitui leituras numéricas brutas por descrições operacionais em português: *"⚠ ALERTA TÉRMICO — Motor USI01-COMP-MT002 registra 78.4°C, acima do limite recomendado..."*

---

## Resultados da Avaliação (20 Consultas de Teste)

| Métrica | Valor | Interpretação |
|---------|-------|--------------|
| **MAP** | **0.954** | Precisão média global excelente (>0.9 = estado da arte para IR industrial) |
| **mP@1** | **1.000** | O 1º resultado retornado é sempre relevante |
| **mP@3** | **0.933** | 9 em cada 10 resultados nos top-3 são relevantes |
| **mP@5** | **0.760** | Precisão robusta mesmo ampliando para 5 resultados |
| **mR@5** | **0.953** | 95% dos ativos relevantes são recuperados nos top-5 |
| **mR@10** | **0.985** | Cobertura quase total nos top-10 |
| **Similaridade cosseno média** | **0.809** | Alta coerência semântica entre query e documentos |

> 20 consultas cobrindo: áreas (COMP, EMBA, UTIL, PROD, EXAU, BOMB, LINE), fabricantes (WEG, ABB, SIEMENS), tensões (220V, 380V, 440V), status operacionais, parâmetros de sensores e tipos de equipamento.

---

## Arquitetura do Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                   SPRINT 1 (base)                           │
│  Corpus 30 ativos · Glossário 40 termos PT/EN · Pipeline 6  │
│  etapas NLP · ISA-5.1 TAGs · Decisões de padronização       │
└─────────────┬───────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────┐
│    DataLoader       │  Carrega corpus · valida 9 campos obrigatórios · ISA-5.1
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  TextPreprocessor   │  Expande sinônimos (operador ↔ técnico) · normaliza unicode
│  + SINONIMOS dict   │  Abreviações industriais (Sprint 1) · 30+ mapeamentos PT/EN
└──────────┬──────────┘
           │  build_rich_text() — concatena 15+ campos por ativo
           ▼
┌─────────────────────┐
│ EmbeddingGenerator  │  paraphrase-multilingual-MiniLM-L12-v2 · 384 dims · L2-norm
└──────────┬──────────┘  np.float32 [30 × 384]
           │
           ▼
┌─────────────────────┐
│   FAISSIndexer      │  IndexFlatIP · cosine similarity · busca exata O(n)
└──────────┬──────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
┌─────────────┐  ┌──────────────────────┐
│ Semantic    │  │  DescriptionGenerator │
│ Search      │  │  9 templates NL       │
│ Engine      │  │  por estado operat.   │
└──────┬──────┘  └──────────────────────┘
       │
       ▼
┌─────────────────────┐
│  MetricsEvaluator   │  P@K · R@K · Average Precision · MAP · 20 queries
└─────────────────────┘
```

---

## Classes Implementadas (10 classes OOP)

| Classe | Responsabilidade | Sprint 1 |
|--------|-----------------|----------|
| `DataLoader` | Carrega e valida o corpus — 9 campos obrigatórios, TAG ISA-5.1 | Base |
| `TextPreprocessor` | Normalização unicode + expansão de sinônimos PT/EN | Estende pipeline |
| `EmbeddingGenerator` | Vetorização com sentence-transformers, L2-norm | Novo |
| `FAISSIndexer` | Índice vetorial IndexFlatIP, busca por similaridade cosseno | Novo |
| `SemanticSearchEngine` | Motor de busca: query → sinônimos → embedding → top-K | Novo |
| `MetricsEvaluator` | P@K, R@K, Average Precision, MAP com ground truth | Novo |
| `VisualizationEngine` | PCA 2D, t-SNE 2D, heatmap cosseno, grafo Pyvis interativo | Novo |
| `DescriptionGenerator` | 9 templates NL por estado operacional, regras de composição | Novo |
| `ArchitectureVisualizer` | Diagrama Graphviz, Mermaid, tabela de classes | Novo |
| `NotebookUI` | Interface visual padronizada Rich (seções, KPIs, conclusões) | Novo |

---

## Templates de Linguagem Natural (9 Cenários)

O `DescriptionGenerator` seleciona automaticamente o template correto com base nos parâmetros do sensor:

| Cenário | Gatilho | Exemplo de saída |
|---------|---------|-----------------|
| `operando_normal` | Parâmetros dentro do limite | *"Motor USI01-COMP-MT001 operando dentro dos parâmetros normais. Última leitura: 440V, 60.2A, 1800 RPM, 54.2°C."* |
| `temperatura_elevada` | `temperatura_c > 70°C` | *"⚠  ALERTA TÉRMICO — Motor USI01-COMP-MT002 registra 78.4°C, acima do limite recomendado."* |
| `vibracao_alerta` | `2.3 ≤ vibracao ≤ 4.5 mm/s` | *"⚠  VIBRAÇÃO ELEVADA — Motor USI01-UTIL-MT012 com 3.80 mm/s (ISO 10816: Zona B)."* |
| `vibracao_alarme` | `vibracao > 4.5 mm/s` | *"⚡︎ ALARME — Motor USI01-COMP-MT003 com 5.80 mm/s (ISO 10816: Zona D — PARADA IMEDIATA)."* |
| `sobrecorrente` | Status `sobrecorrente` | *"⚠  SOBRECORRENTE — Motor FAB02-PROD-MT015 com 82.0A acima do nominal."* |
| `rpm_abaixo` | Status `rpm_abaixo` | *"⚠  RPM ABAIXO — Motor FAB02-LINE-MT029 operando a 950 RPM."* |
| `desligado` | Status `desligado` | *"■ Motor FAB02-EXAU-MV022 DESLIGADO. Aguardando autorização para religamento."* |
| `falha_critica` | Status `falha_critica` | *"⚡︎ FALHA CRÍTICA — Motor FAB02-BOMB-MB026 FORA DE OPERAÇÃO. Acionar manutenção."* |
| `em_manutencao` | Status `em_manutencao` | *"🛠︎ Motor USI01-COMP-MT004 EM MANUTENÇÃO PROGRAMADA. Aguardar liberação."* |

---

## Modelo de Embeddings — Justificativa Técnica

**`paraphrase-multilingual-MiniLM-L12-v2`** (HuggingFace sentence-transformers)

| Critério de escolha | Justificativa |
|--------------------|--------------|
| **Multilíngue** | Corpus Forzy é misto PT/EN — placas técnicas em EN, fichas cadastro em PT, logs de operador em PT coloquial |
| **384 dimensões** | Suficiente para 30 ativos; excelente trade-off custo/qualidade |
| **Paráfrases** | Treinado em pares de paráfrases — captura sinônimos industriais (motor ↔ equipamento, vibração ↔ oscilação) |
| **L2-norm** | Vetores normalizados → IndexFlatIP = similaridade cosseno exata |

Alternativas avaliadas e descartadas:

| Modelo | Motivo da exclusão |
|--------|-------------------|
| `all-MiniLM-L6-v2` | Somente inglês — descarta terminologia PT do operador |
| `all-mpnet-base-v2` | 768d em inglês — superdimensionado e sem PT |
| `BGE-M3` | 1024d — custo computacional injustificado para 30 ativos |

---

## Integração com Sprint 1

| Artefato Sprint 1 | Como é usado na Sprint 2 |
|------------------|--------------------------|
| **30 registros do corpus** | `DataLoader.load_corpus()` — todos os campos respeitados |
| **9 campos obrigatórios** (tag, descricao_curta/longa, fabricante, modelo, localizacao, area, planta, status) | `build_rich_text()` concatena todos os 15+ campos para embedding |
| **Glossário 40 termos PT/EN** | Fonte do `SINONIMOS` dict — 30+ mapeamentos operador↔técnico |
| **Pipeline NLP 6 etapas** | Integrado ao `TextPreprocessor` (normalização, abreviações, unicode) |
| **Abreviações industriais** | `ABREVIACOES` dict expandido antes do embedding |
| **TAG format ISA-5.1** | Validado no `DataLoader`, exibido em todos os outputs |
| **Decisões de padronização** | Campos obrigatórios e vocabulário controlado de fabricantes respeitados |

---

## Normas Técnicas Aplicadas

| Norma | Aplicação |
|-------|-----------|
| **IEC 60034** | Classe de eficiência (IE1–IE3), classe de isolamento (B, F, H) |
| **ISO 10816-3** | Zonas de vibração A (<2.3), B (2.3–4.5), C (4.5–7.1), D (>7.1 mm/s) |
| **ISA-5.1** | TAG format: `{PLANTA}-{AREA}-{TIPO}{SEQ:03d}` (ex: `USI01-COMP-MT001`) |
| **NBR 5383** | Terminologia PT para motores de corrente alternada |
| **NEMA MG1** | Referência cruzada EN para eficiência e proteção IP |

---

## Entregas do Sprint 2

```
Sprint2_PLN/
├── sprint2_pln_consulta_final.ipynb   ← Notebook principal EXECUTADO (Colab, 56 células, 28 outputs)
├── README.md                           ← Este documento
├── requirements.txt                    ← 21 dependências com versões
├── avaliacao/
│   └── consultas_teste.json           ← 20 consultas com ground truth, métricas e resultados
└── docs/
    └── templates_linguagem_natural.md ← 9 templates documentados com gatilhos, exemplos e normas
```

---

## Como Executar no Google Colab

```
1. Fazer upload de sprint2_pln_consulta_final.ipynb no Google Colab
2. Executar célula de instalação (Etapa 0) — ~3 min
3. Runtime → Run all
4. Artefatos gerados em /content/sprint2_data/:
   ├── embeddings.npy        (vetores 30×384)
   ├── corpus.json           (corpus processado)
   ├── faiss.index           (índice FAISS)
   ├── descricoes_nl.json    (descrições NL dos 30 motores)
   ├── arquitetura_pipeline.png
   └── grafo_motores.html    (grafo Pyvis interativo)
```

---

## Autoavaliação — Rubrica FIAP

| Critério | Peso | Entregável | Nota estimada |
|---------|------|-----------|--------------|
| Escolha e aplicação do modelo de embeddings | 25% | `paraphrase-multilingual-MiniLM-L12-v2` · 384d · L2-norm · comparação vs 4 modelos | ✔︎
| Qualidade da busca semântica e métricas | 35% | MAP=0.954 · mP@1=1.000 · 20 consultas · P@K/R@K · sinônimos | ✔︎
| Templates de linguagem natural | 20% | 9 templates · `DescriptionGenerator` · regras composição · todos os estados | ✔︎
| Integração com Sprint 1 | 20% | 30 registros · glossário 40 termos · pipeline 6 etapas · ISA-5.1 | ✔︎
