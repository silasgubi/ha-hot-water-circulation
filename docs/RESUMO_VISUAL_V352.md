# Resumo Visual - Correções v3.5.2

## 🎯 Problemas Identificados vs Soluções

### 1️⃣ Mudança Rápida - Falsos Positivos Constantes

**ANTES** ❌
```
Threshold: 0.010 A/s
Captura: 26% das amostras (218/836)
Problema: Dispara em TODO inrush (2-3s)
Resultado: Alerta a cada 2-5 minutos
```

**DEPOIS** ✅
```
Threshold: 0.025 A/s (P97)
Filtro anti-inrush: Ignora primeiros 10s
Hysteresis: delay_on=3s, delay_off=5s
Resultado: Alertas apenas para problemas REAIS
```

---

### 2️⃣ Desgaste Emergente - Dispara Durante Desligamento

**ANTES** ❌
```
Avalia: Sempre (mesmo bomba desligada)
Delay: Nenhum → alerta instantâneo
Problema: Dispara em transitórios (liga/desliga)
Screenshot: Taxa -0.0443 A/s = desligamento
```

**DEPOIS** ✅
```
Avalia: Só quando bomba ON + estabilizada
Delay on: 30 minutos (confirma persistência)
Delay off: 1 hora (auto-reset)
Resultado: Só alerta desgaste REAL (>30min)
```

---

## 📊 Impacto nas Notificações

### TTS (Voz)

| Alerta | Antes | Depois | Razão |
|--------|-------|--------|-------|
| Mudança rápida | 🔊 Sempre | 📊 Dashboard | Inrush normal |
| Desgaste emergente | 🔊 Sempre | 📊 Dashboard | Preventivo |
| Corrente anormal | 🔊 Sempre | 📊 Dashboard | Monitoramento |
| **Corrente crítica** | 🔊 Sempre | 🔊 Sempre | ✅ Emergência |
| **Timeout 30min** | 🔊 Sempre | 🔊 Sempre | ✅ Vazamento |

**Redução TTS**: 5 alertas → 2 alertas (60% menos ruído)

---

## 🔧 Arquivos Modificados

### template_sensors.yaml

**Mudanças**:
1. `bomba_mudanca_rapida_corrente`:
   - Threshold: 0.010 → 0.025 A/s
   - Filtro: +10s anti-inrush
   - Hysteresis: +3s/5s

2. `bomba_desgaste_emergente`:
   - Condições: +bomba_on +estabilizada
   - Delay on: +30min
   - Delay off: +1h

### automations_bomba.yaml

**Mudanças**:
- Remover TTS de 3 automações:
  - `bomba_alerta_mudanca_rapida`
  - `bomba_alerta_desgaste_emergente`
  - `bomba_alerta_corrente_anormal`
- Manter logs (system_log + logbook)

---

## ⏱️ Linha do Tempo Esperada

### Antes da Implementação
```
00:00 - Bomba liga
00:01 - 🔊 TTS: "Mudança rápida!" (inrush)
00:02 - 🔊 TTS: "Corrente anormal!" (inrush)
00:05 - Bomba estabiliza
...
00:10 - Bomba desliga
00:11 - 🔊 TTS: "Mudança rápida!" (desligamento)
00:12 - 🔊 TTS: "Desgaste emergente!" (falso positivo)
```
**Resultado**: 4 alertas falsos em 12 minutos

---

### Depois da Implementação
```
00:00 - Bomba liga
00:01-00:10 - [Filtro anti-inrush ativo - sem alertas]
00:11 - Bomba estabiliza (✅ corrente_estabilizada = on)
...
10:00 - Bomba desliga
10:01 - [Sensor desgaste para de avaliar - bomba off]
```
**Resultado**: 0 alertas falsos

---

### Se Problema REAL Ocorrer
```
00:00 - Bomba liga
00:10 - Filtro anti-inrush desativa
00:11 - Corrente anormal detectada (>0.025 A/s)
00:14 - Hysteresis confirma (3s) → 📊 Dashboard atualiza
...
00:40 - Problema persiste 30min → 🔊 TTS (se desgaste)
```
**Resultado**: Alerta apenas se problema REAL e PERSISTENTE

---

## ✅ Expectativas Pós-Deploy

| Métrica | Antes | Depois | Melhoria |
|---------|-------|--------|----------|
| Alertas/hora (falsos) | ~12 | ~0 | 100% |
| TTS/dia | ~50 | ~2 | 96% |
| Detecção problemas reais | 100% | 100% | Mantida |
| Auto-reset | Manual | 1h | Automático |

---

## 🚨 Se Ainda Houver Problemas (Pós-Deploy)

### Plano B - Ajustes Incrementais

1. **Mudança rápida ainda dispara?**
   → Aumentar threshold: 0.025 → 0.030 A/s

2. **Desgaste emergente falso positivo?**
   → Aumentar threshold: 3% → 5%
   → Aumentar delay: 30min → 1h

3. **Corrente anormal ainda dispara?**
   → Revisar lógica do binary_sensor

---

**Documentação atualizada**: MUDANCAS_V352.md  
**Próximo passo**: Implementar no Claude Code
