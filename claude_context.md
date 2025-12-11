# CLAUDE_CONTEXT.md v3.0 - Sistema Automação Bomba Água Quente

## RESUMO EXECUTIVO
Sistema Home Assistant que aciona automaticamente bomba de circulação (68W real) ao detectar fluxo em torneira de água quente via Sonoff Mini, utilizando relé Zigbee Tuya TS011F. **v3.0 migra monitoramento de potência → corrente** para detecção precisa de emperramento/desgaste. Inclui múltiplas camadas de proteção (timeout 30min, cooldown 3min, limite 20 ciclos/hora), delays configuráveis (3s liga/20s desliga) e monitoramento de corrente calibrado (normal: 0.303-0.323A baseado em 1365 medições reais).

## MUDANÇAS v2.0 → v3.0

### **Melhorias Principais**
1. ✅ **Monitoramento por corrente** (detecção instantânea de emperramento)
2. ✅ **Cálculo correto de consumo** (Statistics + History Stats)
3. ✅ **Thresholds calibrados** (dados reais de 1365 medições)
4. ✅ **Novo script de calibração** (ajuste automático de limites)

### **Entidades Alteradas**
- **Removidas:** 4 input_numbers de potência + 1 counter
- **Adicionadas:** 4 input_numbers de corrente + 1 counter + 6 sensors
- **Total:** 22 → 28 entidades

## ENTIDADES CRÍTICAS v3.0

### Hardware (DADOS REAIS)
```yaml
# Detecção de fluxo
valve.hot_water  # Sonoff Mini (eWeLink)

# Controle da bomba
switch.bomba_de_circulacao_de_agua_quente  # Tuya TS011F (Zigbee)

# Monitoramento (NOVO EM v3.0: corrente prioritária)
sensor.bomba_de_circulacao_de_agua_quente_corrente  # Amperes em tempo real ⭐
sensor.bomba_de_circulacao_de_agua_quente_potencia  # Watts (mantido para referência)
```

### Helpers (23 ENTIDADES - +1 vs v2.0)

**Timers (4) - SEM MUDANÇAS:**
```yaml
timer.pump_activation_delay         # 00:00:30 - Delay para ligar
timer.pump_deactivation_delay       # 00:01:00 - Delay para desligar
timer.pump_safety_timeout           # 01:00:00 - Proteção máxima
timer.pump_anti_cycle_cooldown      # 00:10:00 - Cooldown entre ciclos
```

**Input Booleans (3) - SEM MUDANÇAS:**
```yaml
input_boolean.pump_manual_override   # Desabilita automação
input_boolean.pump_manual_control    # Controle manual ON/OFF
input_boolean.pump_alerts_enabled    # Habilita notificações
```

**Counters (4) - 1 ALTERADO:**
```yaml
counter.pump_hourly_cycles          # Reset automático a cada hora (max: 50)
counter.pump_daily_cycles           # Reset 00:00 (max: 999)
counter.pump_timeout_events         # Histórico total timeouts (max: 9999)
counter.pump_current_alerts         # ⭐ NOVO: Histórico alertas corrente (max: 9999)
```

**Input DateTime (2) - SEM MUDANÇAS:**
```yaml
input_datetime.pump_last_activation  # Timestamp última ativação
input_datetime.pump_last_timeout     # Timestamp último timeout
```

**Input Numbers (9) - 4 NOVOS (corrente) vs 4 REMOVIDOS (potência):**
```yaml
# ⭐ Thresholds Corrente (NOVOS - calibrados com dados reais)
input_number.pump_current_threshold_low      # 0.15-0.30A, step 0.01, default 0.24
input_number.pump_current_threshold_high     # 0.30-0.50A, step 0.01, default 0.33
input_number.pump_current_normal_min         # 0.25-0.35A, step 0.001, default 0.303
input_number.pump_current_normal_max         # 0.25-0.35A, step 0.001, default 0.323

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

### Template Sensors (12 - +5 vs v2.0)

**Sensores de Estado:**
```yaml
sensor.bomba_status                # Funcionando/Ligada sem carga/Desligada (corrente)
sensor.bomba_sistema_status        # Manual/Limite próximo/Funcionando/Ativo/Standby
sensor.bomba_runtime_hoje          # ⭐ NOVO: History stats preciso (horas funcionamento)
sensor.bomba_eficiencia            # Minutos por ciclo (runtime/ciclos)

# ⭐ Energia (NOVOS - cálculo correto)
sensor.bomba_energia_hoje          # kWh hoje (runtime × 0.068kW)
sensor.bomba_energia_semanal       # kWh 7 dias
sensor.bomba_energia_mensal        # kWh 30 dias

# ⭐ Custos (NOVOS)
sensor.bomba_custo_hoje            # R$ hoje
sensor.bomba_custo_semanal         # R$ semana
sensor.bomba_custo_mensal          # R$ mês

# ⭐ Corrente/Potência (NOVOS)
sensor.bomba_potencia_calculada    # V × I (W)
```

**Sensores Binários:**
```yaml
binary_sensor.bomba_funcionando    # ⭐ ATUALIZADO: corrente > threshold_low
binary_sensor.fluxo_agua_quente    # valve == 'open'
binary_sensor.bomba_modo_seguro    # cycles < max AND override == 'off'
binary_sensor.bomba_corrente_anormal # ⭐ NOVO: corrente fora dos limites quando ligada
```

### Statistics Sensors (3 NOVOS)
```yaml
sensor.bomba_corrente_media        # Média móvel 24h
sensor.bomba_corrente_maxima       # Máximo 24h
sensor.bomba_corrente_minima       # Mínimo 24h
```

### Automações (7 - +1 vs v2.0)

```yaml
bomba_agua_quente_controle_principal    # ⭐ ATUALIZADO: usa corrente
bomba_timeout_seguranca                 # Proteção timeout
bomba_controle_manual                   # Override manual
bomba_alerta_corrente_anormal           # ⭐ NOVO: alerta corrente fora de limites
bomba_reset_contador_horario            # Reset a cada hora
bomba_reset_contador_diario             # Reset à meia-noite
bomba_relatorio_diario                  # ⭐ ATUALIZADO: inclui métricas corrente
```

### Scripts (4 - +1 vs v2.0)

```yaml
pump_system_test                    # ⭐ ATUALIZADO: mede corrente
pump_emergency_reset                # Reset emergência
pump_detailed_report                # ⭐ ATUALIZADO: relatório com corrente
pump_calibrate_current              # ⭐ NOVO: calibração automática de thresholds
```

## ARQUITETURA DO SISTEMA v3.0

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
[sensor...corrente] → 0.303-0.323A (monitoramento) ⭐
[sensor...potencia_calculada] → 68W (V × I)
    ↓
[valve.hot_water] → closed
    ↓
[timer.pump_deactivation_delay] → 20s
    ↓
[switch.bomba_...] → OFF
    ↓
[timer.pump_anti_cycle_cooldown] → 3min
```

### Por Que Corrente é Melhor que Potência?

| Aspecto | Potência (v2.0) | Corrente (v3.0) |
|---------|-----------------|-----------------|
| **Detecção emperramento** | Tardia (problema avançado) | ⚡ Instantânea |
| **Sensibilidade** | Baixa | Alta |
| **Desgaste progressivo** | Mascarado | 📈 Tendência clara |
| **Precisão diagnóstico** | Média | ✅ Excelente |

**Exemplo prático:**
```
Emperramento: corrente ↑ 20% → detecção IMEDIATA
              potência ↑ 5% → detecção tardia
```

## CALIBRAÇÃO (DADOS REAIS v3.0)

### Análise Estatística - 1365 Medições
```
Média:          0.309A
Mediana:        0.317A
Desvio Padrão:  0.039A (12.6% - comportamento estável ✅)
P10:            0.303A
P90:            0.323A
Mínimo:         0.011A (bomba desligada)
Máximo:         0.337A
Outliers:       0 (sem picos anormais ✅)
```

### Thresholds Calibrados
```yaml
🟢 Normal:       0.303 - 0.323A (P10-P90, 80% dos dados)
🟡 Baixo:        < 0.242A (motor girando vazio/ar)
🟠 Alto:         > 0.323A (início sobrecarga)
🔴 Crítico:      > 0.330A (P99 - desligar imediatamente)
🚨 Emergência:   > 0.350A (emperramento detectado)
```

### Valores Operacionais
```yaml
Corrente nominal:    0.309A (média real)
Tensão:              220V
Potência real:       68W (0.309A × 220V)
Potência nominal:    50W (rating do fabricante)
Fator potência:      ~0.75 (68W real / 90VA aparente)
Custo por hora:      R$ 0.058 (0.068kW × R$0.85/kWh)
```

## LÓGICA DE FUNCIONAMENTO v3.0

### Detecção de Funcionamento Real

```yaml
# binary_sensor.bomba_funcionando
# v2.0: Usava potência > 40W (impreciso)
# v3.0: Usa corrente > 0.24A (preciso) ⭐

current = sensor.bomba_..._corrente | float(0)
threshold_low = input_number.pump_current_threshold_low | float(0.24)

if current > threshold_low:
    return True  # Funcionando de verdade
else:
    return False  # Desligada ou problema
```

### Detecção de Anomalias

```yaml
# binary_sensor.bomba_corrente_anormal (NOVO v3.0)

current = sensor.bomba_..._corrente | float(0)
low = input_number.pump_current_threshold_low | float(0.24)
high = input_number.pump_current_threshold_high | float(0.33)
pump_on = switch.bomba == 'on'

if pump_on AND (current < low OR current > high):
    return True  # ⚠️ Anomalia detectada
    
    # Causas possíveis:
    # < 0.24A: ar na tubulação, rotor travado, problema elétrico
    # > 0.33A: emperramento, sobrecarga, desgaste rolamentos
```

### Cálculo de Consumo (CORRIGIDO v3.0)

```yaml
# v2.0: utility_meter (bugado com summation_delivered)
# v3.0: History Stats + cálculo direto ⭐

# 1. Runtime preciso (history_stats)
sensor.bomba_runtime_hoje:
  entity_id: binary_sensor.bomba_funcionando
  state: 'on'
  type: time
  # Resultado: 3.5h (exemplo)

# 2. Energia calculada
sensor.bomba_energia_hoje:
  runtime = 3.5h
  power_kw = 0.068  # 68W real
  energy = runtime × power_kw = 0.238 kWh ✅

# 3. Custo
sensor.bomba_custo_hoje:
  energy = 0.238 kWh
  tariff = 0.85 R$/kWh
  cost = energy × tariff = R$ 0.20 ✅
```

## SCRIPTS NOVOS v3.0

### pump_calibrate_current (NOVO)

```yaml
# Propósito: Calibração automática de thresholds
# Uso: Após instalação ou manutenção

Sequência:
1. Liga bomba por 30s
2. Coleta statistics (média, min, max)
3. Calcula thresholds recomendados:
   - Normal min = média × 0.95
   - Normal max = média × 1.05
   - Alerta baixo = média × 0.80
   - Alerta alto = máximo × 1.10
4. Exibe recomendação para ajuste manual
```

## PROBLEMAS CONHECIDOS + SOLUÇÕES v3.0

### 1. Bomba não liga automaticamente

**Debug atualizado:**
```yaml
# Developer Tools → Template
Fluxo: {{ states('valve.hot_water') }}
Override: {{ states('input_boolean.pump_manual_override') }}
Ciclos: {{ states('counter.pump_hourly_cycles') }}/{{ states('input_number.pump_max_hourly_cycles') }}
Cooldown: {{ states('timer.pump_anti_cycle_cooldown') }}
Automação: {{ states('automation.bomba_agua_quente_controle_principal') }}
Corrente: {{ states('sensor.bomba_de_circulacao_de_agua_quente_corrente') }}A ⭐
```

### 2. Alertas de corrente anormal

**Debug:**
```yaml
Corrente: {{ states('sensor.bomba_de_circulacao_de_agua_quente_corrente') }}A
Baixo: {{ states('input_number.pump_current_threshold_low') }}A
Alto: {{ states('input_number.pump_current_threshold_high') }}A
Média 24h: {{ states('sensor.bomba_corrente_media') }}A
```

**Interpretação:**
| Corrente | Diagnóstico | Ação |
|----------|-------------|------|
| < 0.24A | Motor vazio, ar, problema elétrico | Purgar sistema, verificar conexões |
| 0.24-0.303A | Abaixo do normal, possível desgaste | Monitorar tendência |
| 0.303-0.323A | ✅ Normal | - |
| 0.323-0.33A | Acima do normal, possível sobrecarga | Verificar tubulação |
| > 0.33A | ⚠️ Crítico - emperramento | Desligar, manutenção urgente |

### 3. Consumo não bate com conta de luz

**Verificação v3.0:**
```yaml
# Calcular manualmente:
Runtime hoje: {{ states('sensor.bomba_runtime_hoje') }}h
Potência: 0.068kW
Energia: runtime × 0.068 = X kWh
Custo: X × R$ 0.85 = R$ Y

# Comparar com sensor:
Energia sensor: {{ states('sensor.bomba_energia_hoje') }} kWh
Custo sensor: R$ {{ states('sensor.bomba_custo_hoje') }}

# Se divergir > 10%, executar:
script.pump_calibrate_current
```

## MIGRAÇÃO v2.0 → v3.0

### Checklist Completo

**Antes de começar:**
```bash
# Criar branch git
git checkout -b v3.0-corrente-migration
git add -A
git commit -m "v2.0: Snapshot antes migração"
```

**Passo 1: Backup v2.0**
```yaml
# Copiar arquivos atuais:
- configuration.yaml → versions/v2.0/
- automations.yaml → versions/v2.0/
- scripts.yaml → versions/v2.0/
```

**Passo 2: Criar novos helpers**
```yaml
# Settings → Helpers → Create Helper
✅ input_number.pump_current_threshold_low (0.24A)
✅ input_number.pump_current_threshold_high (0.33A)
✅ input_number.pump_current_normal_min (0.303A)
✅ input_number.pump_current_normal_max (0.323A)
✅ counter.pump_current_alerts
```

**Passo 3: Atualizar configuration.yaml**
```bash
# Substituir seção template: por v3.0
# Adicionar history_stats
# Adicionar statistics sensors
```

**Passo 4: Atualizar automations.yaml**
```bash
# Substituir arquivo completo por v3.0
```

**Passo 5: Atualizar scripts.yaml**
```bash
# Substituir arquivo completo por v3.0
```

**Passo 6: Restart Home Assistant**
```yaml
# Developer Tools → YAML → Restart
```

**Passo 7: Validar**
```yaml
# Developer Tools → Template
# Executar validação de helpers (ver helpers.yaml)
# Testar: script.pump_system_test
```

**Passo 8: Calibrar**
```yaml
# Executar: script.pump_calibrate_current
# Ajustar thresholds se necessário
```

**Passo 9: Remover helpers antigos**
```yaml
# Settings → Helpers → Delete:
❌ input_number.pump_power_threshold_low
❌ input_number.pump_power_threshold_high
❌ input_number.pump_power_normal_min
❌ input_number.pump_power_normal_max
❌ counter.pump_power_alerts
```

**Passo 10: Git merge**
```bash
git add -A
git commit -m "v3.0: Migração completa para corrente"
git checkout main
git merge v3.0-corrente-migration
```

## DEPENDÊNCIAS v3.0

### Home Assistant
- Versão: 2023.x ou superior
- Integrações necessárias:
  - eWeLink (para Sonoff Mini)
  - ZHA ou Zigbee2MQTT (para Tuya)
  - Statistics (built-in)
  - History Stats (built-in)

### Hardware
- Sonoff Mini (WiFi 2.4GHz) - detecção fluxo
- Tuya TS011F (Zigbee) - controle + medição corrente ⭐
- Coordinator Zigbee (ConBee II/Sonoff)
- Bomba 50W 220V

**CRÍTICO:** Tuya TS011F deve expor `sensor...corrente`. Verificar antes de migrar.

---

**VERSÃO:** 3.0.0  
**DATA:** {{now().strftime('%d/%m/%Y')}}  
**BASEADO EM:** Análise de 1365 medições reais de corrente  
**ÚLTIMA ATUALIZAÇÃO:** {{now().strftime('%d/%m/%Y %H:%M')}}
