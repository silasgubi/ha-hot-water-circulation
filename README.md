# 💧 Hot Water Circulation Intelligence

> Sistema inteligente de automação de bomba de circulação de água quente para Home Assistant com detecção de inrush current usando derivative

[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-2024.12-blue.svg)](https://www.home-assistant.io/)
[![Version](https://img.shields.io/badge/Version-3.5.1-success.svg)](https://github.com/silasgubi/ha-hot-water-circulation/releases/tag/v3.5.1)
[![Status](https://img.shields.io/badge/Status-Em%20Produção-brightgreen.svg)](https://github.com/silasgubi/ha-hot-water-circulation)
[![Português BR](https://img.shields.io/badge/Idioma-Portugu%C3%AAs%20BR-green.svg)](https://github.com/silasgubi/ha-hot-water-circulation)

---

## 📖 Sobre o Projeto

**Hot Water Circulation Intelligence** é um sistema de automação residencial que detecta automaticamente a abertura de torneiras de água quente e aciona uma bomba de circulação (50W/0.31A), garantindo água quente instantânea com:

✨ **Detecção automática de fluxo** via Sonoff Mini
⚡ **Monitoramento inteligente por corrente** (0.303-0.323A)
🎯 **Ignora inrush current** usando derivative (dI/dt)
📊 **Alertas preditivos** de desgaste e problemas
🛡️ **Proteção crítica** com desligamento automático (>0.388A)
🔄 **Análise estatística** 24h/7d/30d para detecção de tendências

---

## 🎬 Demonstração

```
┌─────────────────────────────────────────────┐
│ Cenário: Usuário abre torneira água quente │
└─────────────────────────────────────────────┘

t=0s    → Válvula abre (Sonoff Mini detecta)
t=3s    → Delay confirmação (evita acionamentos falsos)
t=3s    → Bomba LIGA (Tuya TS011F)

t=4s    → Inrush: 0.450A (pico de partida)
          dI/dt = +0.150 A/s
          Status: IGNORADO (não estabilizada)

t=6s    → Corrente: 0.312A (operação normal)
          dI/dt = 0.002 A/s
          binary_sensor.bomba_corrente_estabilizada: ON
          Status: NORMAL ✅

t=3min  → Usuário fecha torneira
t=3m20s → Delay desligamento (20s)
t=3m20s → Bomba DESLIGA
```

---

## 🚀 Quick Start

### **Pré-requisitos**
- Home Assistant (2024.12+)
- Sonoff Mini (detecção de fluxo)
- Tuya TS011F com Zigbee (controle da bomba + medição de corrente)
- Bomba de circulação 50W/0.31A

### **Instalação Rápida**

1. **Criar helpers** (Configuration → Helpers):
   ```yaml
   # Input Numbers
   input_number.pump_current_normal_min: 0.303      # Mínimo normal
   input_number.pump_current_normal_max: 0.323      # Máximo normal
   input_number.pump_current_critical_max: 0.388    # Limite crítico
   input_number.pump_activation_delay_seconds: 3    # Delay ligar
   input_number.pump_deactivation_delay_seconds: 20 # Delay desligar
   input_number.pump_timeout_minutes: 30            # Timeout segurança

   # Timers
   timer.pump_activation_delay: 00:00:03
   timer.pump_deactivation_delay: 00:00:20
   timer.pump_safety_timeout: 00:30:00
   ```

2. **Copiar arquivos de config/** para seu Home Assistant:
   ```bash
   cp config/sensors.yaml /config/sensors.yaml
   cp config/template_sensors.yaml /config/template_sensors.yaml
   cp config/automations_bomba.yaml /config/automations.yaml
   cp config/scripts_bomba.yaml /config/scripts.yaml
   ```

3. **Verificar configuração e reiniciar:**
   ```bash
   ha core check
   ha core restart
   ```

4. **Instalar Dashboard Sidebar:**
   - Configurações → Dashboards → Editar Dashboard
   - + ADICIONAR VIEW (tipo: sidebar)
   - Editar em YAML Bruto → colar conteúdo de `lovelace_bomba_sidebar.yaml`
   - Salvar

**Para instalação detalhada:** Veja [CLAUDE.md](CLAUDE.md)

---

## 📂 Estrutura do Projeto

```
ha-hot-water-circulation/
├── CLAUDE.md                          # Contexto completo para IA
├── README.md                          # Este arquivo
├── CHANGELOG.md                       # Histórico de versões
│
├── config/                            # Configurações Home Assistant
│   ├── sensors.yaml                   # Derivative + Statistics (RAW)
│   ├── template_sensors.yaml          # Binary sensors v3.5 (DETECÇÃO)
│   ├── automations_bomba.yaml         # Automações completas (AÇÃO)
│   ├── scripts_bomba.yaml             # Scripts: teste, reset, relatório
│   ├── lovelace_bomba_sidebar.yaml    # Dashboard Mushroom (RECOMENDADO)
│   └── lovelace_bomba_sidebar_native.yaml # Dashboard nativo HA
│
├── docs/                              # Documentação
│   ├── decisions/
│   │   └── 0001-current-vs-power.md   # ADR: Por que corrente?
│   ├── lessons-learned.md             # Lições aprendidas
│   └── dashboard-installation-guide.md # Guia dashboard
│
└── versions/                          # Versões anteriores
    ├── V2/                            # v2.0 (potência)
    └── V3/                            # v3.0 (corrente básica)
```

---

## ✨ Funcionalidades

### **Arquitetura 3 Camadas** ⭐ (v3.5)

```yaml
┌─────────────────────────────────────────────────────────┐
│ CAMADA 1: SENSORES RAW (sensors.yaml)                  │
├─────────────────────────────────────────────────────────┤
│ sensor.bomba_taxa_mudanca_corrente        # dI/dt (A/s) │
│ sensor.bomba_corrente_media_24h           # Statistics  │
│ sensor.bomba_corrente_media_7d            # Statistics  │
│ sensor.bomba_corrente_media_30d           # Baseline    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ CAMADA 2: DETECÇÃO INTELIGENTE (template_sensors.yaml) │
├─────────────────────────────────────────────────────────┤
│ binary_sensor.bomba_corrente_estabilizada               │
│   └─ |dI/dt| < 0.005 A/s por 2s → Ignora inrush        │
│                                                          │
│ binary_sensor.bomba_mudanca_rapida_corrente             │
│   └─ |dI/dt| > 0.010 A/s → Problema agudo              │
│                                                          │
│ binary_sensor.bomba_desgaste_emergente                  │
│   └─ 7d > 30d × 1.03 → Desgaste emergente              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ CAMADA 3: ALERTAS COMBINADOS (template_sensors.yaml)   │
├─────────────────────────────────────────────────────────┤
│ binary_sensor.bomba_corrente_anormal                    │
│   └─ (estabilizada AND fora_faixa) OR mudança_rápida   │
│                                                          │
│ binary_sensor.bomba_corrente_critica                    │
│   └─ > 0.388A → EMERGÊNCIA: Desliga imediato           │
└─────────────────────────────────────────────────────────┘
```

### **Thresholds Calibrados** 📊

Baseado em **1826 amostras** coletadas em dezembro/2024:

| Parâmetro | Valor | Percentil | Ação |
|-----------|-------|-----------|------|
| **Normal Mínimo** | 0.303A | P5 | Operação saudável |
| **Normal Máximo** | 0.323A | P95 | Operação saudável |
| **Crítico** | 0.388A | Max × 1.15 | 🚨 Desligamento imediato |
| **Estabilização** | \|dI/dt\| < 0.005 A/s | - | Ignora inrush após 2s |
| **Mudança Rápida** | \|dI/dt\| > 0.010 A/s | - | ⚠️ Alerta problema agudo |
| **Desgaste** | 7d > 30d × 1.03 | - | 📈 Alerta desgaste emergente |

### **Controle Automático**
- ✅ Detecção de fluxo via Sonoff Mini
- ✅ Delay de confirmação (3s) para evitar falsos positivos
- ✅ Acionamento automático da bomba via Tuya TS011F Zigbee
- ✅ Delay de desligamento (20s) para otimizar ciclos
- ✅ Timeout de segurança (30min) para falhas
- ✅ Filtro de inrush current usando derivative

### **Alertas Inteligentes**
- ✅ Corrente anormal (fora da faixa após estabilização)
- ✅ Mudança rápida (dI/dt > 0.010 A/s)
- ✅ Desgaste emergente (média 7d crescendo vs 30d)
- ✅ Corrente crítica (>0.388A com desligamento imediato)

---

## 🎯 Hardware Suportado

### **Entidades Home Assistant**

| Entity ID | Tipo | Descrição |
|-----------|------|-----------|
| `valve.hot_water` | Sensor | Sonoff Mini - Detecção de fluxo |
| `switch.bomba_de_circulacao_de_agua_quente` | Switch | Tuya TS011F - Controle da bomba |
| `sensor.bomba_de_circulacao_de_agua_quente_corrente` | Sensor | Medição de corrente (A) |
| `sensor.bomba_de_circulacao_de_agua_quente_potencia` | Sensor | Medição de potência (W) |

### **Hardware Testado**

| Componente | Modelo | Função | Status |
|------------|--------|--------|--------|
| Sensor Fluxo | Sonoff Mini | Detecção abertura válvula | ✅ Testado |
| Smart Plug | Tuya TS011F (Zigbee) | Controle + medição corrente | ✅ Testado |
| Bomba | Circulação 50W/0.31A | Circulação água quente | ✅ Testado |

---

## ⚙️ Configuração Avançada

### **Personalizar Thresholds**

```yaml
# config/helpers/pump_thresholds.yaml

input_number:
  pump_current_normal_min:
    name: "Bomba - Corrente Mínima Normal"
    min: 0.2
    max: 0.4
    step: 0.001
    initial: 0.303  # Ajuste conforme sua bomba

  pump_current_normal_max:
    name: "Bomba - Corrente Máxima Normal"
    min: 0.2
    max: 0.4
    step: 0.001
    initial: 0.323  # Ajuste conforme sua bomba
```

### **Script TTS Personalizado**

```yaml
# Exemplo: Alerta por voz quando corrente crítica
automation:
  - alias: "Bomba - Alerta Voz Corrente Crítica"
    trigger:
      - platform: state
        entity_id: binary_sensor.bomba_corrente_critica
        to: 'on'
    action:
      - service: tts.google_translate_say
        data:
          entity_id: media_player.sala
          message: "ALERTA! Corrente da bomba está crítica. Bomba desligada automaticamente."
```

---

## 📊 Dashboards

### **Instalação Sidebar (v3.5.1)**

Dois dashboards disponíveis para adicionar à aba lateral do HA:

1. **`lovelace_bomba_sidebar.yaml`** ⭐ (RECOMENDADO)
   - Interface moderna com Mushroom Cards
   - Requer: HACS + Mushroom Cards + Mini Graph Card
   - Melhor UX mobile e desktop
   - 2 colunas grid responsivo
   - Seção DEBUG com todos os sensores raw

2. **`lovelace_bomba_sidebar_native.yaml`**
   - Usa apenas cards nativos do HA
   - Sem dependências
   - Funciona imediatamente

**Instalação rápida:**
1. Configurações → Dashboards → Editar Dashboard
2. + ADICIONAR VIEW (tipo: sidebar)
3. Editar em YAML Bruto → colar conteúdo
4. Salvar

**Guia completo:** [docs/dashboard-installation-guide.md](docs/dashboard-installation-guide.md)

---

## 📈 Estatísticas v3.5.1

| Métrica | Valor |
|---------|-------|
| **Amostras calibração** | 1826 |
| **Precisão detecção inrush** | ~100% |
| **Falsos positivos** | 0 (após v3.5) |
| **Sensores monitorados** | 13 |
| **Automações** | 6 |
| **Scripts** | 3 |
| **Economia de água** | ~30L/dia* |

_* Estimativa baseada em tempo de espera eliminado_

---

## 🗺️ Roadmap

### **v3.5.1 - Dashboard Sidebar** ✅ CONCLUÍDA (14/12/2024)
- ✅ Dashboard Mushroom Cards (2 colunas grid)
- ✅ Dashboard Native Cards (sem dependências)
- ✅ Seção DEBUG com todos os sensores
- ✅ Guia de instalação detalhado

### **v3.5 - Derivative Intelligence** ✅ CONCLUÍDA (12/12/2024)
- ✅ Derivative correto: `unit_time: s`, `time_window: 00:00:05`
- ✅ 3 binary_sensors: estabilizada, mudança_rápida, desgaste_emergente
- ✅ Lógica ignora inrush automaticamente
- ✅ Bug cooldown timer corrigido

### **v4.0 - Machine Learning** 🔮 (FUTURO)
- [ ] Predição de falhas usando histórico
- [ ] Ajuste automático de thresholds
- [ ] Detecção de padrões de uso

### **v4.5 - Integração Energética** 🔮 (FUTURO)
- [ ] Cálculo de custo por acionamento
- [ ] Otimização por tarifa horária
- [ ] Relatórios mensais de consumo

**Histórico completo:** Veja [CHANGELOG.md](CHANGELOG.md)

---

## 🔧 Troubleshooting

### **Problemas Conhecidos e Soluções**

| Problema | Causa | Solução |
|----------|-------|---------|
| Inrush dispara alerta | v3.0 ou anterior | ✅ Atualizar para v3.5 |
| Sonoff stuck em "open" | Ruído/instabilidade | Delay 3s já implementado |
| Derivative em A/min | Bug v3.0 | ✅ Corrigido v3.5: `unit_time: s` |
| Cooldown bloqueava | Lógica antiga | ✅ Removido v3.0 |

**Detalhes:** [docs/lessons-learned.md](docs/lessons-learned.md)

### **Validação Pós-Instalação**

```bash
# 1. Verificar derivative
ha state get sensor.bomba_taxa_mudanca_corrente
# Deve retornar valor em A/s (ex: 0.002)

# 2. Verificar binary sensors
ha state get binary_sensor.bomba_corrente_estabilizada
ha state get binary_sensor.bomba_mudanca_rapida_corrente
ha state get binary_sensor.bomba_desgaste_emergente

# 3. Teste funcional
# Ligar bomba manualmente
# Aguardar ~2s
# Verificar: estabilizada=on, corrente_anormal=off
```

---

## 🤝 Contribuindo

Contribuições são bem-vindas! Por favor:

1. Fork o projeto
2. Crie uma branch (`git checkout -b feature/MinhaFuncionalidade`)
3. Commit suas mudanças (`git commit -m 'Adiciona MinhaFuncionalidade'`)
4. Push para a branch (`git push origin feature/MinhaFuncionalidade`)
5. Abra um Pull Request

### **Diretrizes de Contribuição**

- ✅ Sempre teste em ambiente seguro primeiro
- ✅ Documente mudanças no CHANGELOG.md
- ✅ Adicione testes quando possível
- ✅ Siga o estilo de código existente
- ❌ NUNCA use dados placeholder - sempre valores reais

---

## 📝 Licença

Este projeto está sob a licença MIT. Veja [LICENSE](LICENSE) para mais detalhes.

---

## 🙏 Agradecimentos

- **Home Assistant** - Plataforma de automação incrível
- **Sonoff** - Hardware confiável e acessível
- **Tuya** - Smart plugs com medição de corrente
- **Comunidade HA Brasil** - Suporte e inspiração

---

## 📚 Documentação Adicional

- **[CLAUDE.md](CLAUDE.md)** - Contexto técnico completo para IA
- **[CHANGELOG.md](CHANGELOG.md)** - Histórico detalhado de versões
- **[docs/lessons-learned.md](docs/lessons-learned.md)** - Lições aprendidas
- **[docs/decisions/0001-current-vs-power.md](docs/decisions/0001-current-vs-power.md)** - ADR: Por que corrente?
- **[docs/dashboard-installation-guide.md](docs/dashboard-installation-guide.md)** - Guia dashboard

---

## 📧 Contato

**Silas Gubitoso**
- GitHub: [@silasgubi](https://github.com/silasgubi)

---

**Versão:** 3.5.1 | **Última Atualização:** 14/12/2024
**Status:** ✅ Sistema em produção com detecção de inrush inteligente

**Feito com ❤️ em São Paulo, Brasil 🇧🇷**

---

## 🎓 Por Que Este Projeto?

### **Problema Original**

Antes da automação, era necessário:
1. Abrir torneira de água quente
2. Esperar 30-60 segundos (água fria percorre tubulação)
3. Desperdiçar ~2L de água por uso
4. Frustração com água fria inesperada

### **Solução Implementada**

Com a automação:
1. Abrir torneira de água quente
2. Sistema detecta (Sonoff Mini)
3. Bomba liga automaticamente após 3s
4. Água quente em ~5 segundos
5. **Economia: ~30L água/dia + conforto**

### **Evolução Técnica**

```
v1.0 (2023) → Controle manual
v2.0 (2024) → Monitoramento por potência (48-53W)
v3.0 (2024) → Migração para corrente (0.303-0.323A)
v3.5 (2024) → Derivative inteligente (ignora inrush)
v3.5.1 (2024) → Dashboard sidebar + bug fixes
```

---

<div align="center">

### ⭐ Se este projeto ajudou você, considere dar uma estrela!

[![GitHub stars](https://img.shields.io/github/stars/silasgubi/ha-hot-water-circulation?style=social)](https://github.com/silasgubi/ha-hot-water-circulation)

</div>
