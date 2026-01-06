# Mudanças Propostas - Bomba v3.5.2
**Data**: 2024-12-18  
**Objetivo**: Eliminar falsos positivos e ajustar estratégia de notificações

---

## 🎯 Resumo das Mudanças

| Mudança | Arquivo | Tipo |
|---------|---------|------|
| Threshold mudança_rápida: 0.010 → 0.025 A/s | template_sensors.yaml | CRÍTICO |
| Adicionar delay anti-inrush (10s) | template_sensors.yaml | CRÍTICO |
| Adicionar hysteresis mudança_rápida | template_sensors.yaml | IMPORTANTE |
| Adicionar delay desgaste_emergente (30min/1h) | template_sensors.yaml | IMPORTANTE |
| Adicionar condições desgaste (bomba_on + estabilizada) | template_sensors.yaml | IMPORTANTE |
| Remover TTS de 3 alertas não-críticos | automations_bomba.yaml | IMPORTANTE |

---

## 📝 Mudança 1: Template Sensors

### Arquivo: `template_sensors.yaml`

#### 1.1 Sensor: `binary_sensor.bomba_mudanca_rapida_corrente`

**ANTES** (v3.5.1):
```yaml
- binary_sensor:
  - name: "Bomba Mudança Rápida Corrente"
    unique_id: pump_rapid_current_change_v3
    state: >
      {{ states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs > 0.010 }}
    attributes:
      rate_change: "{{ states('sensor.bomba_taxa_mudanca_corrente')|float(0) }}"
      threshold: 0.010
      unit: "A/s"
```

**DEPOIS** (v3.5.2):
```yaml
- binary_sensor:
  - name: "Bomba Mudança Rápida Corrente"
    unique_id: pump_rapid_current_change_v3
    state: >
      {% set dI = states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs %}
      {% set bomba_on = is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') %}
      {% set tempo_ligada = (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() %}
      {% set threshold = 0.025 %}
      {{ bomba_on and tempo_ligada > 10 and dI > threshold }}
    delay_on: "00:00:03"
    delay_off: "00:00:05"
    attributes:
      rate_change: "{{ states('sensor.bomba_taxa_mudanca_corrente')|float(0) }}"
      threshold: 0.025
      unit: "A/s"
      time_since_activation: "{{ (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() }}"
      inrush_filter_active: "{{ (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() <= 10 }}"
```

**Mudanças**:
1. Threshold: 0.010 → 0.025 A/s (P97, entre P95-P99)
2. Filtro anti-inrush: Ignora primeiros 10s após ligação
3. Hysteresis: delay_on=3s, delay_off=5s
4. Attributes extras: tempo desde ativação, status filtro inrush

**Justificativa**:
- 0.025 A/s captura apenas P97+ (eventos reais, não inrush normal)
- Delay de 10s permite inrush completar (~2-3s) + margem de segurança
- Hysteresis evita alternância rápida (detectado/limpo)

---

#### 1.2 Sensor: `binary_sensor.bomba_desgaste_emergente`

**ANTES** (v3.5.1):
```yaml
- binary_sensor:
  - name: "Bomba Desgaste Emergente"
    unique_id: pump_emerging_wear_v3
    state: >
      {% set media_7d = states('sensor.bomba_corrente_media_7d')|float(0) %}
      {% set media_30d = states('sensor.bomba_corrente_media_30d')|float(0) %}
      {{ media_7d > media_30d * 1.03 }}
    attributes:
      avg_7d: "{{ states('sensor.bomba_corrente_media_7d')|float(0) }}"
      avg_30d: "{{ states('sensor.bomba_corrente_media_30d')|float(0) }}"
      threshold: "{{ states('sensor.bomba_corrente_media_30d')|float(0) * 1.03 }}"
      threshold_percent: 3
```

**DEPOIS** (v3.5.2):
```yaml
- binary_sensor:
  - name: "Bomba Desgaste Emergente"
    unique_id: pump_emerging_wear_v3
    state: >
      {% set media_7d = states('sensor.bomba_corrente_media_7d')|float(0) %}
      {% set media_30d = states('sensor.bomba_corrente_media_30d')|float(0) %}
      {% set bomba_on = is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') %}
      {% set estabilizada = is_state('binary_sensor.bomba_corrente_estabilizada', 'on') %}
      {% set threshold_percent = 1.03 %}
      {{ bomba_on and estabilizada and media_7d > media_30d * threshold_percent }}
    delay_on: "00:30:00"
    delay_off: "01:00:00"
    attributes:
      avg_7d: "{{ states('sensor.bomba_corrente_media_7d')|float(0) }}"
      avg_30d: "{{ states('sensor.bomba_corrente_media_30d')|float(0) }}"
      threshold: "{{ states('sensor.bomba_corrente_media_30d')|float(0) * 1.03 }}"
      threshold_percent: 3
      pump_running: "{{ is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') }}"
      current_stable: "{{ is_state('binary_sensor.bomba_corrente_estabilizada', 'on') }}"
      difference_percent: "{{ ((states('sensor.bomba_corrente_media_7d')|float(0) / states('sensor.bomba_corrente_media_30d')|float(0) - 1) * 100) | round(2) }}"
```

**Mudanças**:
1. **Condições extras**: Só avalia quando bomba ligada E estabilizada
2. **delay_on: 30min**: Confirma desgaste persistente (não glitch)
3. **delay_off: 1h**: Aguarda normalização antes de limpar
4. **Attributes extras**: Status bomba, estabilização, diferença percentual

**Justificativa**:
- Desgaste é progressivo (>30min), não pontual
- Ignora transitórios de liga/desliga (avalia apenas quando estabilizada)
- Auto-reset após 1h de operação normal
- Threshold 3% mantido (pode ajustar para 5% se falsos positivos persistirem)

**Problema identificado (screenshot)**:
- Sensor disparando durante desligamento (dI/dt = -0.0443 A/s)
- Sem condição de bomba ligada/estabilizada
- Sem delay → alerta instantâneo para variação temporária

---

## 📢 Mudança 2: Automações TTS

### Arquivo: `automations_bomba.yaml`

### 2.1 Alerta Mudança Rápida - REMOVER TTS

**ANTES**:
```yaml
- id: bomba_alerta_mudanca_rapida
  alias: "Bomba - Alerta Mudança Rápida"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_mudanca_rapida_corrente
      to: "on"
  condition:
    - condition: state
      entity_id: input_boolean.pump_alerts_enabled
      state: "on"
  action:
    - service: system_log.write
      data:
        message: "..."
        level: warning
    
    - service: logbook.log
      data:
        name: "⚡ MUDANÇA RÁPIDA"
        message: "..."
    
    # REMOVER ESTE BLOCO:
    - data:
        message: "Atenção! Bomba com mudança rápida de corrente..."
        volume_type: default
      action: script.nabu_speak
```

**DEPOIS**:
```yaml
- id: bomba_alerta_mudanca_rapida
  alias: "Bomba - Alerta Mudança Rápida"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_mudanca_rapida_corrente
      to: "on"
  condition:
    - condition: state
      entity_id: input_boolean.pump_alerts_enabled
      state: "on"
  action:
    - service: system_log.write
      data:
        message: "..."
        level: info  # warning → info
    
    - service: logbook.log
      data:
        name: "⚡ MUDANÇA RÁPIDA"
        message: "Taxa de mudança: {{ state_attr('binary_sensor.bomba_mudanca_rapida_corrente', 'rate_change') }} A/s"
    
    # TTS REMOVIDO - apenas log
```

**Mudanças**:
- ❌ Removido TTS (script.nabu_speak)
- ✅ Mantido system_log (level: warning → info)
- ✅ Mantido logbook
- Dashboard mostra status em tempo real

---

### 2.2 Alerta Desgaste Emergente - REMOVER TTS

**ANTES**:
```yaml
- id: bomba_alerta_desgaste_emergente
  alias: "Bomba - Alerta Desgaste Emergente"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_desgaste_emergente
      to: "on"
  action:
    # ... system_log, logbook ...
    
    # REMOVER ESTE BLOCO:
    - data:
        message: "Atenção! Bomba apresenta sinais de desgaste progressivo..."
        volume_type: default
      action: script.nabu_speak
```

**DEPOIS**:
```yaml
- id: bomba_alerta_desgaste_emergente
  alias: "Bomba - Alerta Desgaste Emergente"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_desgaste_emergente
      to: "on"
  action:
    - service: system_log.write
      data:
        message: "..."
        level: info  # warning → info
    
    - service: logbook.log
      data:
        name: "📈 DESGASTE EMERGENTE"
        message: "Média 7d > 30d. Considere inspeção preventiva."
    
    # TTS REMOVIDO - alerta preventivo, não urgente
```

---

### 2.3 Alerta Corrente Anormal - REMOVER TTS (manter log)

**ANTES**:
```yaml
- id: bomba_alerta_corrente_anormal
  alias: "Bomba - Alerta Corrente Anormal"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_corrente_anormal
      to: "on"
  action:
    # ... counter, system_log, logbook ...
    
    # REMOVER ESTE BLOCO:
    - data:
        message: "Atenção! Bomba com corrente anormal..."
        volume_type: default
      action: script.nabu_speak
```

**DEPOIS**:
```yaml
- id: bomba_alerta_corrente_anormal
  alias: "Bomba - Alerta Corrente Anormal"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_corrente_anormal
      to: "on"
  action:
    - service: counter.increment
      target:
        entity_id: counter.pump_power_alerts
    
    - service: system_log.write
      data:
        message: "..."
        level: info  # warning → info
    
    - service: logbook.log
      data:
        name: "⚠️ CORRENTE ANORMAL"
        message: "Corrente: {{ states('sensor.bomba_de_circulacao_de_agua_quente_corrente') }}A"
    
    # TTS REMOVIDO - monitoramento técnico, não emergência
```

---

### 2.4 Alerta Corrente Crítica - MANTER TTS (emergência)

**NÃO ALTERAR** - TTS correto para emergência:
```yaml
- id: bomba_alerta_corrente_critica
  alias: "Bomba - Alerta Corrente Crítica"
  trigger:
    - platform: numeric_state
      entity_id: sensor.bomba_de_circulacao_de_agua_quente_corrente
      above: input_number.pump_current_critical_max
      for: "00:00:03"
  action:
    - service: switch.turn_off
      target:
        entity_id: switch.bomba_de_circulacao_de_agua_quente
    
    - data:
        message: "ALERTA CRÍTICO! Bomba desligada por corrente excessiva..."
        volume_type: default
      action: script.nabu_speak  # ✅ MANTER
```

---

### 2.5 Timeout Segurança - MANTER TTS (possível vazamento)

**NÃO ALTERAR** - TTS correto para situação crítica:
```yaml
- id: bomba_timeout_seguranca
  alias: "Bomba - Timeout Segurança"
  action:
    - data:
        message: "Atenção! Bomba desligada por timeout de segurança..."
        volume_type: default
      action: script.nabu_speak  # ✅ MANTER
```

---

## 📊 Comparativo TTS

| Alerta | Antes | Depois | Justificativa |
|--------|-------|--------|---------------|
| Mudança rápida | 🔊 TTS | 📊 Log | Inrush normal, não é emergência |
| Desgaste emergente | 🔊 TTS | 📊 Log | Preventivo, não urgente |
| Corrente anormal | 🔊 TTS | 📊 Log | Monitoramento técnico |
| **Corrente crítica** | 🔊 TTS | 🔊 TTS | **Emergência - desligamento** |
| **Timeout 30min** | 🔊 TTS | 🔊 TTS | **Crítico - possível vazamento** |

---

## ✅ Checklist de Implementação

### Fase 1: Backup
- [ ] Backup `template_sensors.yaml`
- [ ] Backup `automations_bomba.yaml`

### Fase 2: Aplicar Mudanças
- [ ] Editar `template_sensors.yaml` (Mudança 1)
- [ ] Editar `automations_bomba.yaml` (Mudança 2.1, 2.2, 2.3)

### Fase 3: Validar
- [ ] `ha core check` (sem erros)
- [ ] `ha core restart`
- [ ] Verificar logs: nenhum erro de template

### Fase 4: Testar
- [ ] Ligar bomba → aguardar 15s → verificar se inrush foi ignorado
- [ ] Verificar dashboard: sensores estáveis
- [ ] Verificar logbook: sem alertas falsos
- [ ] Testar desgaste_emergente: Sensor só avalia quando bomba ligada + estabilizada
- [ ] Confirmar delay 30min: Alerta não dispara instantaneamente

### Fase 5: Monitorar
- [ ] 24h de operação
- [ ] Conferir logbook: apenas eventos reais
- [ ] Confirmar ausência de falsos positivos

### Fase 6: Documentar
- [ ] Atualizar CHANGELOG.md (v3.5.2)
- [ ] Atualizar CLAUDE.md (quick fixes)
- [ ] Adicionar em lessons-learned.md

**Nota**: Se após 7 dias ainda houver falsos positivos em desgaste_emergente, considerar ajustar threshold de 3% para 5%:
```yaml
{% set threshold_percent = 1.05 %}  # 3% → 5%
```

---

## 📁 Arquivos Finais

Após implementação, teremos:
```
config/
├── template_sensors.yaml      # Threshold 0.025 + anti-inrush
├── automations_bomba.yaml     # TTS estratégico (2 alertas)
└── sensors.yaml               # Sem alterações

docs/
└── lessons-learned.md         # Adicionar: "Falsos positivos v3.5.1"

CHANGELOG.md                   # Adicionar v3.5.2
CLAUDE.md                      # Atualizar quick fixes
```

---

**Pronto para implementação no Claude Code**
