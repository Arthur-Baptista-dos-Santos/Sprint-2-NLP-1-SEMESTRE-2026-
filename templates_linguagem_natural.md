# Documento de Templates de Linguagem Natural
## Projeto Forzy — Digital Twin para Manutenção Preditiva de Motores Elétricos
### Sprint 2 — Enriquecimento Textual da Interface de Visualização

**Disciplina:** Processamento de Linguagem Natural, Chatbots & Virtual Agents  
**Instituição:** FIAP · 2TIAPY · 2026  
**Equipe:** Arthur Baptista (RM 565346) · João Pedro (RM 561738) · Nelson Felix (RM 565603)

---

## 1. Objetivo

Substituir a exibição de dados brutos dos sensores por linguagem natural compreensível
ao operador industrial sem formação técnica aprofundada.  
Exemplo de transformação:

| Dados brutos | Linguagem natural |
|---|---|
| `status: temperatura_elevada, temp_c: 78.4, corrente_a: 88.0` | `⚠️ ALERTA TÉRMICO — Motor USI01-COMP-MT002 registra temperatura de 78.4°C, acima do limite recomendado.` |

---

## 2. Regras de Composição Textual

### 2.1 Hierarquia de prioridade de template

Quando múltiplas condições são verdadeiras simultaneamente, aplica-se a seguinte
ordem de prioridade (maior prioridade primeiro):

```
1. falha_critica       → temperatura > 85°C  OU  vibração > 5.5 mm/s  OU  status = "falha_critica"
2. vibracao_alarme     → vibração > 4.5 mm/s  OU  status = "vibracao_alarme"
3. desligado           → status = "desligado"  (rpm = 0 e corrente = 0)
4. em_manutencao       → status = "em_manutencao"
5. sobrecorrente       → status = "sobrecorrente"
6. temperatura_elevada → temperatura > 70°C  OU  status = "temperatura_elevada"
7. vibracao_alerta     → 2.3 ≤ vibração ≤ 4.5 mm/s  OU  status = "vibracao_alerta"
8. rpm_abaixo          → status = "rpm_abaixo"
9. inspecao            → status = "inspecao"
10. operando_normal    → fallback (nenhuma condição anterior atendida)
```

### 2.2 Campos obrigatórios em cada template

| Campo | Fonte | Tipo |
|-------|-------|------|
| `tag` | Cadastro Sprint 1 | string (ISA-5.1) |
| `tensao_v` | especificacao_tecnica | float (V) |
| `corrente_a` | leitura_atual | float (A) |
| `rpm` | leitura_atual | int |
| `temperatura_c` | leitura_atual | float (°C) |
| `vibracao_mm_s` | leitura_atual | float (mm/s) |
| `localizacao` | Cadastro Sprint 1 | string |
| `status` | sistema monitoramento | string (enum) |

### 2.3 Formatação numérica

- Temperaturas: 1 casa decimal + símbolo `°C` → `54.2°C`
- Vibração: 2 casas decimais + unidade `mm/s` → `1.24 mm/s`
- Corrente: 1 casa decimal + unidade `A` → `60.2A`
- RPM: inteiro sem decimais → `1800 RPM`
- Tensão: inteiro + unidade `V` → `380V`

### 2.4 Ícones de estado

| Estado | Ícone | Nível |
|--------|-------|-------|
| Normal | (sem ícone) | Info |
| Alerta | ⚠️ | Warning |
| Alarme / Falha | 🚨 | Critical |
| Desligado | ⬛ | Info |
| Manutenção | 🔧 | Info |
| Inspeção | 🔍 | Info |

---

## 3. Templates por Situação Operacional

### 3.1 Operação Normal

**Gatilho:** Nenhuma condição anormal detectada  
**Variáveis:** `tag`, `tensao_v`, `corrente_a`, `rpm`, `temperatura_c`, `vibracao_mm_s`

```
Motor {tag} operando dentro dos parâmetros normais.
Última leitura: {tensao_v}V, {corrente_a}A, {rpm} RPM e temperatura de {temperatura_c:.1f}°C.
Vibração: {vibracao_mm_s:.2f} mm/s. Status: [OK]
```

**Exemplo de saída:**
> Motor USI01-COMP-MT001 operando dentro dos parâmetros normais. Última leitura: 440V, 60.2A, 1800 RPM e temperatura de 54.2°C. Vibração: 1.20 mm/s. Status: [OK]

---

### 3.2 Temperatura Elevada

**Gatilho:** `temperatura_c > 70°C` ou `status = "temperatura_elevada"`  
**Classificação:** Alerta — não exige parada imediata, mas requer monitoramento intensificado  
**Norma:** IEC 60034-1 — temperatura máxima por classe de isolamento (Classe F: 155°C, Classe B: 130°C)

```
⚠️ ALERTA TÉRMICO — Motor {tag} registra temperatura de {temperatura_c:.1f}°C,
acima do limite recomendado. Tensão: {tensao_v}V | Corrente: {corrente_a}A |
Vibração: {vibracao_mm_s:.2f} mm/s. Verificar ventilação e carga mecânica.
Localização: {localizacao}.
```

**Exemplo de saída:**
> ⚠️ ALERTA TÉRMICO — Motor USI01-COMP-MT002 registra temperatura de 78.4°C, acima do limite recomendado. Tensão: 440V | Corrente: 88.0A | Vibração: 1.50 mm/s. Verificar ventilação e carga mecânica. Localização: USI01.COMP.L1-P2.

---

### 3.3 Vibração Anormal — Alerta (Zona B/C ISO 10816)

**Gatilho:** `2.3 ≤ vibracao_mm_s ≤ 4.5` ou `status = "vibracao_alerta"`  
**Norma:** ISO 10816-3 — Zona B (2.3–4.5 mm/s): operação com restrição, planejar manutenção

```
⚠️ VIBRAÇÃO ELEVADA — Motor {tag} com vibração de {vibracao_mm_s:.2f} mm/s
(ISO 10816: Zona B — atenção).
Temperatura: {temperatura_c:.1f}°C | Corrente: {corrente_a}A.
Verificar desalinhamento e integridade dos rolamentos. Localização: {localizacao}.
```

**Exemplo de saída:**
> ⚠️ VIBRAÇÃO ELEVADA — Motor USI01-COMP-MT003 com vibração de 3.80 mm/s (ISO 10816: Zona B — atenção). Temperatura: 62.1°C | Corrente: 117.0A. Verificar desalinhamento e integridade dos rolamentos. Localização: USI01.COMP.L2-P1.

---

### 3.4 Vibração Anormal — Alarme (Zona D ISO 10816)

**Gatilho:** `vibracao_mm_s > 4.5` ou `status = "vibracao_alarme"`  
**Norma:** ISO 10816-3 — Zona D (> 7.1 mm/s): parada imediata; Zona C (4.5–7.1 mm/s): alerta severo  
**Ação:** Parada imediata recomendada

```
🚨 ALARME DE VIBRAÇÃO — Motor {tag} com vibração de {vibracao_mm_s:.2f} mm/s
(ISO 10816: Zona D — PARADA IMEDIATA RECOMENDADA).
Temperatura: {temperatura_c:.1f}°C. Acionar equipe de manutenção.
```

**Exemplo de saída:**
> 🚨 ALARME DE VIBRAÇÃO — Motor USI01-UTIL-MT012 com vibração de 5.80 mm/s (ISO 10816: Zona D — PARADA IMEDIATA RECOMENDADA). Temperatura: 65.2°C. Acionar equipe de manutenção.

---

### 3.5 Sobrecarga / Sobrecorrente

**Gatilho:** `corrente_a > corrente_nominal * 1.15` ou `status = "sobrecorrente"`  
**Impacto:** Sobrecarga mecânica, deterioração acelerada do isolamento, risco de trip térmico

```
⚠️ SOBRECORRENTE — Motor {tag} com corrente de {corrente_a}A acima do nominal.
Tensão: {tensao_v}V | Temperatura: {temperatura_c:.1f}°C.
Verificar sobrecarga mecânica e acionamento. Localização: {localizacao}.
```

**Exemplo de saída:**
> ⚠️ SOBRECORRENTE — Motor FAB02-PROD-MT015 com corrente de 82.0A acima do nominal. Tensão: 380V | Temperatura: 68.5°C. Verificar sobrecarga mecânica e acionamento. Localização: FAB02.PROD.LA-P2.

---

### 3.6 RPM Abaixo do Esperado

**Gatilho:** `rpm < rpm_nominal * 0.90` ou `status = "rpm_abaixo"`  
**Causas possíveis:** Escorregamento excessivo (motores de indução), sobrecarga, falha no acionamento (VFD)

```
⚠️ RPM ABAIXO DO ESPERADO — Motor {tag} operando a {rpm} RPM.
Verificar escorregamento, sobrecarga ou falha no acionamento.
Temperatura: {temperatura_c:.1f}°C | Vibração: {vibracao_mm_s:.2f} mm/s.
```

**Exemplo de saída:**
> ⚠️ RPM ABAIXO DO ESPERADO — Motor FAB02-LINE-MT029 operando a 950 RPM. Verificar escorregamento, sobrecarga ou falha no acionamento. Temperatura: 57.2°C | Vibração: 1.80 mm/s.

---

### 3.7 Motor Desligado

**Gatilho:** `rpm = 0 AND corrente_a = 0` ou `status = "desligado"`  
**Contexto:** Desligamento programado, aguardando religamento autorizado

```
⬛ Motor {tag} DESLIGADO. Última leitura: temperatura {temperatura_c:.1f}°C.
Localização: {localizacao}. Aguardando autorização para religamento.
```

**Exemplo de saída:**
> ⬛ Motor FAB02-EXAU-MV022 DESLIGADO. Última leitura: temperatura 23.5°C. Localização: FAB02.EXAU.L2-P2. Aguardando autorização para religamento.

---

### 3.8 Falha Crítica

**Gatilho:** `status = "falha_critica"` ou `(temperatura_c > 85°C AND vibracao_mm_s > 5.0)`  
**Ação:** Parada de emergência — acionar equipe de manutenção imediatamente

```
🚨 FALHA CRÍTICA — Motor {tag} FORA DE OPERAÇÃO.
Parâmetros no momento da falha: {temperatura_c:.1f}°C, {corrente_a}A, {vibracao_mm_s:.2f} mm/s.
ACIONAR EQUIPE DE MANUTENÇÃO IMEDIATAMENTE.
Localização: {localizacao}.
```

**Exemplo de saída:**
> 🚨 FALHA CRÍTICA — Motor FAB02-BOMB-MB026 FORA DE OPERAÇÃO. Parâmetros no momento da falha: 92.3°C, 0.0A, 6.90 mm/s. ACIONAR EQUIPE DE MANUTENÇÃO IMEDIATAMENTE. Localização: FAB02.BOMB.L2-P2.

---

### 3.9 Manutenção Preventiva Programada

**Gatilho:** `status = "em_manutencao"`  
**Contexto:** Intervenção planejada — rolamentos, vedações, inspeção elétrica

```
🔧 Motor {tag} EM MANUTENÇÃO PROGRAMADA.
Localização: {localizacao}. Equipe de manutenção responsável.
Aguardar liberação antes de religar.
```

**Exemplo de saída:**
> 🔧 Motor USI01-COMP-MT004 EM MANUTENÇÃO PROGRAMADA. Localização: USI01.COMP.L2-P2. Equipe de manutenção responsável. Aguardar liberação antes de religar.

---

### 3.10 Inspeção de Rotina

**Gatilho:** `status = "inspecao"`  
**Contexto:** Laudo técnico em andamento, dados disponíveis mas resultado pendente

```
🔍 Motor {tag} EM INSPEÇÃO.
Vibração: {vibracao_mm_s:.2f} mm/s | Temperatura: {temperatura_c:.1f}°C | Corrente: {corrente_a}A.
Aguardando laudo técnico.
```

---

## 4. Tabela Resumo dos Templates

| # | Template | Gatilho primário | Ícone | Nível |
|---|----------|-----------------|-------|-------|
| 1 | `operando_normal` | Sem anomalias | — | Info |
| 2 | `temperatura_elevada` | temp > 70°C | ⚠️ | Warning |
| 3 | `vibracao_alerta` | vib 2.3–4.5 mm/s | ⚠️ | Warning |
| 4 | `vibracao_alarme` | vib > 4.5 mm/s | 🚨 | Critical |
| 5 | `sobrecorrente` | corrente > 115% nominal | ⚠️ | Warning |
| 6 | `rpm_abaixo` | rpm < 90% nominal | ⚠️ | Warning |
| 7 | `desligado` | rpm=0, corrente=0 | ⬛ | Info |
| 8 | `falha_critica` | Parada emergência | 🚨 | Critical |
| 9 | `em_manutencao` | Manutenção programada | 🔧 | Info |
| 10 | `inspecao` | Laudo em andamento | 🔍 | Info |

---

## 5. Saídas Geradas para os 30 Motores do Corpus

| TAG | Template | Descrição Gerada (resumo) |
|-----|----------|--------------------------|
| USI01-COMP-MT001 | operando_normal | Motor USI01-COMP-MT001 operando dentro dos parâmetros normais. Última leitura: 440V, 60.2A, 1800 RPM e temperatura de 54.2°C. |
| USI01-COMP-MT002 | temperatura_elevada | ⚠️ ALERTA TÉRMICO — Motor USI01-COMP-MT002 registra temperatura de 78.4°C... |
| USI01-COMP-MT003 | vibracao_alerta | ⚠️ VIBRAÇÃO ELEVADA — Motor USI01-COMP-MT003 com vibração de 3.80 mm/s (ISO 10816: Zona B)... |
| USI01-COMP-MT004 | em_manutencao | 🔧 Motor USI01-COMP-MT004 EM MANUTENÇÃO PROGRAMADA... |
| USI01-COMP-MT005 | operando_normal | Motor USI01-COMP-MT005 operando dentro dos parâmetros normais... |
| USI01-EMBA-MT006 | operando_normal | Motor USI01-EMBA-MT006 operando dentro dos parâmetros normais... |
| USI01-EMBA-MT007 | operando_normal | Motor USI01-EMBA-MT007 operando dentro dos parâmetros normais... |
| USI01-EMBA-MT008 | temperatura_elevada | ⚠️ ALERTA TÉRMICO — Motor USI01-EMBA-MT008 registra temperatura de 72.8°C... |
| USI01-EMBA-MT009 | em_manutencao | 🔧 Motor USI01-EMBA-MT009 EM MANUTENÇÃO PROGRAMADA... |
| USI01-EMBA-MT010 | operando_normal | Motor USI01-EMBA-MT010 operando dentro dos parâmetros normais... |
| USI01-UTIL-MT011 | operando_normal | Motor USI01-UTIL-MT011 operando dentro dos parâmetros normais... |
| USI01-UTIL-MT012 | vibracao_alarme | 🚨 ALARME DE VIBRAÇÃO — Motor USI01-UTIL-MT012 com vibração de 5.80 mm/s (ISO 10816: Zona D)... |
| USI01-UTIL-MT013 | operando_normal | Motor USI01-UTIL-MT013 operando dentro dos parâmetros normais... |
| FAB02-PROD-MT014 | operando_normal | Motor FAB02-PROD-MT014 operando dentro dos parâmetros normais... |
| FAB02-PROD-MT015 | sobrecorrente | ⚠️ SOBRECORRENTE — Motor FAB02-PROD-MT015 com corrente de 82.0A acima do nominal... |
| FAB02-PROD-MT016 | operando_normal | Motor FAB02-PROD-MT016 operando dentro dos parâmetros normais... |
| FAB02-PROD-MT017 | em_manutencao | 🔧 Motor FAB02-PROD-MT017 EM MANUTENÇÃO PROGRAMADA... |
| FAB02-PROD-MT018 | operando_normal | Motor FAB02-PROD-MT018 operando dentro dos parâmetros normais... |
| FAB02-EXAU-MV019 | operando_normal | Motor FAB02-EXAU-MV019 operando dentro dos parâmetros normais... |
| FAB02-EXAU-MV020 | operando_normal | Motor FAB02-EXAU-MV020 operando dentro dos parâmetros normais... |
| FAB02-EXAU-MV021 | temperatura_elevada | ⚠️ ALERTA TÉRMICO — Motor FAB02-EXAU-MV021 registra temperatura de 71.3°C... |
| FAB02-EXAU-MV022 | desligado | ⬛ Motor FAB02-EXAU-MV022 DESLIGADO... |
| FAB02-BOMB-MB023 | operando_normal | Motor FAB02-BOMB-MB023 operando dentro dos parâmetros normais... |
| FAB02-BOMB-MB024 | vibracao_alerta | ⚠️ VIBRAÇÃO ELEVADA — Motor FAB02-BOMB-MB024 com vibração de 3.20 mm/s (ISO 10816: Zona B)... |
| FAB02-BOMB-MB025 | em_manutencao | 🔧 Motor FAB02-BOMB-MB025 EM MANUTENÇÃO PROGRAMADA... |
| FAB02-BOMB-MB026 | falha_critica | 🚨 FALHA CRÍTICA — Motor FAB02-BOMB-MB026 FORA DE OPERAÇÃO... |
| FAB02-LINE-MT027 | operando_normal | Motor FAB02-LINE-MT027 operando dentro dos parâmetros normais... |
| FAB02-LINE-MT028 | operando_normal | Motor FAB02-LINE-MT028 operando dentro dos parâmetros normais... |
| FAB02-LINE-MT029 | rpm_abaixo | ⚠️ RPM ABAIXO DO ESPERADO — Motor FAB02-LINE-MT029 operando a 950 RPM... |
| FAB02-LINE-MT030 | operando_normal | Motor FAB02-LINE-MT030 operando dentro dos parâmetros normais... |

---

## 6. Referências

- **IEC 60034-1** — Máquinas Elétricas Girantes: temperaturas nominais e limites
- **ISO 10816-3** — Avaliação de vibração mecânica em máquinas (zonas A/B/C/D)
- **NBR 5383** — Terminologia PT para máquinas de corrente alternada
- **ISA-5.1** — Instrumentation Symbols and Identification (padrão TAGs)
- **Sprint 1** — `decisoes_padronizacao.md` v1.0.0 — campos obrigatórios e pipeline NLP
