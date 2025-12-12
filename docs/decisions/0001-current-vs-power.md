# ADR-0001: Monitoramento por Corrente vs Potência

**Status**: Aceito | **Data**: 2024-12-11 | **Versão**: v3.0

## Contexto

O sistema v2.0 usava monitoramento por potência (48-53W) para detectar anomalias na bomba. Com mais dados coletados, identificamos que corrente é mais precisa.

## Problema

Como detectar melhor problemas na bomba (travamento, desgaste, sobrecarga)?

## Opções Analisadas

### 1. Potência (W) - Status Quo v2.0
**Prós:**
- Já implementado e calibrado (143 medições)
- Fácil de entender

**Contras:**
- Varia com tensão da rede (220V ± 10%)
- Menos preciso para detectar travamento inicial

### 2. Corrente (A) - Proposto
**Prós:**
- Mais estável (independente de variação de tensão)
- Detecção mais precisa de travamento (corrente sobe antes de potência)
- Permite cálculo de fator de potência

**Contras:**
- Precisa recalibrar thresholds
- Coleta de novos dados necessária

## Decisão

Escolhemos: **Corrente (A)**

## Justificativa

1. **Dados coletados**: 1826 amostras (dezembro 2024) mostram distribuição mais consistente
2. **Precisão**: Desvio padrão menor (0.005A vs 1.5W relativo)
3. **Física**: Motor travando = corrente aumenta primeiro
4. **Calibração**: 
   - Normal: 0.303-0.323A (P5-P95)
   - Crítico: 0.388A (max × 1.15)

## Consequências

### Positivas
- Detecção mais rápida de problemas
- Menos falsos positivos por variação de tensão
- Base para manutenção preditiva (tendências)

### Negativas Aceitas
- Migração necessária (um release)
- Helpers antigos precisam atualização
- Usuários precisam aprender novos valores

## Implementação

**Arquivos afetados:**
- `sensors.yaml`: Derivative e Statistics
- `template_sensors.yaml`: Binary sensors
- `automations.yaml`: Thresholds atualizados

**Helpers novos:**
- `input_number.pump_current_normal_min`: 0.303
- `input_number.pump_current_normal_max`: 0.323
- `input_number.pump_current_critical_max`: 0.388

## Referências

- Análise: 1826 amostras dezembro/2024
- Transcript: `2025-12-12-01-38-33-bomba-agua-v35-data-analysis.txt`
