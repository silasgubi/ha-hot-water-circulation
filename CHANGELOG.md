# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Semantic Versioning](https://semver.org/lang/pt-BR/).

## [3.5.2] - 2026-01-06

### Corrigido
- **Threshold mudança rápida**: Aumentado de 0.010 para 0.025 A/s (P97) - elimina falsos positivos de inrush normal
- **Filtro anti-inrush**: Ignora primeiros 10s após ligação da bomba
- **Hysteresis**: Adicionado delay_on=3s e delay_off=5s em `binary_sensor.bomba_mudanca_rapida_corrente`
- **Desgaste emergente**: Adicionadas condições (bomba_on + estabilizada) e delays (30min on/1h off)

### Modificado
- **TTS estratégico**: Removido de alertas não-críticos (mudança rápida, desgaste emergente, corrente anormal)
- **TTS mantido apenas em**: Corrente crítica (>0.388A) e timeout (30min) - redução de 60% no ruído
- **Log levels**: Alterados de `warning` para `info` nos alertas de monitoramento técnico
- Adicionados attributes extras aos sensores:
  - `bomba_mudanca_rapida_corrente`: `time_since_activation`, `inrush_filter_active`
  - `bomba_desgaste_emergente`: `pump_running`, `current_stable`, `difference_percent`

### Adicionado
- **Delay desgaste emergente**: delay_on=30min confirma persistência, delay_off=1h permite auto-reset
- **Condições inteligentes**: Sensor de desgaste só avalia quando bomba ligada + corrente estabilizada

### Justificativa
- Análise de 1.826 amostras revelou que threshold 0.010 A/s capturava 26% das amostras (inrush normal)
- Novo threshold 0.025 A/s captura apenas eventos reais (P97-P99)
- TTS reduzido de 5 para 2 alertas: apenas emergências reais (corrente crítica e timeout)
- Monitoramento técnico via dashboard e logs (sem poluição sonora)
- Desgaste emergente agora ignora transitórios de liga/desliga

## [3.5.1] - 2025-12-12

### Corrigido
- **Bug cooldown timer**: Removida condicao `timer.pump_anti_cycle_cooldown == "idle"` que bloqueava automacao (timer removido na v3.0)
- **Encoding UTF-8**: Corrigidos caracteres corrompidos em `automations_bomba.yaml` e `scripts_bomba.yaml`

### Adicionado
- **Dashboard v3.5**: `config/dashboard_bomba_v35.yaml` com layout grid 2 colunas
- **Secao DEBUG**: Timers, automacoes e checklist de diagnostico no dashboard

### Modificado
- Renomeados arquivos: `automations.yaml` -> `automations_bomba.yaml`, `scripts.yaml` -> `scripts_bomba.yaml`

## [3.5.0] - 2024-12-12

### Corrigido
- **Derivative sensor**: `unit_time: s` (era `min`), `time_window: "00:00:05"` (era `"00:05:00"`)
- **Mensagens automações**: "A/s" em vez de "A/min"
- **Descrição desgaste emergente**: "comparação 7d vs 30d" em vez de "sustentada >30min"

### Adicionado
- `binary_sensor.bomba_corrente_estabilizada`: |dI/dt| < 0.005 A/s por 2s
- `binary_sensor.bomba_mudanca_rapida_corrente`: |dI/dt| > 0.010 A/s
- `binary_sensor.bomba_desgaste_emergente`: média_7d > média_30d × 1.03
- Attributes detalhados em todos os novos sensores

### Modificado
- `binary_sensor.bomba_corrente_anormal`: Nova lógica `(estabilizada AND fora_faixa) OR mudança_rápida`
- Removido `delay_on` de 3s (substituído por lógica de estabilização)

## [3.0.0] - 2024-12-11

### Adicionado
- Monitoramento por corrente (substitui potência)
- Statistics sensors: 24h, 7d, 30d
- Derivative sensor (com parâmetros incorretos)
- Detecção de desgaste progressivo

### Modificado
- Thresholds: Corrente 0.303-0.323A (era Potência 48-53W)
- Threshold crítico: 0.388A (max × 1.15)

### Removido
- Cycle counting (logs do HA suficientes)
- Energy cost monitoring (utility_meter nativo)
- Cooldown timer (proteção por corrente é melhor)

## [2.0.1] - 2024-01

### Modificado
- Dashboard otimizado para iPad

## [2.0.0] - 2024-01

### Adicionado
- Valores configuráveis via input_numbers
- 22 helpers, 6 automações, 3 scripts
- Timeout 30min, cooldown 3min, limite 20 ciclos/hora
- Dashboard completo

### Modificado
- Thresholds de potência calibrados (143 medições): 48-53W

## [1.0.0] - 2024-01

### Adicionado
- Implementação inicial com valores fixos
- Detecção de fluxo via Sonoff Mini
- Controle de bomba via Tuya TS011F
