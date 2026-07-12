# Pendências — v3.7.0

## Instalação no Home Assistant

**Status: ✅ INSTALADO em z:\ — 2026-07-12 (falta apenas o restart do HA)**

Todos os arquivos abaixo já foram aplicados em `z:\`, incluindo o dashboard
(`z:\dashboards\bomba_agua_quente.yaml`, deploy via `deploy_dashboards.py`).
`input_boolean.pump_lockout` e `sensor.bomba_corrente_sob_carga` só passam a
existir depois do restart (helper novo em `configuration.yaml`).

Mudanças da v3.7.0 (correções da análise crítica):

1. **Corrente sob carga**: novo `sensor.bomba_corrente_sob_carga` (template) — corrente só quando `bomba_corrente_estabilizada` on, `unavailable` no resto. Statistics 7d/30d agora apontam para ele (antes as médias eram diluídas por amostras idle e mediam padrão de uso, não saúde do motor).
2. **Lockout de emergência**: `input_boolean.pump_lockout` — ligado pelo desligamento por corrente crítica, bloqueia a automação principal de religar a bomba até reset manual.
3. **Guard de disponibilidade**: `binary_sensor.bomba_falha_circulacao` fica `unavailable` se o sensor de temperatura cair (antes `float(0)` → falso alarme de "falha Zigbee").
4. **Notificações de desgaste**: alerta de desgaste emergente agora envia push mobile + notificação persistente (antes só system_log/logbook — invisível).

### Arquivos a aplicar em `z:\`

| Arquivo repo | Destino z:\ | Mudança |
|---|---|---|
| `config/template_sensors.yaml` | `z:\template_sensors.yaml` | + sensor sob carga, + availability na falha circulação |
| `config/sensors.yaml` | `z:\sensors.yaml` | Statistics 7d/30d → `sensor.bomba_corrente_sob_carga` |
| `config/automations_bomba.yaml` | `z:\automations\systems\water_pump.yaml` | Lockout (condição + set), notificações desgaste |
| `config/scripts_bomba.yaml` | `z:\scripts` (bloco bomba) | Reset limpa `pump_lockout` |
| `config/helpers_bomba.yaml` | `z:\configuration.yaml` | Novo `input_boolean.pump_lockout` |

### Passos

1. ~~Aplicar arquivos acima em `z:\`~~ ✅ feito
2. **Restart HA** (helper novo em configuration.yaml exige restart) — pendente
3. Verificar que `input_boolean.pump_lockout` existe e está **off**
4. Verificar `sensor.bomba_corrente_sob_carga` = `unavailable` com bomba parada, e = corrente (~0.31A) durante um ciclo de bomba

### ⚠️ Efeito colateral esperado

As médias 7d/30d **resetam** ao trocar de fonte e rebuildam em 7/30 dias. Nesse período o binary sensor de desgaste fica em `false` (guard de `media_30d > 0`) e a automação das 22h é bloqueada pela condição de disponibilidade — comportamento correto, sem falsos alarmes. Detecção de desgaste volta a operar plenamente ~30 dias após instalação.

### Teste do lockout

1. Ligar `input_boolean.pump_lockout` manualmente
2. Abrir torneira → bomba NÃO deve ligar
3. Rodar `script.pump_emergency_reset` → lockout off
4. Abrir torneira → bomba liga normalmente

---

## Herdado da v3.6.0

- Recalibrar `input_number.bomba_temp_funcionamento_minima` (32°C provisório) após 7-14 dias de dados
- Considerar threshold adaptativo por estação (delta relativo)
