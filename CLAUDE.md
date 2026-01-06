# Bomba Água Quente - Contexto IA

⚠️ **TESTE DE LEITURA**: Se leu, comece com "📋 Contexto: Bomba Água Quente v3.5.2"

## Resumo
Sistema Home Assistant que aciona bomba de circulação (50W) via detecção automática de fluxo, com monitoramento inteligente por corrente (v3.5.2) incluindo detecção de inrush, mudanças rápidas e desgaste progressivo.

## Estado Atual
**Versão**: v3.5.2 | **Data**: 2026-01-06 | **Status**: ✅ Produção (testado e validado)

## Quick Reference

### Decisões Arquiteturais
| ID | Decisão | Resultado | Detalhes |
|----|---------|-----------|----------|
| 0001 | Corrente vs Potência | Corrente (0.303-0.323A) | [docs/decisions/0001-current-vs-power.md](docs/decisions/0001-current-vs-power.md) |
| 0002 | Filtro Anti-Inrush v3.5.2 | Threshold 0.025 A/s + delay 10s | [docs/decisions/0002-derivative-inrush-filtering.md](docs/decisions/0002-derivative-inrush-filtering.md) |

### Componentes Críticos
**Hardware**:
- **Detecção**: Sonoff Mini PSF-BD1-GL (eWeLink, firmware 3.8.1)
- **Relé**: Tuya TS011F Zigbee (com medição corrente)
- **Bomba**: 50W nameplate / 68W real / 0.309A média

**Thresholds Calibrados** (1.826 amostras, dez/2024):
- Normal: 0.303-0.323A (P5-P95)
- Crítico: 0.388A (Max × 1.15)
- Mudança rápida: 0.025 A/s (P97) - v3.5.2
- Estabilização: |dI/dt| < 0.005 A/s
- Desgaste: 7d > 30d × 1.03

### Arquitetura v3.5.2 (3 Camadas)
```
CAMADA 1 (SENSORES RAW):
├── sensor.bomba_taxa_mudanca_corrente        # Derivative 5s
├── sensor.bomba_corrente_media_24h           # Statistics 24h
├── sensor.bomba_corrente_media_7d            # Statistics 7d
└── sensor.bomba_corrente_media_30d           # Statistics 30d

CAMADA 2 (DETECÇÃO INTELIGENTE):
├── binary_sensor.bomba_corrente_estabilizada # |dI/dt| < 0.005 A/s por 2s
├── binary_sensor.bomba_mudanca_rapida_corrente # |dI/dt| > 0.025 A/s (filtro 10s + hysteresis)
└── binary_sensor.bomba_desgaste_emergente    # 7d > 30d × 1.03 (delay 30min/1h)

CAMADA 3 (ALERTAS COMBINADOS):
├── binary_sensor.bomba_corrente_anormal      # (estabilizada AND fora_faixa) OR mudança_rápida
└── binary_sensor.bomba_corrente_critica      # > 0.388A (emergência)
```

### Estrutura de Código
| Funcionalidade | Arquivo | Descrição |
|----------------|---------|-----------|
| Derivative + Statistics | `config/sensors.yaml` | Sensores raw v3.5 |
| Binary sensors + Templates | `config/template_sensors.yaml` | Detecção inteligente v3.5.2 |
| Automações v3.5.2 | `config/automations_bomba.yaml` | Controle + alertas (TTS estratégico) |
| Scripts | `config/scripts_bomba.yaml` | Teste, reset, relatório |
| Dashboard Sidebar (Mushroom) | `config/lovelace_bomba_sidebar.yaml` | Dashboard completo para sidebar (RECOMENDADO) |
| Dashboard Sidebar (Native) | `config/lovelace_bomba_sidebar_native.yaml` | Dashboard sem custom cards |
| Dashboard Card | `config/dashboard_bomba_v35.yaml` | Card único v3.5 (legado) |

### Problemas Conhecidos (Quick Fix)
| Problema | Status | Solução |
|----------|--------|---------|
| Falsos positivos inrush | ✅ RESOLVIDO | v3.5.2: threshold 0.025 A/s + filtro 10s + hysteresis |
| TTS excessivo | ✅ RESOLVIDO | v3.5.2: apenas emergências (crítico + timeout) |
| Input numbers não atualizados | ✅ RESOLVIDO | Configurar 0.303A/0.323A via UI |
| Sonoff stuck em "open" | ⚠️ WORKAROUND | Filtro 3s delay em automação |
| Derivative em A/min | ✅ CORRIGIDO | v3.5: unit_time: s |
| Cooldown timer bloqueava | ✅ CORRIGIDO | v3.5.1: condição removida |

**Detalhes**: [docs/lessons-learned.md](docs/lessons-learned.md)

## Arquitetura

### Fluxo de Operação v3.5.2
```
┌─────────────────┐
│ Torneira Aberta │
└────────┬────────┘
         │
    ┌────▼────┐ open
    │ Sonoff  ├────────────────┐
    │  Mini   │                │
    └─────────┘                │
                               │
              ┌────────────────▼────────────────┐
              │ Automação Principal             │
              │ - Delay 3s (confirma fluxo)     │
              │ - Verifica override manual      │
              └────────────────┬────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │ Liga Bomba (Tuya)   │
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┴─────────────────────┐
         │                                           │
    ┌────▼────┐                                 ┌────▼────┐
    │ INRUSH  │ 0-10s                          │ ESTÁVEL │ >10s
    │ Filtrado│ (sensor ignora)                │ Monitor │ (detecta problemas)
    └─────────┘                                 └────┬────┘
                                                     │
                             ┌───────────────────────┼───────────────────────┐
                             │                       │                       │
                        ┌────▼────┐            ┌────▼────┐            ┌────▼────┐
                        │ Normal  │            │ Anormal │            │ Crítico │
                        │ 0.303-  │            │ dI/dt>  │            │ >0.388A │
                        │ 0.323A  │            │ 0.025   │            │         │
                        └─────────┘            └────┬────┘            └────┬────┘
                                                    │                      │
                                              ┌─────▼─────┐          ┌─────▼─────┐
                                              │ Log Info  │          │🔊 TTS +   │
                                              │ Dashboard │          │ Desliga   │
                                              └───────────┘          └───────────┘
```

## Funcionalidades por Versão

### v3.5.2 (✅ PRODUÇÃO ATUAL - 2026-01-06)
- ✅ Threshold mudança rápida: 0.025 A/s (P97)
- ✅ Filtro anti-inrush: 10s + hysteresis 3s/5s
- ✅ Desgaste emergente: condições + delays 30min/1h
- ✅ TTS estratégico: apenas 2 alertas (crítico + timeout)
- ✅ Input numbers configurados: 0.303A/0.323A
- ✅ Zero falsos positivos validado

### v3.5.1 (Produção 2024-12-12 a 2026-01-06)
- ✅ Bug cooldown corrigido
- ✅ Dashboard v3.5 com DEBUG
- ⚠️ Threshold 0.010 A/s causava falsos positivos (26% amostras)

### v3.5 (Produção 2024-12-12 a 2024-12-12)
- ✅ Derivative CORRIGIDO: `unit_time: s`, `time_window: 00:00:05`
- ✅ 3 binary_sensors inteligentes
- ⚠️ Threshold 0.010 A/s muito baixo

### v3.0 (Desenvolvido, não implantado)
- ✅ Migração corrente
- ✅ Statistics 24h/7d/30d
- ❌ Derivative com parâmetros errados

### v2.0 (Legado)
- ✅ Monitoramento por potência (48-53W)
- ✅ 22 helpers

## Funcionalidades Descartadas
- ❌ Cycle counting: Removido v3.0 (logs do HA suficientes)
- ❌ Energy cost monitoring: Removido v3.0 (utility_meter nativo)
- ❌ Cooldown timer: Removido v3.0 (proteção por corrente melhor)

## Dependências
- Home Assistant ≥ 2024.1
- eWeLink Integration (Sonoff)
- Zigbee2MQTT ou ZHA (Tuya)
- Nabu Casa TTS (opcional, para alertas sonoros)

## Debugging
```bash
# Verificar sensores
ha state get sensor.bomba_taxa_mudanca_corrente
ha state get binary_sensor.bomba_mudanca_rapida_corrente

# Verificar thresholds
ha state get input_number.pump_current_normal_min  # Deve ser 0.303
ha state get input_number.pump_current_normal_max  # Deve ser 0.323

# Logs
ha core logs | grep -i "bomba\|pump"
```

## Lições (Quick Ref)
- **Calibração**: SEMPRE usar dados reais (mínimo 100 amostras), nunca teóricos
- **Percentis**: P5/P95 > min/max para thresholds (exclui outliers)
- **Anti-inrush**: Filtrar primeiros 10s + hysteresis evita falsos positivos
- **TTS estratégico**: Apenas emergências (60% menos ruído)

**Detalhes**: [docs/lessons-learned.md](docs/lessons-learned.md)

## Próximos Passos
✅ **v3.5.2 estável** - Sistema operando perfeitamente
- Nenhuma mudança planejada
- Monitoramento contínuo por 7-30 dias

## Referências
- [CHANGELOG.md](CHANGELOG.md) - Histórico de versões
- [docs/analysis/ANALISE_CORRENTE_20241218.md](docs/analysis/ANALISE_CORRENTE_20241218.md) - Análise 1.826 amostras
- [docs/decisions/0002-derivative-inrush-filtering.md](docs/decisions/0002-derivative-inrush-filtering.md) - ADR v3.5.2
- [docs/lessons-learned.md](docs/lessons-learned.md) - Lições técnicas
