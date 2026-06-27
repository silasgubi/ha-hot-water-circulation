# Contexto Arquitetural — Bomba Água Quente

Documento de referência para decisões de design, histórico de versões e diagnóstico. Ver [CLAUDE.md](../CLAUDE.md) para quick reference.

---

## Arquitetura em 3 Camadas (v3.6.0)

```
CAMADA 1 — SENSORES RAW
├── sensor.bomba_de_circulacao_de_agua_quente_corrente  # Tuya TS011F (tempo real)
├── sensor.bomba_taxa_mudanca_corrente                  # Derivative 5s (A/s)
├── sensor.sonoff_1000bcbe07_temperature                # Cano pós-bomba (°C)
├── sensor.bomba_taxa_mudanca_temperatura               # Derivative 5min (°C/min) — v3.6.0
├── sensor.bomba_corrente_media_24h                     # Statistics mean 24h
├── sensor.bomba_corrente_media_7d                      # Statistics mean 7d
└── sensor.bomba_corrente_media_30d                     # Statistics mean 30d

CAMADA 2 — DETECÇÃO INTELIGENTE
├── binary_sensor.bomba_corrente_estabilizada     # |dI/dt| < 0.005 A/s por 2s
├── binary_sensor.bomba_mudanca_rapida_corrente   # |dI/dt| > 0.025 A/s (filtro 10s + hysteresis)
├── binary_sensor.bomba_desgaste_emergente        # 7d > 30d × 1.03 (delay 30min/1h)
└── binary_sensor.bomba_falha_circulacao          # temp < 32°C por 2min com bomba/válvula ativa — v3.6.0

CAMADA 3 — ALERTAS COMBINADOS
├── binary_sensor.bomba_corrente_anormal   # (estabilizada AND fora_faixa) OR mudança_rápida
└── binary_sensor.bomba_corrente_critica  # corrente > 0.388A (emergência, sem delay)
```

---

## Fluxo de Operação

```
Torneira Aberta
      │
  valve.hot_water = open
      │
  Automação Principal
  ├── delay 3s (confirma fluxo real)
  └── verifica override manual
      │
  switch.bomba = ON
      │
  ┌───┴───────────────────────────────┐
  │                                   │
INRUSH (0-10s)                   ESTÁVEL (>10s)
Sensor ignora                    Monitoramento ativo
                                       │
                    ┌──────────────────┼───────────────────┐
                    │                  │                   │
               CORRENTE           TEMPERATURA           DESGASTE
               normal?            subiu?                tendência?
                    │                  │                   │
              0.303-0.323A      >32°C em 2min         7d vs 30d
                    │                  │                   │
              ✅ OK          ❌ FALHA     ❌ DESGASTE
                         ⚠️ TTS + Push  📊 Log + Notif.
                         (Zigbee fault)

CRÍTICO (>0.388A):
→ Desliga bomba imediatamente
→ TTS emergência (skip_quiet_check)
→ Push iPhone critical
```

---

## Versões

### v3.6.0 (2026-06-27) — ⏳ Aguardando instalação
- **Novo**: `binary_sensor.bomba_falha_circulacao` — detecta falha Zigbee silenciosa via temperatura
- **Novo**: `sensor.bomba_taxa_mudanca_temperatura` — derivativo °C/min
- **Novo**: `input_number.bomba_temp_funcionamento_minima` — threshold 32°C configurável
- **Novo**: automação #8 `bomba_alerta_falha_circulacao` — TTS + push iPhone + persistente
- **Motivação**: Zigbee falha de inicialização deixava o switch como "on" no HA mas relé inativo; corrente elétrica não detecta isso pois não flui

### v3.5.2 (2026-01-06) — ✅ Produção
- Threshold mudança rápida: 0.025 A/s (P97) — era 0.010 A/s (26% de falsos positivos)
- Filtro anti-inrush: ignora primeiros 10s + hysteresis 3s/5s
- TTS estratégico: apenas 2 alertas (crítico + timeout) — 60% menos ruído
- Input numbers corretos: 0.303A/0.323A

### v3.5.1 (2024-12-14)
- Bug corrigido: `timer.pump_anti_cycle_cooldown` removido em v3.0 mas condição permanecia, bloqueando automação principal

### v3.5.0 (2024-12-12)
- Derivative CORRIGIDO: `unit_time: s` (era `min`), `time_window: 5s` (era `5min`)
- Sensor reportava A/min em vez de A/s — thresholds inúteis

### v3.0.0 (2024-12-11)
- Migração corrente → substituiu monitoramento por potência
- Statistics 24h/7d/30d para análise de tendência

### v2.0.0 (2024-01)
- Monitoramento por potência (48–53W, 143 medições)
- 22 helpers, 6 automações, cooldown timer, 30 ciclos/hora

---

## Hardware

| Componente | Modelo | Protocolo | Função |
|---|---|---|---|
| Detecção de fluxo | Sonoff Mini PSF-BD1-GL | eWeLink Wi-Fi | Detecta abertura torneira (valve.hot_water) |
| Relé + medição | Tuya TS011F | Zigbee | Aciona bomba + mede corrente |
| Temperatura | Sonoff 1000BCBE07 | Wi-Fi | Sensor cano pós-bomba |
| Bomba | 50W nameplate | — | 68W real, 0.309A média |

**Zigbee**: o rádio Zigbee do HA ocasionalmente falha na inicialização → switch mostra "on" mas relé não fecha. A corrente não sobe (não há circuito fechado) mas o HA não detecta isso. O sensor de temperatura é a única camada capaz de detectar essa falha.

---

## Calibração

### Corrente (1.826 amostras, dez/2024)
| Métrica | Valor |
|---|---|
| P5 (normal_min) | 0.303A |
| P95 (normal_max) | 0.323A |
| Média | 0.309A |
| Máximo real | 0.337A |
| Crítico (max × 1.15) | 0.388A |
| P97 (mudança rápida) | 0.025 A/s |

Análise completa: [docs/analysis/ANALISE_CORRENTE_20241218.md](analysis/ANALISE_CORRENTE_20241218.md)

### Temperatura (dados limitados — banco resetado jun/2026)
| Estado | Temperatura |
|---|---|
| Circulando | 35–46°C |
| Frio / falha | 24–30°C |
| Threshold detecção | 32°C ⚠️ provisório |

Recalibrar após 7-14 dias de operação normal coletando dados.

---

## Funcionalidades Descartadas

| Funcionalidade | Versão removida | Motivo |
|---|---|---|
| Cycle counting | v3.0 | Logs do HA suficientes |
| Energy cost monitoring | v3.0 | `utility_meter` nativo do HA |
| Cooldown timer | v3.0 | Proteção por corrente mais confiável |

---

## Dependências

- Home Assistant ≥ 2024.1
- eWeLink Integration (Sonoff)
- Zigbee2MQTT ou ZHA (Tuya)
- `script.nabu_speak` (Nabu Intelligence System — `Z:\scripts.yaml`)
- `notify.mobile_app_iphone_do_silas` (HA Companion App no iPhone de Silas)

---

## Debugging

```bash
# Sensores corrente
ha state get sensor.bomba_de_circulacao_de_agua_quente_corrente
ha state get sensor.bomba_taxa_mudanca_corrente
ha state get binary_sensor.bomba_mudanca_rapida_corrente
ha state get binary_sensor.bomba_corrente_critica

# Sensores temperatura (v3.6.0)
ha state get sensor.sonoff_1000bcbe07_temperature
ha state get sensor.bomba_taxa_mudanca_temperatura
ha state get binary_sensor.bomba_falha_circulacao

# Thresholds
ha state get input_number.pump_current_normal_min   # 0.303
ha state get input_number.pump_current_normal_max   # 0.323
ha state get input_number.pump_current_critical_max # 0.388
ha state get input_number.bomba_temp_funcionamento_minima  # 32

# Logs
ha core logs | grep -i "bomba\|pump\|circulação"
```

---

## Lições Aprendidas

- **Calibração**: SEMPRE usar dados reais (mínimo 100 amostras), nunca valores teóricos
- **Percentis**: P5/P95 > min/max para thresholds — exclui outliers sem perder cobertura
- **Anti-inrush**: Filtrar primeiros 10s + hysteresis evita 100% dos falsos positivos
- **TTS estratégico**: apenas emergências — 60% menos ruído no dia a dia
- **Derivative unit_time**: sempre verificar — `min` vs `s` muda os valores em 60×
- **Camadas independentes**: corrente e temperatura detectam falhas diferentes; uma não substitui a outra

Detalhes expandidos: [lessons-learned.md](lessons-learned.md)

---

## Referências

- [CHANGELOG.md](../CHANGELOG.md) — histórico completo
- [docs/pending.md](pending.md) — pendências de instalação v3.6.0
- [docs/decisions/0001-current-vs-power.md](decisions/0001-current-vs-power.md) — ADR corrente vs potência
- [docs/decisions/0002-derivative-inrush-filtering.md](decisions/0002-derivative-inrush-filtering.md) — ADR anti-inrush
- [docs/analysis/ANALISE_CORRENTE_20241218.md](analysis/ANALISE_CORRENTE_20241218.md) — análise 1.826 amostras
- [docs/lessons-learned.md](lessons-learned.md) — lições técnicas detalhadas
