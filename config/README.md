# Estrutura de Código - Bomba Água Quente v3.5

## Arquivos

| Arquivo | Linhas | Descrição |
|---------|--------|-----------|
| `sensors.yaml` | ~130 | Derivative, Statistics, History Stats |
| `template_sensors.yaml` | ~750 | Template sensors (robôs, rega, UPS, **bomba**) |
| `automations_bomba.yaml` | ~520 | Automações só da bomba v3.5 |
| `scripts_bomba.yaml` | ~200 | Scripts da bomba (teste, reset, relatório) |

## Onde Está Cada Coisa

### Sensores Raw (Camada 1)
- **Derivative**: `sensors.yaml` linhas 50-63
- **Statistics 24h/7d/30d**: `sensors.yaml` linhas 69-111
- **History Stats runtime**: `sensors.yaml` linhas 41-48

### Detecção Inteligente (Camada 2)
- **bomba_corrente_estabilizada**: `template_sensors.yaml` seção bomba
- **bomba_mudanca_rapida_corrente**: `template_sensors.yaml` seção bomba
- **bomba_desgaste_emergente**: `template_sensors.yaml` seção bomba

### Alertas Combinados (Camada 3)
- **bomba_corrente_anormal**: `template_sensors.yaml` seção bomba
- **bomba_corrente_critica**: `template_sensors.yaml` seção bomba

### Automações
- **Controle principal**: `automations_bomba.yaml` linha 7
- **Timeout segurança**: `automations_bomba.yaml` linha 140
- **Controle manual**: `automations_bomba.yaml` linha 200
- **Desligamento emergência**: `automations_bomba.yaml` linha 260
- **Desgaste progressivo**: `automations_bomba.yaml` linha 320
- **Alerta mudança rápida (v3.5)**: `automations_bomba.yaml` linha 420
- **Alerta desgaste emergente (v3.5)**: `automations_bomba.yaml` linha 475

### Scripts
- **pump_system_test**: `scripts_bomba.yaml` linha 1
- **pump_emergency_reset**: `scripts_bomba.yaml` linha 75
- **pump_detailed_report**: `scripts_bomba.yaml` linha 130

## Como Usar

### Deploy Completo (novo sistema)
1. Copiar `sensors.yaml` → `/config/sensors.yaml`
2. Copiar `template_sensors.yaml` → `/config/template_sensors.yaml`
3. Adicionar conteúdo de `automations_bomba.yaml` ao seu `/config/automations.yaml`
4. Adicionar conteúdo de `scripts_bomba.yaml` ao seu `/config/scripts.yaml`

### Atualização v3.0 → v3.5
1. Substituir seção bomba em `template_sensors.yaml` (linhas 470-603)
2. Substituir automações `bomba_alerta_mudanca_rapida` e `bomba_alerta_desgaste_emergente`
3. Atualizar `sensors.yaml` seção derivative

## Validação

```yaml
# Developer Tools → Template
TESTE v3.5:
- Derivative: {{ states('sensor.bomba_taxa_mudanca_corrente') }} A/s
- Estabilizada: {{ states('binary_sensor.bomba_corrente_estabilizada') }}
- Mudança Rápida: {{ states('binary_sensor.bomba_mudanca_rapida_corrente') }}
- Desgaste Emergente: {{ states('binary_sensor.bomba_desgaste_emergente') }}
- Corrente Anormal: {{ states('binary_sensor.bomba_corrente_anormal') }}
```
