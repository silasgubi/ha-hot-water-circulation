# Sistema Automação Bomba Água Quente v3.5

Sistema inteligente para Home Assistant que aciona automaticamente bomba de circulação ao detectar fluxo de água quente, com detecção inteligente de inrush current via derivative.

## O Que Faz

- **Detecta** abertura de torneira água quente (Sonoff Mini)
- **Aciona** bomba automaticamente após 3s de confirmação
- **Monitora** corrente em tempo real (0.303-0.323A normal)
- **Ignora** pico de partida (inrush) automaticamente
- **Alerta** mudanças rápidas e desgaste emergente
- **Protege** com desligamento crítico (>0.388A)

## Novidades v3.5.1

- **Dashboard Lovelace**: Layout grid 2 colunas com secao DEBUG
- **Bug fix**: Condicao cooldown timer removida (bloqueava automacao)
- **Encoding**: UTF-8 corrigido em todos os arquivos YAML
- **Derivative corrigido**: `unit_time: s` (era min), `time_window: 5s` (era 5min)
- **3 novos sensores**: Estabilizacao, Mudanca Rapida, Desgaste Emergente
- **Inrush ignorado**: Logica `estabilizada AND fora_faixa` evita falsos positivos

## Hardware

| Item | Modelo | Função |
|------|--------|--------|
| Sensor Fluxo | Sonoff Mini | Detectar fluxo água |
| Relé | Tuya TS011F | Controlar bomba (Zigbee) |
| Bomba | Circulação 50W | Circular água quente |

## Thresholds Calibrados

Baseado em **1826 amostras** (dezembro 2024):

| Parâmetro | Valor | Significado |
|-----------|-------|-------------|
| Normal | 0.303-0.323A | Operação saudável |
| Atenção | 0.323-0.388A | Monitorar |
| Crítico | >0.388A | Desliga imediato |

## Instalacao

1. **Backup** dos arquivos atuais
2. **Copiar** arquivos de `config/` para `/config/`
3. **Criar** input_numbers e timers (ver CLAUDE.md)
4. **Reiniciar** Home Assistant

## Estrutura

```
├── CLAUDE.md                      # Contexto para IA (Claude Code)
├── README.md                      # Este arquivo
├── CHANGELOG.md                   # Historico de versoes
├── config/
│   ├── sensors.yaml               # Derivative + Statistics
│   ├── template_sensors.yaml      # Binary sensors v3.5
│   ├── automations_bomba.yaml     # Automacoes completas
│   ├── scripts_bomba.yaml         # Teste, reset, relatorio
│   └── dashboard_bomba_v35.yaml   # Dashboard Lovelace v3.5
└── docs/
    ├── decisions/                 # ADRs
    └── lessons-learned.md         # Licoes aprendidas
```

## Documentacao

- **[CLAUDE.md](CLAUDE.md)** - Contexto técnico completo
- **[CHANGELOG.md](CHANGELOG.md)** - Histórico de versões
- **[docs/lessons-learned.md](docs/lessons-learned.md)** - Lições aprendidas

## Avisos

- **NUNCA** use dados placeholder - sempre valores reais
- **SEMPRE** faça backup antes de atualizar
- **TESTE** em ambiente seguro primeiro

## Licenca

MIT - Livre para uso e modificação

---

**Versao**: 3.5.1
**Ultima Atualizacao**: Dezembro 2025
**Status**: Em Producao
