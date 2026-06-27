# Pendências — v3.6.0

## Instalação no Home Assistant

### 1. Criar helper `bomba_temp_funcionamento_minima`

**Via UI** (recomendado):
1. Settings → Devices & Services → Helpers → + CREATE HELPER
2. Tipo: **Number**
3. Preencher:
   - Name: `Temp. Mínima Esperada (Circulação)`
   - Entity ID: `bomba_temp_funcionamento_minima`
   - Min: `25` | Max: `45` | Step: `0.5`
   - Unit: `°C` | Icon: `mdi:thermometer-water` | Mode: Slider
4. Salvar → definir valor para **32**

**Via configuration.yaml** (alternativa):
```yaml
input_number:
  bomba_temp_funcionamento_minima:
    name: Temp. Mínima Esperada (Circulação)
    min: 25
    max: 45
    step: 0.5
    initial: 32
    unit_of_measurement: "°C"
    icon: mdi:thermometer-water
    mode: slider
```

---

### 2. Atualizar os arquivos YAML no HA

| Arquivo | O que copiar | Como recarregar |
|---------|-------------|-----------------|
| `sensors.yaml` | Bloco do derivative `bomba_taxa_mudanca_temperatura` | Developer Tools → YAML → Template entities |
| `template_sensors.yaml` | Bloco `binary_sensor.bomba_falha_circulacao` | Developer Tools → YAML → Template entities |
| `automations_bomba.yaml` | Automação `bomba_alerta_falha_circulacao` (automação #8) | Developer Tools → YAML → Automations |

Ou reiniciar o HA completo para carregar tudo de uma vez.

---

### 3. Teste de validação

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
