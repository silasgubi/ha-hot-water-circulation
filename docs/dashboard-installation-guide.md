# Guia de Instalação - Dashboard Bomba Água Quente v3.5.1

## 📋 Visão Geral

Este guia explica como adicionar o dashboard da Bomba de Água Quente à aba lateral do Home Assistant.

Existem **duas versões** disponíveis:

1. **`lovelace_bomba_sidebar.yaml`** - Versão completa com Mushroom Cards (RECOMENDADO)
2. **`lovelace_bomba_sidebar_native.yaml`** - Versão usando apenas cards nativos do HA

---

## 🎯 Método 1: Dashboard via UI (RECOMENDADO)

Este é o método mais fácil e não requer editar arquivos de configuração.

### Passo 1: Preparar Custom Cards (apenas para versão Mushroom)

Se for usar a versão com Mushroom Cards, instale via HACS:

1. Abra **HACS** → **Frontend**
2. Pesquise e instale:
   - **Mushroom Cards**
   - **Mini Graph Card** (opcional, para gráficos melhores)
   - **Card Mod** (opcional, para estilização)

3. Reinicie o Home Assistant após instalar

**IMPORTANTE:** Se não tiver HACS ou não quiser usar custom cards, use a versão `lovelace_bomba_sidebar_native.yaml`

### Passo 2: Criar Nova View no Dashboard

1. Abra o **Home Assistant**
2. Vá em **Configurações** → **Dashboards**
3. Selecione seu dashboard principal (geralmente "Home")
4. Clique nos **três pontos** (⋮) no canto superior direito
5. Selecione **"Editar Dashboard"**
6. Clique em **"+ ADICIONAR VIEW"**

### Passo 3: Configurar a Nova View

1. Na nova view, clique em **"EDITAR"** (ícone de lápis)
2. Configure:
   - **Título:** Bomba Água Quente
   - **Ícone:** mdi:pump
   - **Path:** bomba-agua-quente
   - **Tipo:** Sidebar

3. Clique em **"SALVAR"**

### Passo 4: Adicionar o Conteúdo YAML

1. Na view recém-criada, clique nos **três pontos** (⋮)
2. Selecione **"Editar em YAML Bruto"**
3. **APAGUE TODO** o conteúdo existente
4. Cole o conteúdo de um dos arquivos:
   - **Com Mushroom:** `config/lovelace_bomba_sidebar.yaml`
   - **Sem custom cards:** `config/lovelace_bomba_sidebar_native.yaml`

5. Clique em **"SALVAR"**
6. Clique em **"CONCLUÍDO"** para sair do modo de edição

### Passo 5: Verificar

1. A nova aba **"Bomba Água Quente"** deve aparecer na sidebar
2. Clique nela para verificar se tudo está funcionando
3. Verifique se todos os sensores estão exibindo dados corretos

---

## 🎯 Método 2: Dashboard via Configuration.yaml

Este método é mais avançado e permite versionamento via Git.

### Passo 1: Mover Arquivo

Copie o arquivo YAML escolhido para a pasta `/config/`:

```bash
# Escolha UMA das opções:

# Opção 1: Com Mushroom Cards
cp config/lovelace_bomba_sidebar.yaml /config/dashboards/bomba.yaml

# Opção 2: Native Cards
cp config/lovelace_bomba_sidebar_native.yaml /config/dashboards/bomba.yaml
```

### Passo 2: Editar configuration.yaml

Adicione ao seu `configuration.yaml`:

```yaml
lovelace:
  mode: storage
  dashboards:
    bomba-agua-quente:
      mode: yaml
      title: Bomba Água Quente
      icon: mdi:pump
      show_in_sidebar: true
      filename: dashboards/bomba.yaml
```

### Passo 3: Verificar Configuração

```bash
ha core check
```

### Passo 4: Reiniciar Home Assistant

```bash
ha core restart
```

### Passo 5: Verificar

1. Após reiniciar, a aba **"Bomba Água Quente"** deve aparecer na sidebar
2. Clique nela para verificar

---

## 🔧 Pré-requisitos

Antes de instalar o dashboard, certifique-se de que todos os componentes da v3.5 estão configurados:

### ✅ Entidades Necessárias

#### Input Numbers
```yaml
input_number:
  pump_current_normal_min:
    min: 0
    max: 1
    step: 0.001
    initial: 0.303
  pump_current_normal_max:
    min: 0
    max: 1
    step: 0.001
    initial: 0.323
  pump_current_critical_max:
    min: 0
    max: 1
    step: 0.001
    initial: 0.388
  pump_activation_delay_seconds:
    min: 0
    max: 60
    step: 1
    initial: 3
  pump_deactivation_delay_seconds:
    min: 0
    max: 300
    step: 5
    initial: 20
  pump_timeout_minutes:
    min: 0
    max: 120
    step: 5
    initial: 30
```

#### Input Booleans
```yaml
input_boolean:
  pump_manual_override:
    name: Override Manual da Bomba
    icon: mdi:shield-off
  pump_manual_control:
    name: Controle Manual da Bomba
    icon: mdi:hand-back-right
  pump_alerts_enabled:
    name: Alertas TTS Habilitados
    icon: mdi:volume-high
  # Opcional - para seção DEBUG
  pump_debug_mode:
    name: Modo Debug
    icon: mdi:bug
```

#### Timers
```yaml
timer:
  pump_activation_delay:
    duration: "00:00:03"
  pump_deactivation_delay:
    duration: "00:00:20"
  pump_safety_timeout:
    duration: "00:30:00"
```

### ✅ Sensores v3.5

Certifique-se de que os seguintes sensores estão configurados:

- `sensor.bomba_taxa_mudanca_corrente` (Derivative)
- `sensor.bomba_corrente_media_24h` (Statistics)
- `sensor.bomba_corrente_media_7d` (Statistics)
- `sensor.bomba_corrente_media_30d` (Statistics)
- `binary_sensor.bomba_corrente_estabilizada`
- `binary_sensor.bomba_mudanca_rapida_corrente`
- `binary_sensor.bomba_desgaste_emergente`
- `binary_sensor.bomba_corrente_anormal`
- `binary_sensor.bomba_corrente_critica`

Verifique em:
- `config/sensors.yaml` (Derivative + Statistics)
- `config/template_sensors.yaml` (Binary sensors + Templates)

---

## 🎨 Personalização

### Esconder Seção DEBUG

Se não quiser ver a seção DEBUG, você tem duas opções:

**Opção 1: Criar input_boolean e deixar desligado**
```yaml
input_boolean:
  pump_debug_mode:
    name: Modo Debug
    initial: off
```

**Opção 2: Remover seção DEBUG**

Na versão Mushroom (`lovelace_bomba_sidebar.yaml`), apague a SEÇÃO 8 completa (linhas que começam com o comentário "SEÇÃO 8: DEBUG").

Na versão Native, apague o card correspondente.

### Alterar Cores e Temas

O dashboard respeita o tema configurado no Home Assistant. Para personalizar:

1. Instale um tema via HACS (ex: iOS Dark Mode, Mushroom Themes)
2. Ative em **Configurações** → **Perfil** → **Tema**

### Modificar Thresholds no Dashboard

Os thresholds são lidos dos `input_number`, mas se quiser alterar as cores do gauge:

```yaml
severity:
  green: 0.303    # Início da faixa verde
  yellow: 0.323   # Início da faixa amarela
  red: 0.388      # Início da faixa vermelha
```

---

## 🐛 Troubleshooting

### Problema: Cards aparecem em branco

**Causa:** Custom cards não instalados

**Solução:**
1. Instale Mushroom Cards via HACS
2. OU use a versão Native: `lovelace_bomba_sidebar_native.yaml`

### Problema: Entidades "unavailable"

**Causa:** Sensores v3.5 não configurados

**Solução:**
1. Verifique se `sensors.yaml` e `template_sensors.yaml` estão corretos
2. Execute `ha core check`
3. Reinicie o HA
4. Verifique em **Developer Tools** → **Estados** se as entidades existem

### Problema: Gráficos não aparecem

**Causa:** Mini Graph Card não instalado (versão Mushroom)

**Solução:**
1. Instale Mini Graph Card via HACS
2. OU remova o card `custom:mini-graph-card` e use apenas `history-graph`

### Problema: Dashboard não aparece na sidebar

**Causa:** Configuração de view incorreta

**Solução:**
1. Certifique-se de que o tipo é **"sidebar"** (Método 1)
2. Certifique-se de que `show_in_sidebar: true` (Método 2)
3. Limpe cache do navegador (Ctrl+Shift+R)

---

## 📱 Mobile vs Desktop

O dashboard é responsivo e funciona em ambos, mas algumas dicas:

### Mobile
- Use a versão Mushroom para melhor UX mobile
- Considere esconder a seção DEBUG
- Os gráficos podem ser compactados

### Desktop
- Aproveite a seção DEBUG completa
- Gráficos ficam mais legíveis
- Pode dividir em duas colunas (editar grid columns)

---

## 🔄 Atualizações Futuras

### Como atualizar o dashboard

1. Baixe a nova versão do arquivo YAML
2. Se usar **Método 1 (UI)**:
   - Edite a view em YAML Bruto
   - Cole o novo conteúdo
   - Salve

3. Se usar **Método 2 (config.yaml)**:
   - Substitua o arquivo em `/config/dashboards/`
   - Execute `ha core check`
   - Reinicie o HA

### Versionamento

O dashboard segue a versão do sistema:

- **v3.5.1** - Versão atual (dezembro 2024)
- Sensores derivative corrigidos
- Detecção inteligente de inrush

---

## 📚 Documentação Adicional

- **Sistema Completo:** `CLAUDE.md`
- **Lessons Learned:** `docs/lessons-learned.md`
- **ADR Corrente vs Potência:** `docs/decisions/0001-current-vs-power.md`
- **Automações:** `config/automations_bomba.yaml`
- **Scripts:** `config/scripts_bomba.yaml`

---

## 💡 Dicas

1. **Comece com a versão Native** se não tiver experiência com custom cards
2. **Habilite DEBUG mode** inicialmente para entender o funcionamento
3. **Monitore por 7 dias** para ver os sensores de tendência (média 7d/30d) funcionarem
4. **Ajuste thresholds** conforme necessário baseado em seus dados reais
5. **Use o botão Relatório** para análise detalhada do sistema

---

## 📞 Suporte

Em caso de dúvidas:
1. Verifique os logs: **Configurações** → **Sistema** → **Logs**
2. Valide YAML: `ha core check`
3. Verifique estados: **Developer Tools** → **Estados**
4. Consulte `docs/lessons-learned.md` para problemas conhecidos

---

**Desenvolvido para Home Assistant**
**Sistema de Bomba de Água Quente v3.5.1**
