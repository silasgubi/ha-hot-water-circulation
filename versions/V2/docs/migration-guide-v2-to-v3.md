# Guia de Migração v2.0 → v3.0
## Sistema Automação Bomba Água Quente

---

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Pré-requisitos](#pré-requisitos)
3. [Backup Completo](#backup-completo)
4. [Criação de Branch Git](#criação-de-branch-git)
5. [Migração Passo a Passo](#migração-passo-a-passo)
6. [Validação](#validação)
7. [Rollback (se necessário)](#rollback-se-necessário)
8. [FAQ](#faq)

---

## 1. Visão Geral

### O Que Muda?

**Antes (v2.0):**
- Monitoramento por potência (impreciso para emperramento)
- Consumo calculado por utility_meters (bugado)
- 4 thresholds de potência

**Depois (v3.0):**
- ✅ Monitoramento por corrente (detecção instantânea)
- ✅ Consumo calculado por Statistics (preciso)
- ✅ 4 thresholds de corrente calibrados
- ✅ Script de calibração automática

### Por Que Migrar?

| Problema v2.0 | Solução v3.0 |
|---------------|--------------|
| Emperramento detectado tarde | ⚡ Alerta instantâneo via corrente |
| Consumo errado | ✅ Cálculo correto via Statistics |
| Thresholds genéricos | 🎯 Calibração com 1365 medições reais |
| Desgaste invisível | 📈 Tendência clara via corrente média |

### Tempo Estimado

- **Preparação:** 10 min
- **Migração:** 20 min
- **Validação:** 15 min
- **Total:** ~45 min

---

## 2. Pré-requisitos

### Hardware

✅ **Verificar se Tuya TS011F expõe sensor de corrente:**

```yaml
# Developer Tools → States
# Buscar: sensor.bomba_de_circulacao_de_agua_quente_corrente
```

**Se não existe:**
- ❌ Tuya não suporta medição de corrente
- Alternativas: trocar hardware ou ficar na v2.0

### Software

- Home Assistant 2023.x ou superior
- Git instalado (para branching)
- Acesso a Settings → Helpers

### Conhecimento

- Saber editar `configuration.yaml`
- Saber fazer restart do HA
- Ter backup do sistema

---

## 3. Backup Completo

### Opção A: Snapshot Home Assistant

```yaml
# Settings → System → Backups → Create Backup
Nome: "Antes migração v3.0 - {{ now().strftime('%Y%m%d') }}"
```

### Opção B: Backup Manual

```bash
# Criar pasta backup
mkdir -p ~/ha-backup-v2.0

# Copiar arquivos críticos
cp config/configuration.yaml ~/ha-backup-v2.0/
cp config/automations.yaml ~/ha-backup-v2.0/
cp config/scripts.yaml ~/ha-backup-v2.0/

# Exportar helpers (via UI)
# Settings → Helpers → (3 pontinhos) → Download backup
```

---

## 4. Criação de Branch Git

**Se usa Git para config:**

```bash
# Acessar pasta config
cd /config  # ou seu caminho

# Criar branch v3.0
git checkout -b v3.0-corrente-migration

# Commitar estado atual
git add -A
git commit -m "v2.0: Snapshot antes migração corrente"

# Confirmar branch
git branch
# Deve mostrar: * v3.0-corrente-migration
```

**Se NÃO usa Git:**
- Pular esta etapa
- Confiar apenas no backup do passo 3

---

## 5. Migração Passo a Passo

### PASSO 1: Criar Novos Helpers (5 entidades)

**Via Settings → Helpers → Create Helper:**

#### 1.1 Input Number - Corrente Baixa
```yaml
Nome: Limite Corrente Baixa
ID: pump_current_threshold_low
Mínimo: 0.15
Máximo: 0.30
Step: 0.01
Valor Padrão: 0.24
Unidade: A
Icon: mdi:flash-triangle
Modo: Slider
```

#### 1.2 Input Number - Corrente Alta
```yaml
Nome: Limite Corrente Alta
ID: pump_current_threshold_high
Mínimo: 0.30
Máximo: 0.50
Step: 0.01
Valor Padrão: 0.33
Unidade: A
Icon: mdi:flash-triangle
Modo: Slider
```

#### 1.3 Input Number - Corrente Normal Mín
```yaml
Nome: Corrente Normal Mínimo
ID: pump_current_normal_min
Mínimo: 0.25
Máximo: 0.35
Step: 0.001
Valor Padrão: 0.303
Unidade: A
Icon: mdi:flash
Modo: Box
```

#### 1.4 Input Number - Corrente Normal Máx
```yaml
Nome: Corrente Normal Máximo
ID: pump_current_normal_max
Mínimo: 0.25
Máximo: 0.35
Step: 0.001
Valor Padrão: 0.323
Unidade: A
Icon: mdi:flash
Modo: Box
```

#### 1.5 Counter - Alertas Corrente
```yaml
Nome: Pump Current Alerts
ID: pump_current_alerts
Inicial: 0
Step: 1
Mínimo: 0
Máximo: 9999
Icon: mdi:flash-alert
```

**Validar criação:**
```yaml
# Developer Tools → Template
{% set new_helpers = [
  'input_number.pump_current_threshold_low',
  'input_number.pump_current_threshold_high',
  'input_number.pump_current_normal_min',
  'input_number.pump_current_normal_max',
  'counter.pump_current_alerts'
] %}

NOVOS HELPERS:
{% for h in new_helpers %}
{{ h }}: {% if states(h) != 'unknown' %}✅{% else %}❌{% endif %}
{% endfor %}
```

Todos devem mostrar ✅ antes de continuar.

---

### PASSO 2: Atualizar configuration.yaml

#### 2.1 Backup do arquivo atual
```bash
cp config/configuration.yaml config/configuration.yaml.v2.0.bak
```

#### 2.2 Substituir seção template:

**Localizar no seu `configuration.yaml`:**
```yaml
template:
  - sensor:
      # ...sensores antigos...
  - binary_sensor:
      # ...sensores antigos...
```

**Substituir por:** (usar conteúdo completo do arquivo `v3.0-configuration.yaml`)

#### 2.3 Adicionar no final (se não existir):

```yaml
# History Stats
sensor:
  - platform: history_stats
    # ...copiar do v3.0-configuration.yaml...
    
  # Statistics
  - platform: statistics
    # ...copiar do v3.0-configuration.yaml...
```

**Validar sintaxe:**
```yaml
# Developer Tools → YAML → Check Configuration
# Deve mostrar: "Configuration valid!"
```

⚠️ **NÃO reiniciar ainda!**

---

### PASSO 3: Atualizar automations.yaml

#### 3.1 Backup
```bash
cp config/automations.yaml config/automations.yaml.v2.0.bak
```

#### 3.2 Substituir arquivo completo
```bash
# Copiar conteúdo de v3.0-automations.yaml
# para seu config/automations.yaml
```

**Principais mudanças:**
- Condições de potência → corrente
- Nova automação: `bomba_alerta_corrente_anormal`
- Relatório diário atualizado

---

### PASSO 4: Atualizar scripts.yaml

#### 4.1 Backup
```bash
cp config/scripts.yaml config/scripts.yaml.v2.0.bak
```

#### 4.2 Substituir arquivo completo
```bash
# Copiar conteúdo de v3.0-scripts.yaml
# para seu config/scripts.yaml
```

**Novos scripts:**
- `pump_calibrate_current` (calibração automática)

---

### PASSO 5: Restart Home Assistant

```yaml
# Developer Tools → YAML → Restart Home Assistant
```

**Aguardar:** ~2-3 minutos para reinicialização completa.

---

### PASSO 6: Validar Sensores

#### 6.1 Template Sensors
```yaml
# Developer Tools → States
# Buscar cada um:

✅ sensor.bomba_status (deve existir)
✅ sensor.bomba_energia_hoje (novo)
✅ sensor.bomba_energia_semanal (novo)
✅ sensor.bomba_energia_mensal (novo)
✅ sensor.bomba_custo_hoje (novo)
✅ sensor.bomba_custo_semanal (novo)
✅ sensor.bomba_custo_mensal (novo)
✅ sensor.bomba_potencia_calculada (novo)
✅ binary_sensor.bomba_funcionando (atualizado)
✅ binary_sensor.bomba_corrente_anormal (novo)
```

#### 6.2 Statistics Sensors
```yaml
✅ sensor.bomba_corrente_media
✅ sensor.bomba_corrente_maxima
✅ sensor.bomba_corrente_minima
```

**Se algum mostrar "unavailable":**
- Aguardar 5-10 minutos (statistics precisa dados)
- Se persistir, revisar configuration.yaml

---

### PASSO 7: Teste Funcional

#### 7.1 Teste Automático
```yaml
# Developer Tools → Services
service: script.pump_system_test

# Verificar logs:
# Settings → System → Logs
# Buscar: "TESTE SISTEMA"
```

**Resultado esperado:**
```
✅ Bomba ligou
✅ Corrente medida: 0.30-0.32A
✅ Potência calculada: 66-70W
✅ Bomba desligou
```

#### 7.2 Teste Manual com Fluxo Real

1. Abrir torneira água quente
2. Aguardar 3s (delay ativação)
3. Verificar: bomba ligou automaticamente
4. Verificar corrente: 0.303-0.323A (normal)
5. Fechar torneira
6. Aguardar 20s (delay desativação)
7. Verificar: bomba desligou

**Debug se não funcionar:**
```yaml
# Developer Tools → Template
DIAGNÓSTICO v3.0:

Fluxo: {{ states('valve.hot_water') }}
Bomba: {{ states('switch.bomba_de_circulacao_de_agua_quente') }}
Corrente: {{ states('sensor.bomba_de_circulacao_de_agua_quente_corrente') }}A
Funcionando: {{ states('binary_sensor.bomba_funcionando') }}
Override: {{ states('input_boolean.pump_manual_override') }}
Modo seguro: {{ states('binary_sensor.bomba_modo_seguro') }}
```

---

### PASSO 8: Calibração (IMPORTANTE)

```yaml
# Developer Tools → Services
service: script.pump_calibrate_current

# Aguardar: 30 segundos

# Ver resultado em:
# Notificações → "Calibração Concluída"
```

**Ajustar thresholds conforme recomendação:**
```yaml
# Settings → Helpers
# Editar cada input_number conforme sugestão do script
```

---

### PASSO 9: Remover Helpers Antigos

**Após validar que v3.0 está 100% funcional:**

```yaml
# Settings → Helpers → Buscar e deletar:
❌ input_number.pump_power_threshold_low
❌ input_number.pump_power_threshold_high
❌ input_number.pump_power_normal_min
❌ input_number.pump_power_normal_max
❌ counter.pump_power_alerts
```

**Restart após remoção:**
```yaml
# Developer Tools → YAML → Restart
```

---

## 6. Validação

### Checklist Completo

```yaml
# Developer Tools → Template
VALIDAÇÃO v3.0:

✅ Helpers criados (5): 
{{ states('input_number.pump_current_threshold_low') != 'unknown' }}

✅ Sensores energia funcionando:
{{ states('sensor.bomba_energia_hoje') != 'unavailable' }}

✅ Statistics coletando:
{{ states('sensor.bomba_corrente_media') != 'unknown' }}

✅ Automação principal ativa:
{{ states('automation.bomba_agua_quente_controle_principal') == 'on' }}

✅ Alerta corrente configurado:
{{ states('automation.bomba_alerta_corrente_anormal') == 'on' }}

✅ Script calibração existe:
{{ states('script.pump_calibrate_current') != 'unknown' }}

✅ Teste funcional passou:
[executar pump_system_test]
```

### Teste de 24h

**Monitorar por 1 dia:**
- Ciclos funcionando normalmente?
- Corrente dentro de 0.303-0.323A?
- Consumo calculado coerente?
- Sem alertas anormais?

**Se tudo OK:**
→ Migração bem-sucedida! ✅

---

## 7. Rollback (se necessário)

### Quando fazer rollback?

- ❌ Sensor corrente não existe no Tuya
- ❌ Erros recorrentes nos logs
- ❌ Bomba não funciona após 24h de testes
- ❌ Consumo totalmente errado

### Como fazer rollback:

#### Opção A: Via Snapshot
```yaml
# Settings → System → Backups
# Restaurar: "Antes migração v3.0"
```

#### Opção B: Via Git
```bash
cd /config
git checkout main
git branch -D v3.0-corrente-migration
# Restart HA
```

#### Opção C: Manual
```bash
# Restaurar backups
cp ~/ha-backup-v2.0/configuration.yaml config/
cp ~/ha-backup-v2.0/automations.yaml config/
cp ~/ha-backup-v2.0/scripts.yaml config/

# Deletar helpers novos via UI

# Restart HA
```

---

## 8. FAQ

### Q1: Posso testar v3.0 sem desativar v2.0?

**R:** Sim! Crie helpers com prefixo `v3_` temporariamente:
```yaml
v3_pump_current_threshold_low  # ao invés de pump_current_threshold_low
```
Teste em paralelo, valide, depois migre definitivo.

---

### Q2: Meu Tuya não tem sensor de corrente, e agora?

**R:** Opções:
1. Trocar por Shelly PM (tem medição corrente)
2. Ficar na v2.0 (ainda funciona bem)
3. Calcular corrente aproximada: `I = P/V` (impreciso)

---

### Q3: Os valores de corrente parecem errados

**R:** Execute calibração:
```yaml
service: script.pump_calibrate_current
```
Ajuste thresholds conforme recomendação.

---

### Q4: Consumo de energia mudou muito vs v2.0

**R:** Normal! v2.0 estava **errado** por causa do bug do utility_meter. 

Validar manualmente:
```
Runtime hoje: 3h (exemplo)
Potência: 68W
Energia: 3h × 0.068kW = 0.204 kWh
Custo: 0.204 × R$ 0.85 = R$ 0.17

Comparar com sensor.bomba_energia_hoje
```

---

### Q5: Posso pular a calibração?

**R:** Pode, mas NÃO recomendado. 

Thresholds default (0.303-0.323A) são baseados em dados reais, mas **sua bomba pode ser diferente**. 

Calibração leva 30s e otimiza detecção.

---

### Q6: Quanto tempo levam os statistics?

**R:** 
- Primeira leitura: 5-10 minutos
- Dados confiáveis: 1-2 horas
- Estatísticas precisas: 24 horas

Se após 24h ainda "unavailable" → problema config.

---

### Q7: Posso fazer migração sem Git?

**R:** Sim! Use backup manual (Passo 3) e snapshot do HA.

Git facilita rollback mas não é obrigatório.

---

### Q8: Preciso atualizar dashboard?

**R:** Não obrigatório para funcionamento, mas recomendado para ver novas métricas.

Dashboard v3.0 será feito em chat separado conforme combinado.

---

## 📞 Suporte

**Problemas durante migração?**
1. Verificar logs: Settings → System → Logs
2. Validar helpers: Developer Tools → Template
3. Testar sensores: Developer Tools → States
4. Se travar: fazer rollback e reportar erro

---

**Boa migração! 🚀**

---

**Versão:** 1.0  
**Data:** {{now().strftime('%d/%m/%Y')}}  
**Baseado em:** Sistema v2.0 estável + análise 1365 medições
