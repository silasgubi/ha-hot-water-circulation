# Changelog - Sistema Automação Bomba Água Quente

Todas as mudanças notáveis do projeto serão documentadas aqui.

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/)  
Versionamento: [Semantic Versioning](https://semver.org/lang/pt-BR/)

---

## [2.1.0] - 2025-10-22

### Added
- Card de diagnóstico automático com análise inteligente de estado
- Botões de ação rápida (Forçar Trigger, Ver Logs, Ver Automações, Relatório)
- Manual reformatado com tabelas visuais e troubleshooting detalhado
- Análise de potência em tempo real com faixas calibradas
- Status de automações no diagnóstico
- Seção "Como Funciona" com fluxo visual no manual

### Changed
- Layout do dashboard reorganizado para melhor usabilidade
- Manual convertido de texto corrido para tabelas escaneáveis
- Troubleshooting agora organizado por problema específico
- Faixas de potência destacadas com emojis visuais

### Improved
- Diagnóstico agora explica POR QUE bomba está em determinado estado
- Verificação automática de pré-requisitos
- Feedback visual mais claro (cores, emojis, tabelas)
- Acesso rápido a logs e automações via botões

## [2.0.1] - 2024-01-XX

### Alterado
- Dashboard otimizado para visualização em iPad
- Layout responsivo para cards de status
- Melhorias na disposição de elementos na interface

---

## [2.0.0] - 2024-01-XX

### Adicionado
- **9 Input Numbers configuráveis** para ajuste dinâmico via UI:
  - Thresholds de potência (baixo, alto, normal min/max)
  - Limites operacionais (runtime máximo, ciclos/hora)
  - Configurações de tempo (timeout, cooldown, delays)
  - Tarifa de energia personalizável
- **4 Timers dinâmicos** com duração configurável por input_numbers
- **Template sensors avançados**:
  - `sensor.bomba_runtime_hoje` (cálculo histórico)
  - `sensor.bomba_eficiencia` (minutos/ciclo)
  - `sensor.bomba_custo_hoje` (R$ com tarifa configurável)
  - `sensor.bomba_sistema_status` (estado inteligente)
- **Binary sensors de proteção**:
  - `binary_sensor.bomba_funcionando` (detecção real via potência)
  - `binary_sensor.bomba_modo_seguro` (verifica todas condições)
  - `binary_sensor.bomba_potencia_anormal` (alertas inteligentes)
- **Scripts de manutenção**:
  - `pump_system_test` (teste completo automatizado)
  - `pump_emergency_reset` (reset emergência com logs)
  - `pump_detailed_report` (relatório completo via notificação)
- **Automação de relatório diário** (22:00) com estatísticas
- **Utility meters** para tracking de consumo (diário/semanal/mensal)
- **Calibração baseada em dados reais** (143 medições):
  - Faixa normal: 48-53W (83% dos casos)
  - Thresholds científicos baseados em P10/P90

### Modificado
- Lógica de delays agora usa `input_number` em vez de valores fixos
- Automação principal refatorada com condições melhoradas
- Timers com duração dinâmica calculada em tempo real
- Sistema de logs expandido com mais detalhes
- Contadores com limites realistas ajustados

### Melhorado
- Performance do sistema reduzindo chamadas desnecessárias
- Confiabilidade com mais verificações de estado
- Dashboard mais intuitivo e informativo
- Documentação técnica completa gerada

### Corrigido
- Race conditions em acionamentos rápidos
- Problema de timers não cancelando corretamente
- Alertas de potência falso-positivos
- Reset de contadores horários não consistente

---

## [1.0.0] - 2024-01-XX

### Adicionado
- Implementação inicial do sistema de automação
- Detecção de fluxo via Sonoff Mini (`valve.hot_water`)
- Controle de bomba via Tuya TS011F Zigbee
- Monitoramento de potência em tempo real
- Sistema básico de proteções:
  - Timeout fixo de 30 minutos
  - Cooldown fixo de 3 minutos
  - Limite de 20 ciclos por hora
- Delays fixos:
  - Ativação: 3 segundos
  - Desativação: 20 segundos
- Contadores básicos:
  - Ciclos horários
  - Ciclos diários
  - Eventos de timeout
- Dashboard inicial com cards de status
- Automação principal de controle
- Override manual básico
- Logs no logbook para eventos principais

### Hardware Integrado
- Sonoff Mini (eWeLink)
- Tuya TS011F (Zigbee via ZHA)
- Bomba de circulação 50W 220V

### Especificações Técnicas
- Consumo nominal: 50W
- Tensão: 220V
- Corrente: 0.25A
- Custo operacional: ~R$ 0,043/hora (tarifa R$ 0,85/kWh)

---

## Formato das Entradas

### [X.Y.Z] - AAAA-MM-DD

#### Adicionado
- Novas funcionalidades

#### Modificado
- Mudanças em funcionalidades existentes

#### Descontinuado
- Funcionalidades que serão removidas

#### Removido
- Funcionalidades removidas

#### Corrigido
- Correções de bugs

#### Segurança
- Correções de vulnerabilidades

---

## Planejamento Futuro

### [2.1.0] - Planejado
- Integração com assistente de voz (Google/Alexa)
- Previsão de manutenção via Machine Learning
- Detecção automática de anomalias
- Dashboard mobile dedicado

### [2.2.0] - Planejado
- Integração com tarifa dinâmica de energia
- Otimização de horários de funcionamento
- Histórico em banco de dados externo (InfluxDB)
- API REST para integração externa

### [3.0.0] - Planejado
- Suporte multi-bomba
- Sistema de profiles (economia/conforto/balanceado)
- App mobile nativo
- Integração com sistema solar fotovoltaico

---

**Legenda Tipos de Mudança:**
- 🆕 Adicionado
- 🔄 Modificado
- ⚡ Melhorado
- 🐛 Corrigido
- ⚠️ Descontinuado
- 🗑️ Removido
- 🔒 Segurança

---

**Manutenção:** Este arquivo deve ser atualizado a cada mudança implementada no sistema.
