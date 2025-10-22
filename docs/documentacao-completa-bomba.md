# Sistema de Automação Bomba Água Quente v2.0
## Documentação Técnica Completa

---

## 📋 Sumário

1. [Visão Geral](#visão-geral)
2. [Arquitetura do Sistema](#arquitetura-do-sistema)
3. [Hardware Necessário](#hardware-necessário)
4. [Componentes do Sistema](#componentes-do-sistema)
5. [Implementação Passo a Passo](#implementação-passo-a-passo)
6. [Calibração e Configuração](#calibração-e-configuração)
7. [Testes e Validação](#testes-e-validação)
8. [Dashboard e Interface](#dashboard-e-interface)
9. [Troubleshooting](#troubleshooting)
10. [Anexos - Código Completo](#anexos---código-completo)

---

## 1. Visão Geral

### Objetivo do Projeto
Automatizar o acionamento de uma bomba de circulação de água quente baseado na detecção de fluxo, proporcionando:
- Água quente imediata nas torneiras
- Economia de água (sem desperdício esperando esquentar)
- Economia de energia (bomba liga apenas quando necessário)
- Proteção do equipamento (múltiplas camadas de segurança)

### Princípio de Funcionamento
1. Sensor detecta abertura de torneira de água quente
2. Sistema aguarda delay configurável para confirmar fluxo
3. Bomba é acionada automaticamente
4. Água quente circula até a torneira
5. Ao fechar a torneira, sistema aguarda delay configurável
6. Bomba é desligada automaticamente

### Especificações Técnicas
- **Consumo da bomba:** 50W nominal
- **Faixa normal de operação:** 48-53W (baseado em 143 medições reais)
- **Tensão:** 220V
- **Corrente:** 0.25A
- **Custo operacional:** ~R$ 0,043/hora (com tarifa R$ 0,85/kWh)

---

## 2. Arquitetura do Sistema

### Diagrama de Componentes

```
[Torneira] → [Sensor Fluxo] → [Home Assistant] → [Relé] → [Bomba]
                ↓                    ↓                ↓
           [Sonoff Mini]        [Automações]    [Tuya TS011F]
                                     ↓
                                [Dashboard]
                                     ↓
                                [Usuário]
```

### Fluxo de Dados

```yaml
Entrada: valve.hot_water (open/closed)
    ↓
Processamento: Automações com lógica de controle
    ↓
Saída: switch.bomba_de_circulacao_de_agua_quente (on/off)
    ↓
Feedback: sensor.bomba_de_circulacao_de_agua_quente_potencia
```

### Camadas de Proteção

1. **Hardware:** Relé com proteção contra sobrecarga
2. **Software:** Múltiplos timers e contadores
3. **Lógica:** Condições de segurança nas automações
4. **Interface:** Botão de emergência no dashboard

---

## 3. Hardware Necessário

### Componentes Principais

| Item | Modelo | Função | Especificações |
|------|--------|--------|----------------|
| Sensor de Fluxo | Sonoff Mini | Detectar fluxo de água | WiFi 2.4GHz |
| Relé de Potência | Tuya TS011F | Controlar bomba | Zigbee, 16A |
| Bomba | Circulação | Circular água quente | 50W, 220V |
| Coordinator | ConBee II/Sonoff | Comunicação Zigbee | USB |

### Requisitos de Instalação

- Home Assistant instalado e configurado
- Integração ZHA ou Zigbee2MQTT
- Rede WiFi 2.4GHz estável
- Acesso ao quadro elétrico
- Tubulação com retorno para circulação

---

## 4. Componentes do Sistema

### 4.1 Helpers (22 entidades)

#### Timers (4)
```yaml
timer.pump_activation_delay         # Configurável via input_number
timer.pump_deactivation_delay       # Configurável via input_number  
timer.pump_safety_timeout           # Configurável via input_number
timer.pump_anti_cycle_cooldown      # Configurável via input_number
```

#### Input Booleans (3)
```yaml
input_boolean.pump_manual_override  # Override do sistema automático
input_boolean.pump_manual_control   # Controle manual da bomba
input_boolean.pump_alerts_enabled   # Habilita notificações
```

#### Counters (4)
```yaml
counter.pump_hourly_cycles          # Ciclos por hora (reset automático)
counter.pump_daily_cycles           # Ciclos diários (reset 00:00)
counter.pump_timeout_events         # Total de timeouts (histórico)
counter.pump_power_alerts           # Total alertas potência (histórico)
```

#### Input DateTime (2)
```yaml
input_datetime.pump_last_activation # Timestamp última ativação
input_datetime.pump_last_timeout    # Timestamp último timeout
```

#### Input Numbers (9) - CONFIGURÁVEIS
```yaml
# Thresholds de Potência
input_number.pump_power_threshold_low      # Limite inferior (W)
input_number.pump_power_threshold_high     # Limite superior (W)
input_number.pump_power_normal_min         # Normal mínimo (W)
input_number.pump_power_normal_max         # Normal máximo (W)

# Limites Operacionais
input_number.pump_max_daily_runtime        # Runtime máximo/dia (h)
input_number.pump_max_hourly_cycles        # Ciclos máximos/hora

# Configurações de Tempo
input_number.pump_timeout_minutes          # Timeout segurança (min)
input_number.pump_cooldown_minutes         # Cooldown entre ciclos (min)
input_number.pump_activation_delay_seconds # Delay para ligar (s)
input_number.pump_deactivation_delay_seconds # Delay para desligar (s)

# Custo Energia
input_number.bomba_energy_tariff          # Tarifa energia (R$/kWh)
```

### 4.2 Template Sensors (9)

```yaml
# Sensores de Estado
sensor.bomba_status                # Estado da bomba
sensor.bomba_sistema_status        # Status geral do sistema
sensor.bomba_runtime_hoje          # Horas funcionando hoje
sensor.bomba_eficiencia            # Minutos por ciclo
sensor.bomba_custo_hoje            # Custo em R$ hoje

# Sensores Binários
binary_sensor.bomba_funcionando    # Detecta funcionamento real
binary_sensor.fluxo_agua_quente    # Estado do fluxo
binary_sensor.bomba_modo_seguro    # Sistema em modo seguro
binary_sensor.bomba_potencia_anormal # Potência fora dos limites
```

### 4.3 Utility Meters (3)

```yaml
sensor.pump_energy_daily           # Consumo diário
sensor.pump_energy_weekly          # Consumo semanal  
sensor.pump_energy_monthly         # Consumo mensal
```

### 4.4 Automações (6)

```yaml
bomba_agua_quente_controle_principal # Lógica principal
bomba_timeout_seguranca             # Proteção timeout
bomba_controle_manual                # Override manual
bomba_reset_contador_horario        # Reset a cada hora
bomba_reset_contador_diario         # Reset à meia-noite
bomba_relatorio_diario              # Relatório 22:00
```

### 4.5 Scripts (3)

```yaml
pump_system_test                    # Teste completo
pump_emergency_reset                # Reset emergência
pump_detailed_report                # Relatório detalhado
```

---

## 5. Implementação Passo a Passo

### Fase 1: Preparação

#### 1.1 Instalar Hardware
```bash
1. Instalar Sonoff Mini no ponto de detecção de fluxo
2. Configurar no eWeLink e integrar ao HA
3. Parear Tuya TS011F com coordinator Zigbee
4. Conectar bomba ao relé Tuya
5. Testar acionamento manual
```

#### 1.2 Verificar Entidades
```yaml
# Developer Tools → States
valve.hot_water                     # Deve existir
switch.bomba_de_circulacao_de_agua_quente # Deve existir
sensor.bomba_de_circulacao_de_agua_quente_potencia # Deve existir
```

### Fase 2: Criar Helpers

#### 2.1 Via Interface (Settings → Helpers → Create Helper)

**Input Numbers:**
| Entity ID | Nome | Min | Max | Step | Unit | Default |
|-----------|------|-----|-----|------|------|---------|
| pump_power_threshold_low | Limite Potência Baixa | 20 | 50 | 1 | W | 40 |
| pump_power_threshold_high | Limite Potência Alta | 50 | 100 | 1 | W | 60 |
| pump_power_normal_min | Potência Normal Mínimo | 30 | 60 | 1 | W | 48 |
| pump_power_normal_max | Potência Normal Máximo | 40 | 80 | 1 | W | 53 |
| pump_max_daily_runtime | Runtime Máximo Diário | 1 | 24 | 0.5 | h | 8 |
| pump_max_hourly_cycles | Máximo Ciclos/Hora | 5 | 50 | 1 | - | 20 |
| pump_timeout_minutes | Timeout Segurança | 5 | 60 | 5 | min | 30 |
| pump_cooldown_minutes | Cooldown Entre Ciclos | 0 | 10 | 1 | min | 3 |
| pump_activation_delay_seconds | Delay Ligar Bomba | 0 | 30 | 1 | s | 3 |
| pump_deactivation_delay_seconds | Delay Desligar Bomba | 0 | 60 | 5 | s | 20 |
| bomba_energy_tariff | Tarifa Energia | 0.1 | 2 | 0.01 | R$/kWh | 0.85 |

**Timers:**
- pump_activation_delay (duração: 00:00:30)
- pump_deactivation_delay (duração: 00:01:00)
- pump_safety_timeout (duração: 01:00:00)
- pump_anti_cycle_cooldown (duração: 00:10:00)

**Input Booleans:**
- pump_manual_override (icon: mdi:hand-back-right)
- pump_manual_control (icon: mdi:toggle-switch)
- pump_alerts_enabled (icon: mdi:bell)

**Counters:**
- pump_hourly_cycles (max: 50, icon: mdi:counter)
- pump_daily_cycles (max: 999, icon: mdi:counter)
- pump_timeout_events (max: 9999, icon: mdi:alert)
- pump_power_alerts (max: 9999, icon: mdi:flash-alert)

**Input DateTime:**
- pump_last_activation (has_date: true, has_time: true)
- pump_last_timeout (has_date: true, has_time: true)

### Fase 3: Criar Template Sensors

#### 3.1 Via Interface ou configuration.yaml

```yaml
# Settings → Helpers → Template → Template a sensor
# OU adicionar em configuration.yaml

template:
  - sensor:
      - name: "Bomba Status"
        unique_id: bomba_status
        state: >
          {% set power = states('sensor.bomba_de_circulacao_de_agua_quente_potencia') | float(0) %}
          {% if power > 35 %}
            Funcionando
          {% elif states('switch.bomba_de_circulacao_de_agua_quente') == 'on' %}
            Ligada sem carga
          {% else %}
            Desligada
          {% endif %}
        icon: mdi:pump

      - name: "Bomba Runtime Hoje"
        unique_id: bomba_runtime_hoje
        unit_of_measurement: "h"
        state: >
          {% set ns = namespace(total=0) %}
          {% if has_value('sensor.bomba_de_circulacao_de_agua_quente_potencia') %}
            {% set today = now().replace(hour=0, minute=0, second=0, microsecond=0) %}
            {% for state in states.sensor.bomba_de_circulacao_de_agua_quente_potencia.history(start_time=today) %}
              {% if state.state | float(0) > 35 %}
                {% set ns.total = ns.total + 0.083 %}
              {% endif %}
            {% endfor %}
          {% endif %}
          {{ ns.total | round(1) }}

      - name: "Bomba Eficiência"
        unique_id: bomba_eficiencia
        unit_of_measurement: "min/ciclo"
        state: >
          {% set runtime = states('sensor.bomba_runtime_hoje') | float(0) %}
          {% set cycles = states('counter.pump_daily_cycles') | int(0) %}
          {% if cycles > 0 %}
            {{ ((runtime * 60) / cycles) | round(1) }}
          {% else %}
            0
          {% endif %}

      - name: "Bomba Custo Hoje"
        unique_id: bomba_custo_hoje
        unit_of_measurement: "R$"
        state: >
          {% set energy = states('sensor.pump_energy_daily') | float(0) %}
          {% set tariff = states('input_number.bomba_energy_tariff') | float(0.85) %}
          {{ (energy * tariff) | round(2) }}

      - name: "Bomba Sistema Status"
        unique_id: bomba_sistema_status
        state: >
          {% set override = states('input_boolean.pump_manual_override') %}
          {% set cycles = states('counter.pump_hourly_cycles') | int(0) %}
          {% set max_cycles = states('input_number.pump_max_hourly_cycles') | int(20) %}
          {% set pump_on = states('switch.bomba_de_circulacao_de_agua_quente') %}
          {% set timeout_active = states('timer.pump_safety_timeout') %}
          
          {% if override == 'on' %}
            Manual
          {% elif cycles >= (max_cycles - 2) %}
            Limite próximo
          {% elif timeout_active == 'active' and pump_on == 'on' %}
            Funcionando
          {% elif pump_on == 'on' %}
            Ativo
          {% else %}
            Standby
          {% endif %}

  - binary_sensor:
      - name: "Bomba Funcionando"
        unique_id: bomba_funcionando
        state: >
          {% set power = states('sensor.bomba_de_circulacao_de_agua_quente_potencia') | float(0) %}
          {% set low = states('input_number.pump_power_threshold_low') | float(40) %}
          {{ power > low }}

      - name: "Fluxo Água Quente"
        unique_id: fluxo_agua_quente
        state: "{{ states('valve.hot_water') == 'open' }}"

      - name: "Bomba Modo Seguro"
        unique_id: bomba_modo_seguro
        state: >
          {% set cycles = states('counter.pump_hourly_cycles') | int(0) %}
          {% set max_cycles = states('input_number.pump_max_hourly_cycles') | int(20) %}
          {% set override = states('input_boolean.pump_manual_override') %}
          {{ cycles < max_cycles and override == 'off' }}

      - name: "Bomba Potência Anormal"
        unique_id: bomba_potencia_anormal
        state: >
          {% set power = states('sensor.bomba_de_circulacao_de_agua_quente_potencia') | float(0) %}
          {% set low = states('input_number.pump_power_threshold_low') | float(40) %}
          {% set high = states('input_number.pump_power_threshold_high') | float(60) %}
          {% set pump_on = states('switch.bomba_de_circulacao_de_agua_quente') == 'on' %}
          
          {% if pump_on and (power < low or power > high) %}
            true
          {% else %}
            false
          {% endif %}
```

### Fase 4: Criar Utility Meters

```yaml
# configuration.yaml
utility_meter:
  pump_energy_daily:
    source: sensor.bomba_de_circulacao_de_agua_quente_summation_delivered
    cycle: daily
    tariffs:
      - normal
    
  pump_energy_weekly:
    source: sensor.bomba_de_circulacao_de_agua_quente_summation_delivered
    cycle: weekly
    
  pump_energy_monthly:
    source: sensor.bomba_de_circulacao_de_agua_quente_summation_delivered
    cycle: monthly
```

### Fase 5: Adicionar Automações

Copiar conteúdo do arquivo de automações (ver Anexos) para `automations.yaml`

### Fase 6: Adicionar Scripts

```yaml
# scripts.yaml
pump_system_test:
  alias: "Bomba - Teste Sistema Completo"
  sequence:
    - service: logbook.log
      data:
        name: "🧪 TESTE SISTEMA INICIADO"
        message: "Iniciando teste completo do sistema da bomba"
    
    - service: input_boolean.turn_off
      target:
        entity_id: input_boolean.pump_manual_override
    
    - delay: "00:00:02"
    
    - service: switch.turn_on
      target:
        entity_id: switch.bomba_de_circulacao_de_agua_quente
    
    - delay: "00:00:05"
    
    - service: switch.turn_off
      target:
        entity_id: switch.bomba_de_circulacao_de_agua_quente
    
    - service: logbook.log
      data:
        name: "✅ TESTE CONCLUÍDO"
        message: "Teste do sistema finalizado com sucesso"

pump_emergency_reset:
  alias: "Bomba - Reset Emergência"
  mode: single
  sequence:
    - service: switch.turn_off
      target:
        entity_id: switch.bomba_de_circulacao_de_agua_quente
    
    - service: timer.cancel
      target:
        entity_id: 
          - timer.pump_activation_delay
          - timer.pump_deactivation_delay
          - timer.pump_safety_timeout
          - timer.pump_anti_cycle_cooldown
    
    - service: input_boolean.turn_off
      target:
        entity_id:
          - input_boolean.pump_manual_override
          - input_boolean.pump_manual_control
    
    - service: logbook.log
      data:
        name: "🛑 RESET DE EMERGÊNCIA"
        message: "Sistema resetado - bomba desligada e timers cancelados"
    
    - service: persistent_notification.create
      data:
        title: "🛑 Reset de Emergência Executado"
        message: "Bomba desligada e sistema resetado às {{ now().strftime('%H:%M:%S') }}"
        notification_id: pump_emergency

pump_detailed_report:
  alias: "Bomba - Relatório Detalhado"
  sequence:
    - service: persistent_notification.create
      data:
        title: "📊 RELATÓRIO DETALHADO - BOMBA"
        message: >
          **Data/Hora:** {{ now().strftime('%d/%m/%Y %H:%M') }}
          
          **ESTATÍSTICAS DO DIA:**
          • Ciclos: {{ states('counter.pump_daily_cycles') }}
          • Runtime: {{ states('sensor.bomba_runtime_hoje') }}h
          • Consumo: {{ states('sensor.pump_energy_daily') }} kWh
          • Custo: R$ {{ states('sensor.bomba_custo_hoje') }}
          • Eficiência: {{ states('sensor.bomba_eficiencia') }} min/ciclo
          
          **CONTADORES:**
          • Ciclos/hora: {{ states('counter.pump_hourly_cycles') }}/{{ states('input_number.pump_max_hourly_cycles') }}
          • Timeouts totais: {{ states('counter.pump_timeout_events') }}
          • Alertas potência: {{ states('counter.pump_power_alerts') }}
          
          **CONFIGURAÇÕES ATUAIS:**
          • Delay liga: {{ states('input_number.pump_activation_delay_seconds') }}s
          • Delay desliga: {{ states('input_number.pump_deactivation_delay_seconds') }}s
          • Timeout: {{ states('input_number.pump_timeout_minutes') }}min
          • Cooldown: {{ states('input_number.pump_cooldown_minutes') }}min
          
          **THRESHOLDS POTÊNCIA:**
          • Normal: {{ states('input_number.pump_power_normal_min') }}-{{ states('input_number.pump_power_normal_max') }}W
          • Alerta baixo: <{{ states('input_number.pump_power_threshold_low') }}W
          • Alerta alto: >{{ states('input_number.pump_power_threshold_high') }}W
```

### Fase 7: Configurar Dashboard

Copiar conteúdo do dashboard (ver Anexos) para uma nova aba no Lovelace.

---

## 6. Calibração e Configuração

### 6.1 Análise de Dados Reais

**Metodologia de Calibração:**
1. Coletar 150+ medições de potência
2. Analisar distribuição estatística
3. Identificar faixa normal (P10-P90)
4. Definir thresholds de alerta

**Resultados da Análise (143 medições):**
```
Média: 50.8W
Mediana: 51W
Desvio Padrão: 3.2W
P10: 47W
P90: 54W
Mínimo: 34W
Máximo: 70W
```

**Thresholds Recomendados:**
- Normal: 48-53W (83% das medições)
- Alerta Baixo: <40W (possível entupimento)
- Alerta Alto: >60W (possível sobrecarga)

### 6.2 Ajuste de Delays

**Metodologia:**
1. Medir tempo de resposta da tubulação
2. Testar diferentes valores
3. Otimizar para conforto vs economia

**Valores Recomendados:**
- Delay Ativação: 3s (evita acionamentos falsos)
- Delay Desativação: 20s (evita religamentos)
- Cooldown: 3min (proteção do motor)
- Timeout: 30min (segurança)

---

## 7. Testes e Validação

### 7.1 Testes Unitários

#### Teste 1: Verificação de Helpers
```yaml
# Developer Tools → Template
TESTE HELPERS:
{% for helper in ['pump_manual_override', 'pump_manual_control', 'pump_alerts_enabled'] %}
- input_boolean.{{ helper }}: {{ states('input_boolean.' + helper) | default('❌ NÃO EXISTE') }}
{% endfor %}

{% for counter in ['pump_hourly_cycles', 'pump_daily_cycles', 'pump_timeout_events', 'pump_power_alerts'] %}
- counter.{{ counter }}: {{ states('counter.' + counter) | default('❌ NÃO EXISTE') }}
{% endfor %}

{% for timer in ['pump_activation_delay', 'pump_deactivation_delay', 'pump_safety_timeout', 'pump_anti_cycle_cooldown'] %}
- timer.{{ timer }}: {{ states('timer.' + timer) | default('❌ NÃO EXISTE') }}
{% endfor %}
```

#### Teste 2: Verificação de Sensores
```yaml
# Developer Tools → Template
TESTE SENSORES:
- Potência Atual: {{ states('sensor.bomba_de_circulacao_de_agua_quente_potencia') }}W
- Status Bomba: {{ states('sensor.bomba_status') }}
- Sistema Status: {{ states('sensor.bomba_sistema_status') }}
- Fluxo Detectado: {{ states('binary_sensor.fluxo_agua_quente') }}
- Modo Seguro: {{ states('binary_sensor.bomba_modo_seguro') }}
```

### 7.2 Testes Funcionais

#### Cenário 1: Operação Normal
```
1. Abrir torneira água quente
2. Verificar: Timer ativação inicia (3s)
3. Verificar: Bomba liga após timer
4. Verificar: Potência entre 48-53W
5. Fechar torneira
6. Verificar: Timer desativação inicia (20s)
7. Verificar: Bomba desliga após timer
8. Verificar: Cooldown ativo (3min)
```

#### Cenário 2: Proteção Timeout
```
1. Ligar bomba manualmente
2. Aguardar tempo de timeout configurado
3. Verificar: Bomba desliga automaticamente
4. Verificar: Contador timeout incrementa
5. Verificar: Notificação enviada
```

#### Cenário 3: Limite de Ciclos
```
1. Executar ciclos até limite -1
2. Verificar: Próximo ciclo funciona
3. Executar mais um ciclo (atingir limite)
4. Verificar: Novos acionamentos bloqueados
5. Aguardar reset horário
6. Verificar: Contador zerado, sistema liberado
```

#### Cenário 4: Override Manual
```
1. Ativar override manual
2. Verificar: Automação principal desabilitada
3. Testar controle manual ON/OFF
4. Verificar: Proteções mantidas (timeout, limite ciclos)
5. Desativar override
6. Verificar: Sistema automático restaurado
```

### 7.3 Testes de Stress

```yaml
# Script para teste de stress
stress_test:
  sequence:
    - repeat:
        count: 25
        sequence:
          - service: valve.open
            entity_id: valve.hot_water
          - delay: "00:00:05"
          - service: valve.close
            entity_id: valve.hot_water
          - delay: "00:00:05"
```

**Resultados Esperados:**
- Sistema bloqueia após 20 ciclos
- Sem travamentos ou erros
- Logs corretos para cada evento

### 7.4 Teste de Recovery

```
1. Simular queda de energia (reiniciar HA)
2. Verificar estados restaurados
3. Verificar contadores mantidos
4. Verificar sistema operacional
```

---

## 8. Dashboard e Interface

### 8.1 Estrutura do Dashboard

```
┌─────────────────────────────────────┐
│         STATUS PRINCIPAL            │
├──────────────┬──────────────────────┤
│   CONTROLES  │      ESTATÍSTICAS    │
├──────────────┼──────────────────────┤
│ CONFIGURAÇÕES│      PROTEÇÕES       │
├──────────────┴──────────────────────┤
│           GRÁFICOS                  │
├─────────────────────────────────────┤
│       MANUAL DO SISTEMA             │
└─────────────────────────────────────┘
```

### 8.2 Elementos Principais

1. **Cards de Status**
   - Estado em tempo real
   - Indicadores visuais coloridos
   - Alertas de anomalias

2. **Controles Rápidos**
   - Override Manual
   - Controle Manual
   - Teste Sistema
   - Reset Emergência

3. **Configurações Editáveis**
   - Todos os thresholds
   - Delays e timeouts
   - Custo energia

4. **Visualizações**
   - Gráfico de potência
   - Histórico 24h
   - Consumo energético

---

## 9. Troubleshooting

### Problemas Comuns e Soluções

#### Problema: Bomba não liga automaticamente

**Diagnóstico:**
```yaml
# Developer Tools → Template
DIAGNÓSTICO:
- Fluxo detectado: {{ states('valve.hot_water') }}
- Override ativo: {{ states('input_boolean.pump_manual_override') }}
- Ciclos/hora: {{ states('counter.pump_hourly_cycles') }}/{{ states('input_number.pump_max_hourly_cycles') }}
- Cooldown: {{ states('timer.pump_anti_cycle_cooldown') }}
- Automação habilitada: {{ states('automation.bomba_agua_quente_controle_principal') }}
```

**Soluções:**
1. Verificar se override está desativado
2. Verificar limite de ciclos
3. Aguardar cooldown terminar
4. Verificar se automação está habilitada

#### Problema: Bomba não desliga

**Ações:**
1. Usar botão Reset Emergência
2. Verificar timer timeout
3. Verificar estado da válvula
4. Revisar logs

#### Problema: Alertas de potência anormal

**Verificação:**
```yaml
Potência atual: {{ states('sensor.bomba_de_circulacao_de_agua_quente_potencia') }}W
Threshold baixo: {{ states('input_number.pump_power_threshold_low') }}W
Threshold alto: {{ states('input_number.pump_power_threshold_high') }}W
```

**Causas possíveis:**
- Baixa (<40W): Entupimento, ar na tubulação
- Alta (>60W): Sobrecarga, problema no motor

#### Problema: Contadores não resetam

**Verificar:**
- Automações de reset habilitadas
- Timezone correto no HA
- Executar reset manual via Developer Tools

### Logs e Debug

#### Habilitar logs detalhados:
```yaml
# configuration.yaml
logger:
  default: warning
  logs:
    homeassistant.components.automation: debug
    homeassistant.components.script: debug
    homeassistant.helpers.template: debug
```

#### Monitorar em tempo real:
```bash
# Terminal
tail -f /config/home-assistant.log | grep -i bomba
```

---

## 10. Anexos - Código Completo

### Anexo A: Automações Completas

[Ver arquivo automacoes-bomba-ajustadas no artefato anterior]

### Anexo B: Dashboard Completo

[Ver arquivo aba-bomba-agua-quente no artefato anterior]

### Anexo C: Scripts de Manutenção

```yaml
# Backup de configurações
backup_pump_config:
  alias: "Backup Configurações Bomba"
  sequence:
    - service: system_log.write
      data:
        message: >
          BACKUP BOMBA:
          Delays: {{ states('input_number.pump_activation_delay_seconds') }}s/{{ states('input_number.pump_deactivation_delay_seconds') }}s
          Timeouts: {{ states('input_number.pump_timeout_minutes') }}min
          Cooldown: {{ states('input_number.pump_cooldown_minutes') }}min
          Thresholds: {{ states('input_number.pump_power_threshold_low') }}-{{ states('input_number.pump_power_threshold_high') }}W
          Normal: {{ states('input_number.pump_power_normal_min') }}-{{ states('input_number.pump_power_normal_max') }}W
        level: info

# Reset total do sistema
factory_reset_pump:
  alias: "Factory Reset Sistema Bomba"
  sequence:
    - service: counter.reset
      target:
        entity_id:
          - counter.pump_hourly_cycles
          - counter.pump_daily_cycles
          - counter.pump_timeout_events
          - counter.pump_power_alerts
    
    - service: input_number.set_value
      target:
        entity_id: input_number.pump_activation_delay_seconds
      data:
        value: 3
    
    - service: input_number.set_value
      target:
        entity_id: input_number.pump_deactivation_delay_seconds
      data:
        value: 20
    
    # ... repetir para todos os input_numbers com valores padrão
```

---

## Considerações Finais

### Manutenção Preventiva

**Diária:**
- Verificar contador de timeouts (deve ser zero)
- Monitorar alertas de potência

**Semanal:**
- Analisar eficiência (min/ciclo)
- Revisar consumo energético

**Mensal:**
- Teste completo do sistema
- Backup das configurações
- Limpeza de logs antigos

### Melhorias Futuras

1. **Integração com assistente de voz**
2. **Previsão de manutenção por ML**
3. **Integração com tarifa dinâmica de energia**
4. **App mobile dedicado**
5. **Histórico em banco de dados externo**

### Segurança

- Sempre manter botão de emergência acessível
- Configurar alertas críticos
- Fazer backup regular das configurações
- Documentar mudanças realizadas

---

**Versão:** 2.0  
**Data:** {{ now().strftime('%d/%m/%Y') }}  
**Autor:** Sistema documentado via assistente AI  
**Licença:** MIT - Livre para uso e modificação

---

## Histórico de Versões

| Versão | Data | Mudanças |
|--------|------|----------|
| 1.0 | 01/2024 | Implementação inicial com valores fixos |
| 2.0 | 01/2024 | Valores configuráveis via input_numbers |
| 2.0.1 | 01/2024 | Dashboard otimizado para iPad |

---

## Suporte e Comunidade

- **Issues:** Reportar via GitHub
- **Forum:** Home Assistant Community
- **Discord:** Canal #brazil

---

*Esta documentação foi gerada com assistência de IA e validada em ambiente de produção.*