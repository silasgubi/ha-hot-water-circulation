# Sistema Automação Bomba Água Quente v2.0

Sistema inteligente para Home Assistant que aciona automaticamente bomba de circulação ao detectar fluxo de água quente, proporcionando água quente instantânea e economia de energia/água.

## 🎯 O Que Faz

- **Detecta** abertura de torneira água quente (Sonoff Mini)
- **Aciona** bomba automaticamente após 3s de confirmação
- **Circula** água quente até a torneira
- **Desliga** bomba 20s após fechar torneira
- **Protege** equipamento com múltiplas camadas de segurança
- **Monitora** consumo, ciclos e potência em tempo real

## 📊 Benefícios Reais

- Água quente **imediata** nas torneiras
- Economia de **~15 litros/dia** (sem desperdício)
- Custo operacional: **R$ 0,043/hora**
- Consumo médio: **50W** (48-53W em operação normal)
- ROI: ~6-8 meses

## 🛠️ Hardware Necessário

| Item | Modelo | Função |
|------|--------|--------|
| Sensor Fluxo | Sonoff Mini | Detectar fluxo água |
| Relé Potência | Tuya TS011F | Controlar bomba (Zigbee) |
| Bomba | Circulação 50W | Circular água quente |
| Coordinator | ConBee II/Sonoff | Comunicação Zigbee |

## 🚀 Instalação Rápida

### 1. Pré-requisitos
- Home Assistant instalado
- Integração ZHA ou Zigbee2MQTT
- Sonoff Mini pareado (eWeLink)
- Tuya TS011F pareado (Zigbee)

### 2. Criar Helpers
```bash
Settings → Helpers → Create Helper
```
Criar conforme [CLAUDE_CONTEXT.md](CLAUDE_CONTEXT.md) seção "ENTIDADES CRÍTICAS"

**Resumo:**
- 4 Timers (delays e timeouts)
- 3 Input Booleans (controles)
- 4 Counters (ciclos e eventos)
- 2 Input DateTime (timestamps)
- 9 Input Numbers (configurações)

### 3. Adicionar Template Sensors
Editar `configuration.yaml`:
```yaml
template:
  - sensor:
      # [copiar da documentação]
  - binary_sensor:
      # [copiar da documentação]

utility_meter:
  # [copiar da documentação]
```

### 4. Adicionar Automações
Editar `automations.yaml` - adicionar 6 automações da documentação

### 5. Adicionar Scripts
Editar `scripts.yaml` - adicionar 3 scripts da documentação

### 6. Validar e Reiniciar
```bash
Developer Tools → YAML → Check Configuration
Settings → System → Restart
```

### 7. Testar
```bash
Services → script.pump_system_test → Execute
```

### 8. Configurar Dashboard
```bash
# Copiar card de diagnóstico e manual
cp config/dashboards/bomba-card-diagnostico-manual.yaml /config/www/

# Editar dashboard principal
# Substituir card do manual antigo pelo novo
```

**Ou via UI:**
1. Settings → Dashboards → Bomba Água Quente → Edit
2. Deletar card "Manual do Sistema" antigo
3. Add Card → Manual
4. Cole conteúdo de `bomba-card-diagnostico-manual.yaml`
5. Save

## 🎛️ Configurações Principais

### Delays (ajustáveis via UI)
- **Ligar:** 3s (confirma fluxo real)
- **Desligar:** 20s (evita religamentos)
- **Cooldown:** 3min (proteção motor)
- **Timeout:** 30min (segurança máxima)

### Thresholds Potência (calibrados)
- **Normal:** 48-53W (83% das medições)
- **Alerta Baixo:** <40W (entupimento/ar)
- **Alerta Alto:** >60W (sobrecarga)

### Limites Operacionais
- **Max ciclos/hora:** 20
- **Max runtime/dia:** 8h
- **Tarifa energia:** R$ 0,85/kWh (configurável)

## 🛡️ Proteções Implementadas

1. **Timeout Safety:** desliga após 30min se esquecer ligada
2. **Limite Ciclos:** máximo 20 ciclos/hora (proteção motor)
3. **Cooldown:** 3min entre ciclos (resfriamento)
4. **Override Manual:** desabilita automação para manutenção
5. **Reset Emergência:** botão desliga tudo imediatamente
6. **Alertas Potência:** notifica se consumo anormal

## 📱 Interface Dashboard

Cards disponíveis:
- **Status Principal:** estado em tempo real
- **Controles:** override, manual, testes, emergência
- **Estatísticas:** ciclos, runtime, custo, eficiência
- **Configurações:** todos parâmetros editáveis via UI
- **Gráficos:** potência e histórico 24h

## 📱 Interface Dashboard v2.1

### Novidades
- **🧪 Diagnóstico Automático:** Analisa e explica estado do sistema em tempo real
- **⚡ Ação Rápida:** Botões de acesso rápido (logs, automações, relatório, trigger)
- **📖 Manual Reformatado:** Tabelas visuais, fluxo de funcionamento, troubleshooting detalhado

### Cards Disponíveis

| Card | Descrição | Tipo |
|------|-----------|------|
| **Status Principal** | Estado + gauge potência | Mantido |
| **Controles** | 7 botões (override, manual, teste, reset + 3 novos) | Expandido |
| **Diagnóstico** | Análise inteligente do estado atual | 🆕 NOVO |
| **Ação Rápida** | Forçar trigger, logs, automações, relatório | 🆕 NOVO |
| **Status Tempo Real** | Entities com estados | Mantido |
| **Proteções** | Timers e limites | Mantido |
| **Configurações** | Input numbers editáveis | Mantido |
| **Automações** | Lista com last-triggered | Mantido |
| **Manual** | Guia completo reformatado | 🔄 REFORMULADO |
| **Histórico** | Gráfico 24h | Mantido |

### Funcionalidades do Diagnóstico

✅ Detecta e explica por que bomba está em cada estado  
✅ Verifica pré-requisitos automaticamente (override, limites, cooldown)  
✅ Análise de potência com faixas calibradas em tabela  
✅ Status de todos os timers em tempo real  
✅ Lista automações e seus estados  
✅ Estatísticas consolidadas do dia  
✅ Últimos eventos (ativação, timeout)  
✅ Atualização automática sem reload  

### Como Acessar

1. Sidebar → Bomba Água Quente
2. Card "Diagnóstico do Sistema" mostra análise automática
3. Botões de ação rápida abaixo do diagnóstico
4. Scroll para baixo → Manual completo reformatado

### Arquivo de Configuração

Dashboard completo em: `config/dashboards/bomba-card-diagnostico-manual.yaml`

## 🔧 Manutenção

### Diária
- Verificar contador timeouts (deve ser zero)
- Monitorar alertas potência

### Semanal
- Analisar eficiência (min/ciclo)
- Revisar consumo energético

### Mensal
- Executar `script.pump_system_test`
- Backup configurações
- Limpar logs antigos

## 🐛 Troubleshooting

### Bomba não liga
```yaml
# Debug via Developer Tools → Template
Fluxo: {{ states('valve.hot_water') }}
Override: {{ states('input_boolean.pump_manual_override') }}
Ciclos: {{ states('counter.pump_hourly_cycles') }}
Cooldown: {{ states('timer.pump_anti_cycle_cooldown') }}
```

**Soluções:**
- Desativar override
- Aguardar cooldown
- Verificar limite ciclos

### Bomba não desliga
1. Usar `script.pump_emergency_reset`
2. Verificar estado válvula
3. Revisar logs

### Potência anormal
- **<40W:** limpar tubulação, purgar ar
- **>60W:** verificar motor, sobrecarga elétrica

## 📚 Documentação Completa

- **[CLAUDE_CONTEXT.md](CLAUDE_CONTEXT.md)** - contexto técnico completo para IA
- **[CHANGELOG.md](CHANGELOG.md)** - histórico de versões
- **[docs/documentacao-completa-bomba.md](docs/documentacao-completa-bomba.md)** - manual detalhado
- **[config/](config/)** - arquivos de configuração (automations, scripts, configuration)

## 🔗 Links Úteis

- [Home Assistant](https://www.home-assistant.io/)
- [Integração eWeLink](https://github.com/AlexxIT/SonoffLAN)
- [ZHA](https://www.home-assistant.io/integrations/zha/)
- [Zigbee2MQTT](https://www.zigbee2mqtt.io/)

## 📊 Dados Calibrados (Real)

Sistema calibrado com **143 medições** reais:
- Média: 50.8W
- Mediana: 51W
- Desvio: ±3.2W
- Range normal: 48-53W

## ⚠️ Avisos Importantes

- **NUNCA** use dados placeholder/fictícios nas entidades
- **SEMPRE** baseie-se nas medições reais do seu sistema
- **Configure** alertas para eventos críticos
- **Mantenha** backup das configurações
- **Documente** mudanças realizadas

## 🤝 Contribuindo

1. Fork do projeto
2. Crie branch para feature
3. Commit com mensagens claras
4. Abra Pull Request

## 📝 Licença

MIT - Livre para uso e modificação

## 👤 Autor

Silas Gubitoso

---

**Versão:** 2.1.0  
**Última Atualização:** 2025-10-22  
**Status:** Produção ✅
