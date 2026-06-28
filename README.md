# `Forzy PLN Sprint 2: Busca Semantica em Ativos Industriais`

> Camada de inteligencia linguistica do Digital Twin Forzy. Busca semantica com FAISS + embeddings multilingue (paraphrase-multilingual-MiniLM-L12-v2), MAP=0.954 em 20 consultas de teste. Sprint 2 da disciplina de PLN, FIAP · 2TIAPY · 2026.

---

## `Tecnologias`

![Python](https://img.shields.io/badge/Python-3.10+-blue)
![FAISS](https://img.shields.io/badge/FAISS-IndexFlatIP-orange)
![sentence-transformers](https://img.shields.io/badge/sentence--transformers-384d-green)
![Pandas](https://img.shields.io/badge/pandas-DataFrame-purple)
![Pyvis](https://img.shields.io/badge/Pyvis-grafo%20interativo-teal)
![Colab](https://img.shields.io/badge/Google%20Colab-notebook-F9AB00)
![License](https://img.shields.io/badge/license-MIT-green)

---

## `O que faz`

Implementa a camada de inteligencia linguistica do Digital Twin Forzy para monitoramento preditivo de motores eletricos industriais. Converte 30 ativos em vetores semanticos de 384 dimensoes, permitindo que operadores busquem com linguagem natural ("motor com calor elevado na area de compressores") e o sistema recupere os ativos corretos mesmo quando a ficha tecnica usa terminologia tecnica diferente ("temperatura_elevada", area "COMP").

---

## `Resultados`

| Metrica | Valor | Interpretacao |
|---------|-------|--------------|
| **MAP** | **0.954** | Precisao media global excelente (>0.9 = estado da arte para IR industrial) |
| **mP@1** | **1.000** | O 1 resultado retornado e sempre relevante |
| **mP@3** | **0.933** | 9 em cada 10 resultados nos top-3 sao relevantes |
| **mP@5** | **0.760** | Precisao robusta ampliando para 5 resultados |
| **mR@5** | **0.953** | 95% dos ativos relevantes recuperados nos top-5 |
| **mR@10** | **0.985** | Cobertura quase total nos top-10 |
| **Similaridade cosseno media** | **0.809** | Alta coerencia semantica entre query e documentos |

> 20 consultas cobrindo areas (COMP, EMBA, UTIL, PROD, EXAU, BOMB, LINE), fabricantes (WEG, ABB, SIEMENS), tensoes, status operacionais e tipos de equipamento.

---

## `Arquitetura`

```
Corpus (30 ativos) + Glossario (40 termos PT/EN)
    |
    DataLoader          carrega e valida 9 campos obrigatorios, TAG ISA-5.1
    |
    TextPreprocessor    normaliza unicode + expande sinonimos PT/EN (30+ mapeamentos)
    |
    build_rich_text()   concatena 15+ campos por ativo em string semantica
    |
    EmbeddingGenerator  paraphrase-multilingual-MiniLM-L12-v2, 384d, L2-norm
    |
    FAISSIndexer        IndexFlatIP, similaridade cosseno exata O(n)
    |
    +-- SemanticSearchEngine    query -> sinonimos -> embedding -> top-K
    |
    +-- DescriptionGenerator    9 templates NL por estado operacional
    |
    MetricsEvaluator    P@K, R@K, Average Precision, MAP com ground truth
```

---

## `Classes implementadas (10 classes OOP)`

| Classe | Responsabilidade |
|--------|-----------------|
| `DataLoader` | Carrega e valida corpus: 9 campos obrigatorios, TAG ISA-5.1 |
| `TextPreprocessor` | Normalizacao unicode + expansao de sinonimos PT/EN |
| `EmbeddingGenerator` | Vetorizacao com sentence-transformers, L2-norm |
| `FAISSIndexer` | Indice vetorial IndexFlatIP, busca por similaridade cosseno |
| `SemanticSearchEngine` | Motor de busca: query, sinonimos, embedding, top-K |
| `MetricsEvaluator` | P@K, R@K, Average Precision, MAP com ground truth |
| `VisualizationEngine` | PCA 2D, t-SNE 2D, heatmap cosseno, grafo Pyvis interativo |
| `DescriptionGenerator` | 9 templates NL por estado operacional, regras de composicao |
| `ArchitectureVisualizer` | Diagrama Graphviz, Mermaid, tabela de classes |
| `NotebookUI` | Interface visual padronizada Rich (secoes, KPIs, conclusoes) |

---

## `Templates de Linguagem Natural (9 cenarios)`

| Cenario | Gatilho | Exemplo |
|---------|---------|---------|
| `operando_normal` | Parametros dentro do limite | "Motor USI01-COMP-MT001 operando dentro dos parametros normais. Ultima leitura: 440V, 60.2A, 1800 RPM, 54.2C." |
| `temperatura_elevada` | temperatura_c > 70 | "ALERTA TERMICO: Motor USI01-COMP-MT002 registra 78.4C, acima do limite recomendado." |
| `vibracao_alerta` | 2.3 <= vibracao <= 4.5 mm/s | "VIBRACAO ELEVADA: Motor USI01-UTIL-MT012 com 3.80 mm/s (ISO 10816: Zona B)." |
| `vibracao_alarme` | vibracao > 4.5 mm/s | "ALARME: Motor USI01-COMP-MT003 com 5.80 mm/s (ISO 10816: Zona D, PARADA IMEDIATA)." |
| `sobrecorrente` | Status sobrecorrente | "SOBRECORRENTE: Motor FAB02-PROD-MT015 com 82.0A acima do nominal." |
| `rpm_abaixo` | Status rpm_abaixo | "RPM ABAIXO: Motor FAB02-LINE-MT029 operando a 950 RPM." |
| `desligado` | Status desligado | "Motor FAB02-EXAU-MV022 DESLIGADO. Aguardando autorizacao para religamento." |
| `falha_critica` | Status falha_critica | "FALHA CRITICA: Motor FAB02-BOMB-MB026 FORA DE OPERACAO. Acionar manutencao." |
| `em_manutencao` | Status em_manutencao | "Motor USI01-COMP-MT004 EM MANUTENCAO PROGRAMADA. Aguardar liberacao." |

---

## `Modelo de Embeddings`

**`paraphrase-multilingual-MiniLM-L12-v2`** (sentence-transformers)

| Criterio | Justificativa |
|----------|--------------|
| Multilingue | Corpus Forzy e misto PT/EN: placas tecnicas em EN, fichas em PT, logs de operador em PT coloquial |
| 384 dimensoes | Trade-off otimo custo/qualidade para 30 ativos |
| Parafrase | Treinado em pares: captura sinonimos industriais (motor = equipamento, vibracao = oscilacao) |
| L2-norm | Vetores normalizados, IndexFlatIP = similaridade cosseno exata |

---

## `Normas tecnicas aplicadas`

| Norma | Aplicacao |
|-------|-----------|
| IEC 60034 | Classe de eficiencia (IE1-IE3), classe de isolamento (B, F, H) |
| ISO 10816-3 | Zonas de vibracao A (<2.3), B (2.3-4.5), C (4.5-7.1), D (>7.1 mm/s) |
| ISA-5.1 | TAG format: `{PLANTA}-{AREA}-{TIPO}{SEQ:03d}` (ex: `USI01-COMP-MT001`) |
| NBR 5383 | Terminologia PT para motores de corrente alternada |
| NEMA MG1 | Referencia cruzada EN para eficiencia e protecao IP |

---

## `Estrutura`

```
Sprint-2-NLP-1-SEMESTRE-2026-/
├── sprint2_pln_consulta_final.ipynb   notebook principal executado (56 celulas, 28 outputs)
├── README.md
├── consultas_teste.json               20 consultas com ground truth, metricas e resultados
└── templates_linguagem_natural.md     9 templates documentados com gatilhos e exemplos
```

---

## `Como executar (Google Colab)`

```python
# 1. Upload de sprint2_pln_consulta_final.ipynb no Google Colab
# 2. Executar celula de instalacao (Etapa 0): aprox. 3 min
# 3. Runtime → Run all

# Artefatos gerados em /content/sprint2_data/:
# embeddings.npy        vetores 30x384
# corpus.json           corpus processado
# faiss.index           indice FAISS
# descricoes_nl.json    descricoes NL dos 30 motores
# grafo_motores.html    grafo Pyvis interativo
```

---

## `Integracao com Sprint 1`

| Artefato Sprint 1 | Uso na Sprint 2 |
|------------------|-----------------|
| 30 registros do corpus | `DataLoader.load_corpus()` com todos os campos respeitados |
| 9 campos obrigatorios | `build_rich_text()` concatena todos os 15+ campos para embedding |
| Glossario 40 termos PT/EN | Fonte do dict `SINONIMOS`: 30+ mapeamentos operador-tecnico |
| Pipeline NLP 6 etapas | Integrado ao `TextPreprocessor` |
| Abreviacoes industriais | Dict `ABREVIACOES` expandido antes do embedding |
| TAG format ISA-5.1 | Validado no `DataLoader`, exibido em todos os outputs |

---

## `Licenca`

Distribuido sob a licenca MIT. Veja [LICENSE](LICENSE) para mais informacoes.

---

## `Autor`

**Arthur Baptista dos Santos**
RM 565346 · Inteligencia Artificial · FIAP 2025-2026

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Arthur%20Baptista-0077B5?logo=linkedin)](https://linkedin.com/in/arthur-baptista-dos-santos)
[![GitHub](https://img.shields.io/badge/GitHub-Arthur--Baptista--dos--Santos-181717?logo=github)](https://github.com/Arthur-Baptista-dos-Santos)
