# Dashboard Bomba Água Quente v3.5.1

Este diretório contém dois arquivos de dashboard para o sistema de Bomba de Água Quente.

## 📁 Arquivos Disponíveis

### 1. `lovelace_bomba_sidebar.yaml` (RECOMENDADO)

**Dashboard completo com Mushroom Cards**

- ✅ Interface moderna e responsiva
- ✅ Cards personalizados com melhor UX
- ✅ Gráficos avançados (Mini Graph Card)
- ✅ Melhor visualização mobile
- ⚠️ **Requer:** Mushroom Cards, Mini Graph Card, Card Mod (via HACS)

**Use se:** Você tem HACS instalado e quer a melhor experiência visual

### 2. `lovelace_bomba_sidebar_native.yaml`

**Dashboard usando apenas cards nativos do Home Assistant**

- ✅ Sem dependências de custom cards
- ✅ Funciona imediatamente após instalação
- ✅ Todas as funcionalidades principais
- ✅ Gráficos básicos (history-graph)
- ℹ️ **Requer:** Nada além do HA padrão

**Use se:** Você não tem HACS ou prefere não usar custom cards

## 🚀 Instalação Rápida

### Opção A: Via Interface (RECOMENDADO)

1. **Configurações** → **Dashboards**
2. Selecione dashboard principal → **Editar Dashboard**
3. **+ ADICIONAR VIEW**
4. Configure:
   - Título: **Bomba Água Quente**
   - Ícone: **mdi:pump**
   - Tipo: **Sidebar**
5. Editar em **YAML Bruto**
6. Cole conteúdo de um dos arquivos acima
7. **SALVAR**

### Opção B: Via configuration.yaml

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

## 📖 Documentação Completa

Veja o guia completo em: **`docs/dashboard-installation-guide.md`**

## 🎨 Seções do Dashboard

Ambos os dashboards incluem:

1. **Status Principal** - Visão geral do sistema
2. **Monitoramento de Corrente** - Gauge + métricas (atual, taxa, médias)
3. **Diagnóstico v3.5** - Análise inteligente em tempo real
4. **Histórico** - Gráficos 2h/24h
5. **Controles** - Override, manual, teste, reset
6. **Hardware** - Dispositivos e sensores
7. **Configurações** - Thresholds, delays, timeouts
8. **DEBUG** - Timers, automações, troubleshooting
9. **Informações** - Versão e documentação

## 🔧 Pré-requisitos

Antes de instalar, certifique-se de ter:

### Entidades Necessárias

- ✅ `valve.hot_water` (Sonoff Mini)
- ✅ `switch.bomba_de_circulacao_de_agua_quente` (Tuya TS011F)
- ✅ `sensor.bomba_de_circulacao_de_agua_quente_corrente`
- ✅ `sensor.bomba_taxa_mudanca_corrente` (Derivative v3.5)
- ✅ Todos os binary_sensors v3.5 (estabilizada, mudanca_rapida, etc.)

### Helpers

- ✅ 6x `input_number` (thresholds, delays, timeout)
- ✅ 3x `input_boolean` (override, manual, alerts)
- ✅ 3x `timer` (activation, deactivation, timeout)

Veja lista completa em: `docs/dashboard-installation-guide.md`

## 🐛 Problemas Comuns

### Cards em branco
→ Instale Mushroom Cards via HACS OU use versão Native

### Entidades "unavailable"
→ Verifique `sensors.yaml` e `template_sensors.yaml`

### Dashboard não aparece
→ Certifique-se tipo = "sidebar" ou `show_in_sidebar: true`

## 📱 Compatibilidade

- ✅ **Desktop:** Todas as funcionalidades
- ✅ **Mobile:** Layout responsivo
- ✅ **Tablet:** Otimizado
- ✅ **Dark/Light Mode:** Segue tema do HA

## 🔄 Diferenças entre Versões

| Funcionalidade | Mushroom | Native |
|----------------|----------|--------|
| Gauge de Corrente | ✅ | ✅ |
| Histórico 2h/24h | ✅ | ✅ |
| Diagnóstico Inteligente | ✅ | ✅ |
| Mini Graph Card | ✅ | ❌ |
| Mushroom Cards | ✅ | ❌ |
| Card Mod (estilo) | ✅ | ❌ |
| Funciona sem HACS | ❌ | ✅ |

## 📚 Arquivos Relacionados

- `sensors.yaml` - Derivative + Statistics (CAMADA 1)
- `template_sensors.yaml` - Binary sensors v3.5 (CAMADA 2/3)
- `automations_bomba.yaml` - Controle e alertas
- `scripts_bomba.yaml` - Teste, reset, relatório
- `dashboard_bomba_v35.yaml` - Dashboard antigo (card único)

## 💡 Recomendações

1. **Primeira instalação?** → Use versão Native
2. **Quer melhor UX?** → Instale HACS + Mushroom
3. **Modo DEBUG ativo?** → Crie `input_boolean.pump_debug_mode`
4. **Monitore 7 dias** → Para sensores de tendência funcionarem

---

**v3.5.1** - Dezembro 2024
**Sistema de Circulação de Água Quente**
