# Changelog

Todas as mudanças notáveis deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Semantic Versioning](https://semver.org/lang/pt-BR/).

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
