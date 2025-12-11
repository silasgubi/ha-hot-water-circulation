# Git Workflow - Migração v3.0
## Branch Strategy para Teste Seguro

---

## 📋 Visão Geral

Este guia detalha como usar Git para testar v3.0 em paralelo com v2.0 estável, permitindo rollback fácil se necessário.

---

## 🌳 Estrutura de Branches

```
main (v2.0 estável)
  │
  └─── v3.0-corrente-migration (branch de desenvolvimento)
         │
         ├─ configuration.yaml (v3.0)
         ├─ automations.yaml (v3.0)
         ├─ scripts.yaml (v3.0)
         └─ helpers criados via UI
```

---

## 🚀 Passo a Passo Completo

### FASE 1: Preparação

#### 1.1 Verificar estado atual
```bash
cd /config  # ou seu caminho para config HA

# Verificar branch atual
git branch
# Deve mostrar: * main

# Verificar status
git status
# Commitar qualquer mudança pendente se necessário
```

#### 1.2 Snapshot do estado v2.0
```bash
# Adicionar tudo ao staging
git add -A

# Commitar estado atual
git commit -m "v2.0.1: Snapshot estável antes migração v3.0

- Sistema funcionando 100%
- 22 helpers configurados
- Monitoramento por potência
- Consumo via utility_meters
- Base para rollback se necessário"

# Confirmar commit
git log --oneline -1
```

---

### FASE 2: Criar Branch v3.0

#### 2.1 Criar e mudar para branch
```bash
# Criar branch a partir da main
git checkout -b v3.0-corrente-migration

# Confirmar mudança
git branch
# Deve mostrar: * v3.0-corrente-migration
```

#### 2.2 Tag opcional (milestone)
```bash
# Criar tag para fácil referência
git tag v2.0.1-stable

# Verificar tags
git tag
```

---

### FASE 3: Implementar Mudanças

#### 3.1 Criar helpers via UI
```yaml
# Settings → Helpers → Create Helper
# (não precisa commitar - HA faz automático)

✅ input_number.pump_current_threshold_low
✅ input_number.pump_current_threshold_high
✅ input_number.pump_current_normal_min
✅ input_number.pump_current_normal_max
✅ counter.pump_current_alerts
```

#### 3.2 Atualizar configuration.yaml
```bash
# Fazer backup primeiro
cp configuration.yaml configuration.yaml.v2.0.bak

# Editar configuration.yaml
# (substituir template: e adicionar sensors)

# Verificar mudanças
git diff configuration.yaml

# Stagear mudanças
git add configuration.yaml

# Commit
git commit -m "feat(config): migração para monitoramento por corrente

- Adicionado template sensors para corrente
- Adicionado history_stats para runtime preciso
- Adicionado statistics sensors (média/min/max)
- Sensores de energia calculados corretamente
- Atualizado binary_sensor.bomba_funcionando (corrente > 0.24A)
- Adicionado binary_sensor.bomba_corrente_anormal

BREAKING CHANGE: monitoramento agora usa corrente ao invés de potência"
```

#### 3.3 Atualizar automations.yaml
```bash
# Backup
cp automations.yaml automations.yaml.v2.0.bak

# Editar automations.yaml
# (trocar condições de potência por corrente)

# Commit
git add automations.yaml
git commit -m "feat(automations): atualizar para usar corrente

- Condições baseadas em corrente ao invés de potência
- Nova automação: bomba_alerta_corrente_anormal
- Relatório diário inclui métricas de corrente
- Mantém toda lógica de proteções existente"
```

#### 3.4 Atualizar scripts.yaml
```bash
# Backup
cp scripts.yaml scripts.yaml.v2.0.bak

# Editar scripts.yaml
# (atualizar relatórios + adicionar calibração)

# Commit
git add scripts.yaml
git commit -m "feat(scripts): adicionar calibração e atualizar relatórios

- Novo script: pump_calibrate_current (30s)
- pump_system_test agora mede corrente
- pump_detailed_report inclui análise completa corrente
- Relatórios mostram thresholds calibrados"
```

#### 3.5 Atualizar documentação
```bash
# Adicionar novos docs
git add CLAUDE_CONTEXT.md CHANGELOG.md migration-guide-v2-to-v3.md

git commit -m "docs: adicionar documentação v3.0

- CLAUDE_CONTEXT.md v3.0 (migração corrente)
- CHANGELOG.md v3.0 (breaking changes)
- migration-guide-v2-to-v3.md (guia completo)
- Análise estatística 1365 medições
- Thresholds calibrados com dados reais"
```

---

### FASE 4: Testar v3.0

#### 4.1 Restart Home Assistant
```bash
# Via UI: Developer Tools → YAML → Restart

# Aguardar restart completo (~2-3 min)
```

#### 4.2 Validar instalação
```bash
# Executar testes
# (via UI: Services → script.pump_system_test)

# Ver logs
# (Settings → System → Logs)

# Commit resultados
git add -A
git commit -m "test: validação inicial v3.0

- Teste funcional: PASSOU ✅
- Sensores criados: 12/12 ✅
- Statistics funcionando ✅
- Corrente medida: 0.30-0.32A ✅"
```

#### 4.3 Calibrar sistema
```bash
# Executar calibração
# (Services → script.pump_calibrate_current)

# Anotar recomendações
# Ajustar thresholds via UI se necessário

# Commit calibração
git add -A
git commit -m "config: calibração inicial

- Corrente média: 0.309A
- Normal: 0.303-0.323A
- Thresholds ajustados conforme calibração"
```

---

### FASE 5: Período de Teste (24h)

#### 5.1 Monitorar operação
```bash
# Verificar diariamente:
# - Ciclos funcionando?
# - Corrente estável?
# - Alertas corretos?

# Commitar observações
git commit --allow-empty -m "test: dia 1 de testes

- Ciclos: 15/15 OK ✅
- Corrente: 0.303-0.323A estável ✅
- Consumo: R$ 0.20 (coerente) ✅
- Alertas: 0 falsos positivos ✅"
```

#### 5.2 Ajustes finos (se necessário)
```bash
# Se precisar ajustar algo
git add <arquivos>
git commit -m "fix: ajuste fino thresholds

- Normal max: 0.323 → 0.325A
- Motivo: 2% medições em 0.324A"
```

---

### FASE 6: Decisão Final

#### ✅ **OPÇÃO A: Aprovar v3.0 (sucesso)**

```bash
# Mergear na main
git checkout main
git merge v3.0-corrente-migration

# Verificar merge
git log --oneline -5

# Tag release
git tag v3.0.0 -m "Release v3.0.0 - Migração Corrente

- Monitoramento por corrente
- Consumo correto via Statistics
- Thresholds calibrados (1365 medições)
- Calibração automática
- 100% testado e validado"

# Push (se usar repositório remoto)
git push origin main
git push origin v3.0.0

# Limpar branch (opcional)
git branch -d v3.0-corrente-migration
```

#### ❌ **OPÇÃO B: Rollback (problemas)**

```bash
# Voltar para main (v2.0)
git checkout main

# Deletar branch problemática
git branch -D v3.0-corrente-migration

# Confirmar versão
git log --oneline -1
# Deve mostrar último commit v2.0

# Restart HA para voltar v2.0
# (Developer Tools → YAML → Restart)

# Remover helpers v3.0 via UI
# Settings → Helpers → Delete:
# - pump_current_threshold_low
# - pump_current_threshold_high
# - pump_current_normal_min
# - pump_current_normal_max
# - pump_current_alerts

# Sistema volta para v2.0 100% ✅
```

---

## 📊 Status dos Arquivos

### Antes da Migração (main)
```bash
git log --oneline -3
# abc1234 v2.0.1: Snapshot estável antes migração v3.0
# def5678 feat: dashboard otimizado iPad
# ghi9012 v2.0.0: valores configuráveis
```

### Durante Testes (branch)
```bash
git checkout v3.0-corrente-migration
git log --oneline -5
# xyz1111 test: dia 1 de testes
# xyz2222 config: calibração inicial
# xyz3333 test: validação inicial v3.0
# xyz4444 docs: adicionar documentação v3.0
# xyz5555 feat(scripts): adicionar calibração
```

### Após Merge (main atualizada)
```bash
git checkout main
git log --oneline -5
# xyz1111 test: dia 1 de testes
# xyz2222 config: calibração inicial
# xyz3333 test: validação inicial v3.0
# xyz4444 docs: adicionar documentação v3.0
# xyz5555 feat(scripts): adicionar calibração
```

---

## 🔍 Comandos Úteis

### Ver diferenças entre branches
```bash
# Ver arquivos mudados
git diff main v3.0-corrente-migration --name-only

# Ver conteúdo específico
git diff main v3.0-corrente-migration -- configuration.yaml
```

### Ver histórico completo
```bash
# Log gráfico
git log --oneline --graph --all

# Commits só da branch
git log main..v3.0-corrente-migration
```

### Comparar tags
```bash
# Diferença v2.0.1 → v3.0.0
git diff v2.0.1-stable v3.0.0 --stat
```

---

## 💾 Backups Adicionais

Além do Git, manter:

```bash
# Snapshot Home Assistant
# Settings → System → Backups → Create

# Export helpers
# Settings → Helpers → (menu) → Download

# Backup manual arquivos críticos
tar -czf backup-v2.0-$(date +%Y%m%d).tar.gz \
  configuration.yaml \
  automations.yaml \
  scripts.yaml
```

---

## 📝 Notas Importantes

1. **Branch é local:** Mudanças só afetam seu HA quando você edita arquivos
2. **Helpers persistem:** Criados via UI, não dependem de branch
3. **Restart necessário:** HA lê arquivos no boot, então precisa restart após trocar branch
4. **Rollback limpo:** Git garante código v2.0, mas precisa remover helpers v3.0 via UI

---

## ✅ Checklist Git

```bash
☐ Status inicial limpo (git status)
☐ Commit v2.0 estável criado
☐ Tag v2.0.1-stable criada
☐ Branch v3.0-corrente-migration criada
☐ Helpers v3.0 criados via UI
☐ configuration.yaml atualizado e commitado
☐ automations.yaml atualizado e commitado
☐ scripts.yaml atualizado e commitado
☐ Documentação commitada
☐ Teste funcional passou
☐ Calibração executada
☐ 24h de testes monitorados
☐ Decisão: merge ou rollback
☐ Tag v3.0.0 criada (se sucesso)
☐ Branch limpa (opcional)
```

---

**Pronto para começar?** 🚀

```bash
cd /config
git status  # Verificar
git add -A  # Stagear
git commit -m "v2.0.1: Snapshot antes v3.0"  # Commitar
git checkout -b v3.0-corrente-migration  # Branch
# ... seguir FASE 3 em diante ...
```

---

**Versão:** 1.0  
**Data:** {{now().strftime('%d/%m/%Y')}}  
**Comando rápido:** `cat git-workflow-v3.md | grep "^git "`
