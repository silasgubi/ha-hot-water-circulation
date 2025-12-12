# Bomba Água Quente - Contexto IA

⚠️ **TESTE DE LEITURA**: Se leu este arquivo, comece com "📋 Contexto: Bomba Água Quente v3.5"

## Resumo
Sistema Home Assistant para automação de bomba de circulação de água quente (50W/0.31A). Detecta fluxo via Sonoff Mini, aciona bomba via Tuya TS011F Zigbee. v3.5 usa derivative (dI/dt) para distinguir inrush current de problemas reais.

## Estado Atual
**Versão**: v3.5 | **Data**: 2024-12-12 | **Status**: Pronto para Deploy

## Quick Reference

### Decisões Arquiteturais
| ID | Decisão | Resultado | Detalhes |
|----|---------|-----------|----------|
| 0001 | Corrente vs Potência | Corrente (mais preciso) | [docs/decisions/0001-current-vs-power.md](docs/decisions/0001-current-vs-power.md) |
| 0002 | Derivative para inrush | 5s janela, A/s | Inline |

### Thresholds Calibrados (1826 amostras dez/2024)
| Parâmetro | Valor | Cobertura |
|-----------|-------|-----------|
| Normal min | 0.303A | P5 |
| Normal max | 0.323A | P95 |
| Crítico | 0.388A | max × 1.15 |
| Estabilização | \|dI/dt\| < 0.005 A/s | Após 2s = estável |
| Mudança rápida | \|dI/dt\| > 0.010 A/s | Problema agudo |
| Crescimento | 7d > 30d × 1.03 | Desgaste emergente |

### Entidades Hardware
```yaml
valve.hot_water                                    # Sonoff Mini - detecção fluxo
switch.bomba_de_circulacao_de_agua_quente          # Tuya TS011F - controle bomba
sensor.bomba_de_circulacao_de_agua_quente_corrente # Corrente (A)
sensor.bomba_de_circulacao_de_agua_quente_potencia # Potência (W)
```

### Arquitetura v3.5 - 3 Camadas
```
CAMADA 1 (RAW):
├── sensor.bomba_taxa_mudanca_corrente        # Derivative: dI/dt em A/s
├── sensor.bomba_corrente_media_24h           # Statistics 24h
├── sensor.bomba_corrente_media_7d            # Statistics 7d
└── sensor.bomba_corrente_media_30d           # Statistics 30d (baseline)

CAMADA 2 (DETECÇÃO INTELIGENTE):
├── binary_sensor.bomba_corrente_estabilizada # |dI/dt| < 0.005 por 2s
├── binary_sensor.bomba_mudanca_rapida_corrente # |dI/dt| > 0.010
└── binary_sensor.bomba_desgaste_emergente    # 7d > 30d × 1.03

CAMADA 3 (ALERTAS COMBINADOS):
├── binary_sensor.bomba_corrente_anormal      # (estabilizada AND fora_faixa) OR mudança_rápida
└── binary_sensor.bomba_corrente_critica      # > 0.388A (emergência)
```

### Estrutura de Código
| Funcionalidade | Arquivo | Descrição |
|----------------|---------|-----------|
| Derivative + Statistics | `config/sensors.yaml` | Sensores raw v3.5 |
| Binary sensors + Templates | `config/template_sensors.yaml` | Detecção inteligente |
| Automações v3.5 | `config/automations.yaml` | Controle + alertas |
| Scripts | `config/scripts.yaml` | Teste, reset, relatório |

### Problemas Conhecidos (Quick Fix)
| Problema | Solução |
|----------|---------|
| Inrush dispara alerta | v3.5 resolve: usa `estabilizada` |
| Sonoff stuck em "open" | Filtro 3s delay em automação |
| Derivative em A/min | CORRIGIDO v3.5: unit_time: s |

**Detalhes**: [docs/lessons-learned.md](docs/lessons-learned.md)

## Funcionalidades por Versão

### v2.0 (Produção atual)
- ✅ Monitoramento por potência (48-53W)
- ✅ 22 helpers, 6 automações, 3 scripts
- ✅ Timeout 30min, cooldown 3min

### v3.0 (Desenvolvido, não implementado)
- ✅ Migração para corrente (0.303-0.323A)
- ✅ Statistics sensors (24h/7d/30d)
- ❌ Derivative com parâmetros errados

### v3.5 (Este release)
- ✅ Derivative CORRIGIDO: `unit_time: s`, `time_window: 00:00:05`
- ✅ 3 novos binary_sensors (estabilizada, mudança_rápida, desgaste_emergente)
- ✅ Lógica inteligente ignora inrush automaticamente
- ✅ Mensagens com unidades corretas (A/s)

## Funcionalidades Descartadas
- ❌ Cycle counting: Removido v3.0 (logs do HA suficientes)
- ❌ Energy cost monitoring: Removido v3.0 (utility_meter nativo)
- ❌ Cooldown timer: Removido v3.0 (proteção por corrente é melhor)

## Input Numbers Necessários
```yaml
input_number.pump_current_normal_min    # 0.303
input_number.pump_current_normal_max    # 0.323  
input_number.pump_current_critical_max  # 0.388
input_number.pump_activation_delay_seconds    # 3
input_number.pump_deactivation_delay_seconds  # 20
input_number.pump_timeout_minutes       # 30
```

## Timers Necessários
```yaml
timer.pump_activation_delay
timer.pump_deactivation_delay
timer.pump_safety_timeout
```

## Deploy

### Pré-requisitos
1. Criar input_numbers se não existem
2. Criar timers se não existem
3. `input_boolean.pump_alerts_enabled` para TTS

### Passos
1. Backup arquivos atuais
2. Copiar `config/sensors.yaml` → `/config/sensors.yaml`
3. Copiar seção bomba de `config/template_sensors.yaml` → `/config/template_sensors.yaml`
4. Copiar automações bomba de `config/automations.yaml` → `/config/automations.yaml`
5. `ha core check && ha core restart`

### Validação
```bash
# Verificar derivative
ha state get sensor.bomba_taxa_mudanca_corrente

# Verificar novos binary_sensors
ha state get binary_sensor.bomba_corrente_estabilizada
ha state get binary_sensor.bomba_mudanca_rapida_corrente
ha state get binary_sensor.bomba_desgaste_emergente

# Teste funcional
# Ligar bomba → após ~2s estabilizada=on → corrente_anormal=off (inrush ignorado)
```

## Próximos Passos
- [ ] Deploy v3.5 em produção
- [ ] Validar comportamento inrush
- [ ] Coletar dados 7 dias para confirmar thresholds
- [ ] Ajustar se necessário

## Links
- **Lessons Learned**: [docs/lessons-learned.md](docs/lessons-learned.md)
- **ADR Corrente vs Potência**: [docs/decisions/0001-current-vs-power.md](docs/decisions/0001-current-vs-power.md)
