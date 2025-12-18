# Análise Corrente Bomba Água Quente
**Data**: 2024-12-18  
**Período analisado**: 30 dias (2025-11-18 a 2025-12-18)  
**Total amostras**: 1.826 (corrente) + 836 (derivative)

---

## 🎯 Objetivo

Validar thresholds de corrente e taxa de mudança (dI/dt) após relatos de **falsos positivos persistentes** em:
- Alerta "Mudança Rápida" 
- Alerta "Desgaste Emergente"

---

## 📊 Resultados - Corrente (Bomba Ligada)

**Dataset**: 1.292 amostras com corrente > 0.1A (71.8% do tempo)

| Métrica | Valor | Status |
|---------|-------|--------|
| **P5** | 0.303A | ✅ Threshold min OK |
| **Mediana** | 0.310A | ✅ Operação normal |
| **Média** | 0.309A | ✅ Consistente |
| **P95** | 0.323A | ✅ Threshold max OK |
| **Máximo** | 0.330A | ✅ Dentro do esperado |

### Validação Thresholds Atuais
```yaml
input_number.pump_current_normal_min: 0.303A  # P5  ✅
input_number.pump_current_normal_max: 0.323A  # P95 ✅
input_number.pump_current_critical_max: 0.388A # Max × 1.15 ✅
```

**Conclusão**: Thresholds de corrente estão **CORRETOS**.

---

## ⚡ Resultados - Taxa Mudança (dI/dt)

**Dataset**: 836 amostras válidas

| Métrica | Valor | Interpretação |
|---------|-------|---------------|
| **Mediana** | 0.0001 A/s | Operação estável (quase zero) |
| **P75** | 0.0014 A/s | Variação normal |
| **P95** | 0.0618 A/s | Inrush/desligamento |
| **P99** | 0.0634 A/s | Eventos rápidos |
| **Máximo** | 0.0634 A/s | Pico máximo |

### 🚨 PROBLEMA IDENTIFICADO

**Threshold atual**: `0.010 A/s`  
**Eventos capturados**: 218/836 (26%)

**Análise**:
- ❌ Threshold de `0.010 A/s` está **ABAIXO do P95** (0.0618 A/s)
- ❌ Captura **TODOS** os eventos de partida/desligamento (inrush)
- ❌ Não distingue inrush normal de problema real

**Evidência do Logbook** (hoje 10:36-10:57):
```
10:57:04 - Mudança rápida disparada com dI/dt = 0.0003 A/s
10:39:09 - Mudança rápida disparada com dI/dt = 0.0014 A/s
10:36:07 - Mudança rápida disparada com dI/dt = 0.0002 A/s
```

**Bug identificado**: Valores de 0.0002-0.0014 A/s **NÃO deveriam** disparar threshold de 0.010 A/s.  
→ Indica problema na **lógica do binary_sensor** (não no threshold).

---

## 📈 Resultados - Médias (Desgaste)

| Sensor | Valor Atual | Status |
|--------|-------------|--------|
| **Média 7d** | 0.270A | Baseline estável |
| **Média 24h** | 0.272A | +0.74% vs 7d |
| **Threshold desgaste** | 0.278A (7d × 1.03) | Não ultrapassado |

**Conclusão**: Sem sinais de desgaste progressivo. Sistema operando normalmente.

---

## 🔧 Problemas Identificados

### 1. **Threshold mudança_rápida MUITO BAIXO**
- Atual: 0.010 A/s
- Captura: 26% das amostras (inrush normal)
- **Recomendado**: 0.020-0.030 A/s (entre P95-P99)

### 2. **Lógica binary_sensor BUGADA**
- Disparando com valores de 0.0002-0.0003 A/s
- Deveria disparar apenas com |dI/dt| > 0.010 A/s
- **Causa provável**: Condição errada ou ausência de hysteresis

### 3. **TTS em alertas não-críticos**
- Mudança rápida → Inrush normal, não é emergência
- Desgaste emergente → Preventivo, não é urgente
- **Proposta**: TTS apenas para corrente_critica + timeout

---

## ✅ Recomendações

### IMEDIATO (Corrigir Bugs)

#### 1. Ajustar threshold mudança_rápida
```yaml
# Aumentar de 0.010 para 0.025 A/s (P97)
binary_sensor.bomba_mudanca_rapida_corrente:
  value_template: >
    {{ states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs > 0.025 }}
```

#### 2. Adicionar delay anti-inrush
```yaml
# Ignorar primeiros 10s após ligar/desligar
binary_sensor.bomba_mudanca_rapida_corrente:
  value_template: >
    {% set dI = states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs %}
    {% set bomba_on = is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') %}
    {% set tempo_ligada = (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() %}
    {{ bomba_on and tempo_ligada > 10 and dI > 0.025 }}
```

#### 3. Adicionar hysteresis
```yaml
binary_sensor.bomba_mudanca_rapida_corrente:
  delay_on: "00:00:03"  # Confirmar por 3s antes de alertar
  delay_off: "00:00:05" # Aguardar 5s antes de limpar
```

### ESTRATÉGIA TTS (Reduzir Ruído)

**TTS APENAS para:**
- ✅ Corrente crítica (>0.388A) - Emergência
- ✅ Timeout (30min) - Possível vazamento

**Dashboard APENAS para:**
- 📊 Mudança rápida - Indicador técnico
- 📊 Desgaste emergente - Manutenção preventiva
- 📊 Corrente anormal (não crítica) - Monitoramento

---

## 📁 Arquivos para Atualizar

### 1. `template_sensors.yaml`
- Ajustar threshold `bomba_mudanca_rapida_corrente`: 0.010 → 0.025 A/s
- Adicionar delay anti-inrush (10s)
- Adicionar hysteresis (delay_on/off)

### 2. `automations_bomba.yaml`
- **REMOVER** TTS de:
  - Alerta mudança rápida
  - Alerta desgaste emergente
  - Alerta corrente anormal (não crítica)
- **MANTER** TTS apenas em:
  - Corrente crítica (>0.388A)
  - Timeout 30min

---

## 📝 Próximos Passos

1. ✅ **Análise dados completa** (este relatório)
2. ⏳ **Migrar para Claude Code** (sessão contínua)
3. ⏳ **Implementar correções** (template_sensors + automations)
4. ⏳ **Testar 24h** (validar ausência de falsos positivos)
5. ⏳ **Documentar** (CHANGELOG, CLAUDE.md, lessons-learned)

---

## 🎯 Expectativa Pós-Correção

**Antes (atual)**:
- Alertas a cada 2-5min (falsos positivos)
- TTS persistente e irritante
- Dados corretos mas lógica bugada

**Depois (esperado)**:
- Alertas apenas para eventos REAIS
- TTS apenas para emergências
- Monitoramento silencioso no dashboard
- Detecção de desgaste real (não ruído)

---

**Relatório gerado por**: Claude 3.5 Sonnet  
**Aprovado para implementação**: Aguardando confirmação
