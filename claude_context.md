# CLAUDE_CONTEXT.md - Sistema Automação Bomba Água Quente

## RESUMO EXECUTIVO
Sistema Home Assistant que aciona automaticamente bomba de circulação (50W) ao detectar fluxo em torneira de água quente via Sonoff Mini, utilizando relé Zigbee Tuya TS011F. Inclui múltiplas camadas de proteção (timeout 30min, cooldown 3min, limite 20 ciclos/hora), delays configuráveis (3s liga/20s desliga) e monitoramento de potência calibrado (normal: 48-53W baseado em 143 medições reais).

## ENTIDADES CRÍTICAS

### Hardware (DADOS REAIS)
```yaml
# Detecção de fluxo
valve.hot_water  # Sonoff Mini (eWeLink)

# Controle da bomba
switch.bomba_de_circulacao_de_agua_quente  # Tuya TS011F (Zigbee)

# Monitoramento
sensor.bomba_de_circulacao_de_agua_quente_potencia  # Watts em tempo real
sensor.bomba_de_circulacao_de_agua_quente_summation_delivered  # kWh total
```

### Helpers (22 ENTIDADES)

**Timers (4):**
```yaml
timer.pump_activation_delay         # 00:00:30 - Delay para ligar
timer.pump_deactivation_delay       # 00:01:00 - Delay para desligar
timer.pump_safety_timeout           # 01:00:00 - Proteção máxima
timer.pump_anti_cycle_cooldown      # 00:10:00 - Cooldown entre ciclos
```

**Input Booleans (3):**
```yaml
input_boolean.pump_manual_override   # Desabilita automação
input_boolean.pump_manual_control    # Controle manual ON/OFF
input_boolean.pump_alerts_enabled    # Habilita notificações
```

**Counters (4):**
```yaml
counter.pump_hourly_cycles          # Reset automático a cada hora (max: 50)
counter.pump_daily_cycles           # Reset 00:00 (max: 999)
counter.pump_timeout_events         # Histórico total timeouts (max: 9999)
counter.pump_power_alerts           # Histórico alertas potência (max: 9999)
```

**Input DateTime (2):**
```yaml
input_datetime.pump_last_activation  # Timestamp última ativação
input_datetime.pump_last_timeout     # Timestamp último timeout
```

**Input Numbers (9) - CONFIGURÁVEIS:**
```yaml
# Thresholds Potência
input_number.pump_power_threshold_low      # 20-50W, step 1, default 40
input_number.pump_power_threshold_high     # 50-100W, step 1, default 60
input_number.pump_power_normal_min         # 30-60W, step 1, default 48
input_number.pump_power_normal_max         # 40-80W, step 1, default 53

# Limites Operacionais
input_number.pump_max_daily_runtime        # 1-24h, step 0.5, default 8
input_number.pump_max_hourly_cycles        # 5-50, step 1, default 20

# Tempos
input_number.pump_timeout_minutes          # 5-60min, step 5, default 30
input_number.pump_cooldown_minutes         # 0-10min, step 1, default 3
input_number.pump_activation_delay_seconds # 0-30s, step 1, default 3
input_number.pump_deactivation_delay_seconds # 0-60s, step 5, default 20

# Custo
input_number.bomba_energy_tariff          # 0.1-2 R$/kWh, step 0.01, default 0.85
```

### Template Sensors (9)

**Sensores de Estado:**
```yaml
sensor.bomba_status                # Funcionando/Ligada sem carga/Desligada
sensor.bomba_sistema_status        # Manual/Limite próximo/Funcionando/Ativo/Standby
sensor.bomba_runtime_hoje          # Horas funcionamento (history lookup)
sensor.bomba_eficiencia            # Minutos por ciclo (runtime/ciclos)
sensor.bomba_custo_hoje            # R$ = energy * tarifa
```

**Sensores Binários:**
```yaml
binary_sensor.bomba_funcionando    # power > threshold_low
binary_sensor.fluxo_agua_quente    # valve == 'open'
binary_sensor.bomba_modo_seguro    # cycles < max AND override == 'off'
binary_sensor.bomba_potencia_anormal # power fora dos limites quando ligada
```

### Utility Meters (3)
```yaml
sensor.pump_energy_daily    # Reset diário
sensor.pump_energy_weekly   # Reset semanal
sensor.pump_energy_monthly  # Reset mensal
```

### Automações (6)

**1. bomba_agua_quente_controle_principal**
- Trigger: valve.hot_water state change
- Lógica complexa de ativação/desativação com delays
- Proteções: override, ciclos, cooldown, modo seguro

**2. bomba_timeout_seguranca**
- Trigger: timer.pump_safety_timeout finished
- Desliga bomba forçadamente
- Incrementa counter.pump_timeout_events
- Atualiza input_datetime.pump_last_timeout

**3. bomba_controle_manual**
- Trigger: input_boolean.pump_manual_control toggle
- Override de automação principal
- Mantém proteções (timeout, limites)

**4. bomba_reset_contador_horario**
- Trigger: cron a cada hora cheia
- Reset counter.pump_hourly_cycles

**5. bomba_reset_contador_diario**
- Trigger: time 00:00:00
- Reset counter.pump_daily_cycles

**6. bomba_relatorio_diario**
- Trigger: time 22:00:00
- Envia notificação com estatísticas do dia

### Scripts (3)

**pump_system_test:**
- Teste completo: liga → aguarda 5s → desliga
- Log no logbook

**pump_emergency_reset:**
- Desliga bomba
- Cancela todos timers
- Desabilita overrides
- Cria notificação persistente

**pump_detailed_report:**
- Notificação com todos os dados (estatísticas, contadores, configurações, thresholds)

## ARQUITETURA DO SISTEMA

### Fluxo de Dados
```
[Torneira] → [valve.hot_water] 
    ↓
[automation.bomba_agua_quente_controle_principal]
    ↓ (verifica condições)
[timer.pump_activation_delay] → 3s
    ↓
[switch.bomba_...] → ON
    ↓
[timer.pump_safety_timeout] → 30min (proteção)
    ↓
[sensor...potencia] → 48-53W (monitoramento)
    ↓
[valve.hot_water] → closed
    ↓
[timer.pump_deactivation_delay] → 20s
    ↓
[switch.bomba_...] → OFF
    ↓
[timer.pump_anti_cycle_cooldown] → 3min
```

### Camadas de Proteção

**Nível 1 - Hardware:**
- Tuya TS011F com proteção sobrecarga (16A)

**Nível 2 - Software:**
- Timeout safety: desliga após 30min
- Cooldown: mínimo 3min entre ciclos
- Limite ciclos: máximo 20/hora

**Nível 3 - Lógica:**
- Delays para confirmar intenção
- Override manual para emergências
- Modo seguro verifica todas condições

**Nível 4 - Monitoramento:**
- Alertas potência anormal (<40W ou >60W)
- Contadores de eventos críticos
- Timestamps para auditoria

## LÓGICA DE FUNCIONAMENTO

### Automação Principal (Pseudocódigo)

```python
if valve.hot_water == 'open':
    if NOT pump_manual_override:
        if pump_modo_seguro:  # cycles < max AND cooldown idle
            if timer.pump_anti_cycle_cooldown == 'idle':
                # Inicia delay de ativação
                timer.pump_activation_delay.start(
                    duration=input_number.pump_activation_delay_seconds
                )
                
                # Quando timer termina:
                if valve still 'open':  # reconfirma
                    switch.bomba.turn_on()
                    counter.pump_hourly_cycles.increment()
                    counter.pump_daily_cycles.increment()
                    input_datetime.pump_last_activation = now()
                    
                    # Inicia timeout de segurança
                    timer.pump_safety_timeout.start(
                        duration=input_number.pump_timeout_minutes * 60
                    )

elif valve.hot_water == 'closed':
    if switch.bomba == 'on':
        # Inicia delay de desativação
        timer.pump_deactivation_delay.start(
            duration=input_number.pump_deactivation_delay_seconds
        )
        
        # Quando timer termina:
        if valve still 'closed':  # reconfirma
            switch.bomba.turn_off()
            timer.pump_safety_timeout.cancel()
            
            # Inicia cooldown
            timer.pump_anti_cycle_cooldown.start(
                duration=input_number.pump_cooldown_minutes * 60
            )
```

### Detecção de Funcionamento Real

```yaml
# binary_sensor.bomba_funcionando
# NÃO confiar apenas em switch.bomba == 'on'
# Usar potência real para confirmar funcionamento

power = sensor.bomba_..._potencia | float(0)
threshold_low = input_number.pump_power_threshold_low | float(40)

if power > threshold_low:
    return True  # Funcionando de verdade
else:
    return False  # Desligada ou problema
```

## CALIBRAÇÃO (DADOS REAIS)

### Análise Estatística - 143 Medições
```
Média:          50.8W
Mediana:        51W
Desvio Padrão:  3.2W
P10:            47W
P90:            54W
Mínimo:         34W
Máximo:         70W
```

### Thresholds Calibrados
```yaml
Normal (83% dos casos):  48-53W
Alerta Baixo:           <40W  # Possível entupimento/ar
Alerta Alto:            >60W  # Possível sobrecarga
```

### Valores Operacionais
```yaml
Consumo nominal:     50W
Custo por hora:      R$ 0,043 (tarifa R$ 0,85/kWh)
Tensão:              220V
Corrente:            0.25A
```

## PROBLEMAS CONHECIDOS + SOLUÇÕES

### 1. Bomba não liga automaticamente

**Debug:**
```yaml
# Developer Tools → Template
Fluxo: {{ states('valve.hot_water') }}
Override: {{ states('input_boolean.pump_manual_override') }}
Ciclos: {{ states('counter.pump_hourly_cycles') }}/{{ states('input_number.pump_max_hourly_cycles') }}
Cooldown: {{ states('timer.pump_anti_cycle_cooldown') }}
Automação: {{ states('automation.bomba_agua_quente_controle_principal') }}
```

**Soluções:**
- Verificar override desativado
- Aguardar cooldown terminar
- Verificar limite ciclos não atingido
- Confirmar automação habilitada

### 2. Bomba não desliga

**Ações:**
1. Usar `script.pump_emergency_reset`
2. Verificar estado `valve.hot_water`
3. Verificar timer `pump_safety_timeout`
4. Revisar logbook/logs

### 3. Alertas potência anormal

**Debug:**
```yaml
Potência: {{ states('sensor.bomba_de_circulacao_de_agua_quente_potencia') }}W
Baixo: {{ states('input_number.pump_power_threshold_low') }}W
Alto: {{ states('input_number.pump_power_threshold_high') }}W
```

**Causas:**
- Baixa (<40W): entupimento, ar na tubulação
- Alta (>60W): sobrecarga, problema motor

### 4. Contadores não resetam

**Verificar:**
- Automações de reset habilitadas
- Timezone correto no HA
- Executar reset manual:
```yaml
service: counter.reset
target:
  entity_id: counter.pump_hourly_cycles
```

## TEMPLATES DE DEBUGGING

### Verificação Completa
```yaml
# Developer Tools → Template
SISTEMA BOMBA - DIAGNÓSTICO COMPLETO

HARDWARE:
- Fluxo detectado: {{ states('valve.hot_water') }}
- Bomba (switch): {{ states('switch.bomba_de_circulacao_de_agua_quente') }}
- Potência: {{ states('sensor.bomba_de_circulacao_de_agua_quente_potencia') }}W

STATUS:
- Bomba Status: {{ states('sensor.bomba_status') }}
- Sistema Status: {{ states('sensor.bomba_sistema_status') }}
- Funcionando (real): {{ states('binary_sensor.bomba_funcionando') }}
- Modo Seguro: {{ states('binary_sensor.bomba_modo_seguro') }}

PROTEÇÕES:
- Override: {{ states('input_boolean.pump_manual_override') }}
- Ciclos hora: {{ states('counter.pump_hourly_cycles') }}/{{ states('input_number.pump_max_hourly_cycles') }}
- Ciclos dia: {{ states('counter.pump_daily_cycles') }}
- Timeout events: {{ states('counter.pump_timeout_events') }}

TIMERS:
- Activation delay: {{ states('timer.pump_activation_delay') }}
- Deactivation delay: {{ states('timer.pump_deactivation_delay') }}
- Safety timeout: {{ states('timer.pump_safety_timeout') }}
- Cooldown: {{ states('timer.pump_anti_cycle_cooldown') }}

ESTATÍSTICAS:
- Runtime hoje: {{ states('sensor.bomba_runtime_hoje') }}h
- Eficiência: {{ states('sensor.bomba_eficiencia') }} min/ciclo
- Custo hoje: R$ {{ states('sensor.bomba_custo_hoje') }}
```

### Teste de Helpers
```yaml
HELPERS EXISTEM?
{% for helper in ['pump_manual_override', 'pump_manual_control', 'pump_alerts_enabled'] %}
- input_boolean.{{ helper }}: {{ states('input_boolean.' + helper) | default('❌ NÃO EXISTE') }}
{% endfor %}

{% for counter in ['pump_hourly_cycles', 'pump_daily_cycles', 'pump_timeout_events', 'pump_power_alerts'] %}
- counter.{{ counter }}: {{ states('counter.' + counter) | default('❌ NÃO EXISTE') }}
{% endfor %}
```

## MODIFICAÇÕES COMUNS

### Ajustar Delays
```yaml
# Via UI: Settings → Helpers → Input Number
# Ou via service call:

service: input_number.set_value
target:
  entity_id: input_number.pump_activation_delay_seconds
data:
  value: 5  # novo valor em segundos
```

### Alterar Thresholds Potência
```yaml
# Baseado em novas medições
service: input_number.set_value
target:
  entity_id: input_number.pump_power_normal_min
data:
  value: 45  # novo mínimo normal
```

### Adicionar Nova Notificação

**Adicionar em `config/automations.yaml`:**
```yaml
- alias: "Bomba - Notificação Customizada"
  trigger:
    - platform: state
      entity_id: binary_sensor.bomba_potencia_anormal
      to: "on"
      for: "00:05:00"  # 5min sustentado
  condition:
    - condition: state
      entity_id: input_boolean.pump_alerts_enabled
      state: "on"
  action:
    - service: notify.mobile_app_seu_dispositivo
      data:
        title: "⚠️ Alerta Bomba"
        message: "Potência anormal por 5min: {{ states('sensor.bomba_de_circulacao_de_agua_quente_potencia') }}W"
```

## INSTRUÇÕES REGENERAÇÃO COMPLETA

### 1. Criar Helpers (via UI)
```bash
Settings → Devices & Services → Helpers → Create Helper
```
Seguir especificações da seção "ENTIDADES CRÍTICAS" acima.

Ou criar via `config/helpers.yaml`:
```yaml
# Ver arquivo config/helpers.yaml para referência completa
```

### 2. Adicionar Template Sensors
Editar `config/configuration.yaml`:
```yaml
template:
  - sensor:
      # [COPIAR de docs/documentacao-completa-bomba.md seção 4.3]
  - binary_sensor:
      # [COPIAR de docs/documentacao-completa-bomba.md seção 4.3]
```

### 3. Adicionar Utility Meters
Ainda em `config/configuration.yaml`:
```yaml
utility_meter:
  pump_energy_daily:
    source: sensor.bomba_de_circulacao_de_agua_quente_summation_delivered
    cycle: daily
  pump_energy_weekly:
    source: sensor.bomba_de_circulacao_de_agua_quente_summation_delivered
    cycle: weekly
  pump_energy_monthly:
    source: sensor.bomba_de_circulacao_de_agua_quente_summation_delivered
    cycle: monthly
```

### 4. Adicionar Automações
Copiar conteúdo de `config/automations.yaml` para o seu `automations.yaml`.

### 5. Adicionar Scripts
Copiar conteúdo de `config/scripts.yaml` para o seu `scripts.yaml`.

### 6. Validar e Reiniciar
```bash
# Developer Tools → YAML → Check Configuration
# Se OK: Settings → System → Restart Home Assistant
```

### 7. Testar Sistema
```yaml
# Services → script.pump_system_test → Execute
# Verificar logs: Settings → System → Logs
```

## DEPENDÊNCIAS

### Home Assistant
- Versão: 2023.x ou superior
- Integrações necessárias:
  - eWeLink (para Sonoff Mini)
  - ZHA ou Zigbee2MQTT (para Tuya)

### Hardware
- Sonoff Mini (WiFi 2.4GHz)
- Tuya TS011F (Zigbee)
- Coordinator Zigbee (ConBee II/Sonoff)
- Bomba 50W 220V

### Rede
- WiFi 2.4GHz estável
- Cobertura Zigbee adequada

### Opcional
- Mobile App para notificações
- Recorder/History para gráficos

## ESTRUTURA DE ARQUIVOS

```
bomba-agua-quente/
├── CLAUDE_CONTEXT.md              # Este arquivo (IA-friendly)
├── README.md                      # Overview para humanos
├── CHANGELOG.md                   # Histórico de versões
├── config/
│   ├── automations.yaml           # 6 automações
│   ├── scripts.yaml               # 3 scripts
│   ├── configuration.yaml         # template sensors + utility_meters
│   └── helpers.yaml               # Referência criação helpers
├── docs/
│   └── documentacao-completa-bomba.md  # Doc detalhada original
└── tests/
    └── system_test.yaml           # Testes automatizados
```

## CHANGELOG RESUMIDO

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 01/2024 | Implementação inicial valores fixos |
| 2.0 | 01/2024 | Valores configuráveis via input_numbers |
| 2.0.1 | 01/2024 | Dashboard otimizado iPad |

Ver [CHANGELOG.md](CHANGELOG.md) para detalhes completos.

---

**ÚLTIMA ATUALIZAÇÃO:** {{now().strftime('%d/%m/%Y %H:%M')}}  
**BASEADO EM:** docs/documentacao-completa-bomba.md  
**ESTRUTURA:** Padrão definido nas instruções do Claude
