# Pendências — v3.6.0

## Instalação no Home Assistant

**Status: ✅ INSTALADO — 2026-06-27**

Todos os arquivos foram aplicados diretamente em `z:\` (HA config):

| Arquivo | Mudança | Status |
|---------|---------|--------|
| `z:\configuration.yaml` | `input_number.bomba_temp_funcionamento_minima` (32°C) | ✅ |
| `z:\sensors.yaml` | `sensor.bomba_taxa_mudanca_temperatura` (derivative °C/min) | ✅ |
| `z:\template_sensors.yaml` | `binary_sensor.bomba_falha_circulacao` (Layer 2d) | ✅ |
| `z:\automations\systems\water_pump.yaml` | Automação #8 `bomba_alerta_falha_circulacao` | ✅ |
| `z:\AUTOMATIONS_STRUCTURE.md` | 7 → 8 automações, total 114 → 115 | ✅ |
| `z:\CHANGELOG.md` | Entrada v3.6.0 na seção Unreleased | ✅ |

---

## Próximos passos necessários

### 1. Recarregar no HA

```
Developer Tools → YAML → Reload Template Entities
Developer Tools → YAML → Reload Automations
Developer Tools → YAML → Restart (para configuration.yaml — o helper novo exige restart)
```

Ou restart completo via Developer Tools → YAML → Restart Home Assistant.

### 2. Definir valor do helper

Após restart, verificar que `input_number.bomba_temp_funcionamento_minima` aparece com valor 32°C.
Se aparecer vazio: Settings → Helpers → buscar "Temp. Mínima" → definir 32.

### 3. Teste de validação (recomendado)

**Teste de falha (confirma que alerta funciona):**
1. Definir `bomba_temp_funcionamento_minima` para **50°C** via UI
2. Abrir uma torneira (ou ligar bomba manualmente)
3. Aguardar **2 minutos**
4. Verificar:
   - `binary_sensor.bomba_falha_circulacao` = `on`
   - TTS disparou no speaker da sala
   - Push notification chegou no iPhone
   - Notificação persistente aparece no dashboard

**Teste normal (confirma zero falsos positivos):**
1. Voltar threshold para **32°C**
2. Abrir torneira normalmente
3. Confirmar que temperatura sobe acima de 32°C antes de 2 min → sem alerta

---

### 4. Calibração futura (após 7-14 dias)

Com banco de dados populado, analisar:
- Temperatura mínima real no funcionamento normal → ajustar threshold se necessário
- Verificar se algum falso positivo ocorreu (manhãs frias, aquecedor lento)
- Considerar `input_number.bomba_temp_delta_minimo` (delta relativo) para threshold adaptativo por estação
