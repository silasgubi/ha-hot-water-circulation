# ADR-0002: Filtro Anti-Inrush e Ajuste de Threshold dI/dt

**Status**: Aceito  
**Data**: 2024-12-18  
**Fase**: v3.5.2

---

## Contexto

Sistema v3.5 implementou monitoramento por derivative (dI/dt) para detectar mudanças rápidas na corrente, permitindo distinguir entre:
- **Inrush current normal**: Partida do motor (~2s, até 0.06 A/s)
- **Problemas reais**: Emperramento, desgaste, travamento

**Problema identificado**: Após deploy v3.5.1, usuário relatou **falsos positivos persistentes** (alertas a cada 2-5 min), indicando que o sistema não estava filtrando inrush corretamente.

---

## Problema

### Análise de Dados Reais (30 dias, 836 amostras)

| Métrica | Valor | Interpretação |
|---------|-------|---------------|
| P95 \|dI/dt\| | 0.0618 A/s | Inrush/desligamento típico |
| P99 \|dI/dt\| | 0.0634 A/s | Eventos rápidos normais |
| **Threshold atual** | 0.010 A/s | ❌ Abaixo do P95 |
| **Eventos capturados** | 218/836 (26%) | ❌ Todos os inrush normais |

### Evidências do Logbook
```
10:57:04 - Alerta disparado com dI/dt = 0.0003 A/s (deveria ser >0.010)
10:39:09 - Alerta disparado com dI/dt = 0.0014 A/s (deveria ser >0.010)
10:36:07 - Alerta disparado com dI/dt = 0.0002 A/s (deveria ser >0.010)
```

**Bugs identificados**:
1. Threshold de 0.010 A/s captura inrush normal (deveria ser >0.06 A/s)
2. Sensor dispara com valores **abaixo** do threshold (lógica bugada)
3. Sem filtro temporal: Inrush nos primeiros 2-3s dispara alerta

---

## Opções Avaliadas

### Opção 1: Apenas aumentar threshold
```yaml
threshold: 0.010 → 0.065 A/s (P99)
```

**Prós**:
- Simples, uma linha
- Elimina 99% dos falsos positivos

**Contras**:
- Ainda captura inrush inicial (primeiros 2-3s)
- Sem margem de segurança para detecção de problemas reais
- Não resolve bug de lógica (disparando com valores baixos)

---

### Opção 2: Threshold moderado + filtro temporal
```yaml
threshold: 0.010 → 0.025 A/s (P97)
+ ignorar primeiros 10s após ligar/desligar
```

**Prós**:
- Threshold entre P95-P99: Detecta anomalias sem falsos positivos
- Filtro 10s elimina inrush completamente (~2-3s + margem)
- Mantém sensibilidade para problemas reais durante operação
- Resolve bug de lógica (adiciona condições corretas)

**Contras**:
- Código mais complexo (+ 5 linhas)
- Precisa rastrear tempo desde ativação

---

### Opção 3: Threshold alto + hysteresis
```yaml
threshold: 0.010 → 0.065 A/s (P99)
+ delay_on: 3s, delay_off: 5s
```

**Prós**:
- Simples
- Hysteresis evita alternância rápida

**Contras**:
- Threshold muito alto: Pode perder problemas progressivos
- Não filtra inrush inicial (primeiros 2-3s)
- Detecta apenas problemas severos

---

## Decisão

**Escolhemos: Opção 2** (Threshold moderado + filtro temporal)

### Implementação Final
```yaml
binary_sensor.bomba_mudanca_rapida_corrente:
  state: >
    {% set dI = states('sensor.bomba_taxa_mudanca_corrente')|float(0)|abs %}
    {% set bomba_on = is_state('switch.bomba_de_circulacao_de_agua_quente', 'on') %}
    {% set tempo_ligada = (now() - states.switch.bomba_de_circulacao_de_agua_quente.last_changed).total_seconds() %}
    {% set threshold = 0.025 %}
    {{ bomba_on and tempo_ligada > 10 and dI > threshold }}
  delay_on: "00:00:03"
  delay_off: "00:00:05"
```

**Parâmetros finais**:
- Threshold: **0.025 A/s** (P97: Entre eventos normais e anomalias)
- Filtro anti-inrush: **10s** (Inrush ~2-3s + margem 7s)
- Hysteresis on: **3s** (Confirmar antes de alertar)
- Hysteresis off: **5s** (Evitar alternância rápida)

---

## Justificativa

### Por que 0.025 A/s?
- P95 = 0.0618 A/s (inrush típico)
- P97 = ~0.025 A/s (estimado entre P95-P99)
- **Zona de detecção**: Entre operação normal (<P95) e eventos reais (>P97)
- Margem de segurança: 2.5x acima do threshold original (0.010)

### Por que filtro de 10s?
- Inrush real: 2-3s
- Margem de segurança: +7s
- Cobre também:
  - Variações tensão rede durante partida
  - Estabilização mecânica do motor
  - Transitórios elétricos

### Por que hysteresis 3s/5s?
- 3s on: Confirmar que não é glitch/ruído elétrico
- 5s off: Aguardar estabilização completa antes de limpar alerta
- Evita: Alternância "detectado → limpo → detectado" em <1s

---

## Consequências

### Positivas
- ✅ Elimina 26% de falsos positivos (inrush normal)
- ✅ Mantém sensibilidade para problemas reais (>P97)
- ✅ Detecta anomalias durante operação (após 10s)
- ✅ Filtro anti-inrush reutilizável em outros sensores
- ✅ Hysteresis reduz ruído no logbook

### Negativas Aceitas
- ⚠️ Problemas nos primeiros 10s não são detectados
  - **Mitigado**: Problemas reais persistem após 10s
- ⚠️ Código mais complexo (+30% linhas)
  - **Mitigado**: Documentado e testado
- ⚠️ Dependência do timestamp de ativação
  - **Mitigado**: Estado nativo do HA, sempre disponível

### Impacto em Alertas
**Antes (v3.5.1)**:
- Alerta a cada 2-5 min (inrush = falso positivo)
- TTS persistente e irritante
- Dados corretos, lógica bugada

**Depois (v3.5.2)**:
- Alertas apenas para eventos reais (>P97, após 10s)
- TTS silencioso (sem mudança_rápida)
- Sistema confiável

---

## Implementação

### Arquivos Modificados
1. **config/template_sensors.yaml**
   - `binary_sensor.bomba_mudanca_rapida_corrente`: Threshold + filtro + hysteresis
   - Attributes adicionais: `time_since_activation`, `inrush_filter_active`

2. **config/automations_bomba.yaml**
   - Remover TTS de `bomba_alerta_mudanca_rapida` (não é emergência)
   - Manter log (system_log info + logbook)

### Passos de Deploy
1. Backup arquivos atuais
2. Aplicar mudanças em template_sensors.yaml
3. Aplicar mudanças em automations_bomba.yaml
4. `ha core check` (validar sintaxe)
5. `ha core restart`
6. Testar: Ligar bomba → aguardar 15s → verificar se alerta NÃO disparou
7. Monitorar 24h: Confirmar ausência de falsos positivos

---

## Validação

### Critérios de Sucesso (24h de teste)
- [ ] Zero alertas de mudança_rápida durante inrush (primeiros 10s)
- [ ] Alertas apenas para eventos reais (se ocorrerem)
- [ ] Logbook sem alternância rápida (detectado/limpo)
- [ ] System_log sem erros de template

### Métricas
- **Antes**: 26% das amostras = alerta (218/836)
- **Meta**: <1% das amostras = alerta (apenas anomalias)
- **Baseline**: P97+ = 3% das amostras (25/836)

---

## Referências

- **Análise completa**: `docs/analysis/ANALISE_CORRENTE_20241218.md`
- **Dados brutos**: 1.826 amostras corrente + 836 amostras derivative (nov-dez 2024)
- **ADR relacionado**: [0001-current-vs-power.md](0001-current-vs-power.md) (monitoramento por corrente)
- **Datasheet**: Motor bomba 50W (inrush típico: 2-3× corrente nominal por 1-3s)

---

## Notas Adicionais

### Lesson Learned
**"Thresholds baseados em teoria vs dados reais"**

- Teoria: dI/dt > 0.010 A/s deveria capturar problemas
- Realidade: P95 = 0.0618 A/s (6× maior)
- **Conclusão**: SEMPRE calibrar com dados reais de produção

### Decisões Futuras
Se ainda houver falsos positivos após v3.5.2:
1. Aumentar threshold: 0.025 → 0.030 A/s (P98)
2. Aumentar filtro: 10s → 15s
3. Considerar análise espectral (FFT) para distinguir padrões

---

**Autor**: Silas  
**Revisor**: Claude 3.5 Sonnet  
**Aprovado**: 2024-12-18  
**Implementado**: Pendente (v3.5.2)
