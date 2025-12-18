# Lições Aprendidas - Bomba Água Quente

⚠️ **VERIFICAÇÃO**: Se leu este arquivo, mencione "📚 Lições carregadas"

## Hardware

### Sonoff Mini - Sensor Fluxo Stuck
**Problema**: Sensor ocasionalmente fica "stuck" em estado "open" durante pulsos rápidos de água
**Causa**: Firmware do Sonoff tem debounce insuficiente para fluxos muito rápidos
**Solução**: Delay de 3s na automação antes de acionar bomba (confirma fluxo real)
**Validado**: 2024-01 (v2.0)
**Arquivos**: `automations.yaml` - `pump_activation_delay`
**Referências**: Reportado em fórum eWeLink

### Tuya TS011F - Leitura de Corrente
**Problema**: Leitura de corrente tem precisão de ±0.01A
**Causa**: ADC do chip tem resolução limitada
**Solução**: Usar Statistics sensor para média (reduz ruído)
**Validado**: 2024-12 (v3.0)
**Arquivos**: `sensors.yaml` - `bomba_corrente_media_*`
**Referências**: Datasheet Tuya

## Software

### Derivative - Unidade de Tempo
**Problema**: Derivative sensor configurado com `unit_time: min` e `time_window: "00:05:00"` não detectava inrush (~2s)
**Causa**: Janela de 5 minutos dilui evento de 2 segundos para quase zero
**Solução**: `unit_time: s` e `time_window: "00:00:05"` (5 segundos)
**Validado**: 2024-12-12 (v3.5)
**Arquivos**: `sensors.yaml` - `bomba_taxa_mudanca_corrente`
**Referências**: Documentação HA Derivative

### Inrush Current - Falsos Positivos
**Problema**: Pico de corrente na partida (~0.35A→0.31A em 2s) disparava alertas
**Causa**: Sensor `bomba_corrente_anormal` com delay simples não era suficiente
**Solução**: Lógica v3.5 - só alerta se `estabilizada AND fora_faixa`
**Validado**: 2024-12-12 (v3.5)
**Arquivos**: `template_sensors.yaml` - `bomba_corrente_anormal`, `bomba_corrente_estabilizada`
**Referências**: Arquitetura 3 camadas v3.5

### Utility Meter - Sensor Cumulativo
**Problema**: Utility meter calculava errado com sensor `summation_delivered` (cumulativo)
**Causa**: Reset do utility meter não considera que fonte é sempre crescente
**Solução**: v3.0 removeu utility_meter, usa history_stats para runtime
**Validado**: 2024-12 (v3.0)
**Arquivos**: `sensors.yaml` - `bomba_runtime_hoje` (history_stats)
**Referências**: Issue HA #XXXXX

### History Stats - Template Complexo
**Problema**: Template sensor calculando runtime era impreciso e pesado
**Causa**: Loop em Jinja2 sobre history é lento e pode falhar
**Solução**: Usar `history_stats` nativo do HA
**Validado**: 2024-12 (v3.0)
**Arquivos**: `sensors.yaml` linha 41-48
**Referências**: Documentação HA History Stats

### Threshold Mudança Rápida - Falsos Positivos Persistentes
**Problema**: Threshold 0.010 A/s disparava em 26% das amostras (inrush normal + desligamento)
**Causa**: Threshold muito baixo (abaixo do P95=0.0618 A/s), sem filtro anti-inrush, sem hysteresis
**Solução**:
- Threshold aumentado para 0.025 A/s (P97)
- Filtro anti-inrush: ignora primeiros 10s após ligação
- Hysteresis: delay_on=3s, delay_off=5s
**Validado**: 2024-12-18 (v3.5.2)
**Arquivos**: `template_sensors.yaml` - `bomba_mudanca_rapida_corrente`
**Referências**: Análise 1.826 amostras (docs/analysis/ANALISE_CORRENTE_20241218.md)

### TTS Estratégico vs Universal
**Problema**: TTS em alertas técnicos (mudança rápida, desgaste emergente) causava poluição sonora
**Causa**: Nem todo alerta requer ação imediata - monitores técnicos vs emergências
**Solução**:
- TTS APENAS em emergências: corrente crítica (>0.388A) + timeout (30min)
- Alertas técnicos: system_log (info) + logbook + dashboard
**Validado**: 2024-12-18 (v3.5.2)
**Arquivos**: `automations_bomba.yaml`
**Referências**: Análise falsos positivos (docs/MUDANCAS_V352.md)

## Calibração

### Thresholds - Dados Reais vs Teóricos
**Problema**: Valores teóricos (50W/0.23A) não batiam com realidade
**Causa**: Bomba real consome mais que especificado (68W/0.31A)
**Solução**: SEMPRE calibrar com dados reais (mínimo 100 amostras)
**Validado**: 2024-01 (143 amostras potência), 2024-12 (1826 amostras corrente)
**Arquivos**: `CLAUDE.md` - seção Thresholds
**Referências**: Análise estatística nos transcripts

### Cobertura Estatística - P5/P95 vs Min/Max
**Problema**: Usar min/max absoluto inclui outliers
**Causa**: Outliers são eventos únicos, não representam operação normal
**Solução**: Usar percentis P5/P95 para faixa normal
**Validado**: 2024-12 (v3.5)
**Arquivos**: `template_sensors.yaml` - thresholds 0.303-0.323A
**Referências**: Análise de 1826 amostras

## Arquitetura

### Big Bang vs Incremental
**Problema**: Atualizações incrementais deixavam sistema inconsistente
**Causa**: Sensores dependem uns dos outros, atualizar parcialmente quebra
**Solução**: Migração "Big Bang" - atualizar todos arquivos de uma vez
**Validado**: 2024-12 (v3.0, v3.5)
**Arquivos**: Todos
**Referências**: Workflow definido em user preferences

### Documentação - Estado Atual vs Planejado
**Problema**: Documentação misturava "planejado" com "implementado"
**Causa**: Documentar antes de validar
**Solução**: CLAUDE.md sempre reflete estado ATUAL, sobrescrito a cada versão
**Validado**: 2024-12
**Arquivos**: `CLAUDE.md`
**Referências**: User preferences - workflow obrigatório
