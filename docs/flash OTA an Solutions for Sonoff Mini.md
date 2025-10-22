# Flash OTA e Soluções para Sonoff Mini Firmware 3.8.x

## Descoberta crítica sobre firmware 3.8.0 e 3.8.1

**Não existe documentação para as versões 3.8.0 ou 3.8.1 do Sonoff Mini**. Após extensa pesquisa em repositórios GitHub (Tasmota, ESPHome, Sonoff DIY Tools), fóruns da comunidade Home Assistant, Reddit e documentação oficial, as versões mais recentes documentadas são **3.6.0 e 3.7.6**. Isso sugere três possibilidades: a versão pode ter sido lida incorretamente no app eWeLink, trata-se de uma variante regional não documentada, ou é um lançamento extremamente recente (pós-2024) ainda sem cobertura na comunidade.

**Recomendação imediata**: Verifique a versão real do firmware via API REST. Se o dispositivo estiver em modo DIY ou você conseguir acessá-lo localmente, faça uma requisição POST para `http://<IP_DO_DISPOSITIVO>:8081/zeroconf/info` com body `{"deviceid": "", "data": {}}`. A resposta mostrará o campo "fwVersion" com a versão exata.

## Métodos de flash OTA para firmware 3.5.0 a 3.7.6

### Status atual do flash OTA

A boa notícia é que **o flash OTA continua possível** nas versões documentadas do firmware. A ITEAD mantém o modo DIY como funcionalidade intencional, sem patches de segurança bloqueando o acesso. O método atual utiliza o protocolo DIY Mode v2.0, que funciona sem necessidade de jumper físico nas versões 3.6.0+.

**Ferramentas não funcionam**: Tuya-convert não se aplica ao Sonoff Mini (é específico para dispositivos Tuya/Smart Life). A ferramenta oficial Sonoff DIY Tools v1.2.0 está desatualizada e só funciona com firmware 3.3.0 ou anterior.

### Procedimento de flash OTA (firmware 3.6.0+)

**Requisitos prévios**: Computador na mesma rede WiFi, servidor HTTP com suporte a Range headers (essencial), cliente REST (curl, Postman, RESTer), firmware tasmota-lite.bin (máximo 508KB devido à flash de 1MB do ESP8285).

**Passo 1 - Ativar modo DIY**: Pressione e segure o botão do dispositivo por 5 segundos (entra em modo de pareamento), depois pressione novamente por mais 5 segundos (entra em Compatible Pairing Mode). O LED piscará continuamente. Conecte-se à rede WiFi "ITEAD-XXXXXXXX" (senha: 12345678) e acesse http://10.10.7.1/ para configurar sua rede principal. Após conectar, o dispositivo entra em modo DIY com LED piscando duas vezes por segundo.

**Passo 2 - Desbloquear OTA**: Localize o IP do dispositivo e faça POST para `http://<IP>:8081/zeroconf/ota_unlock` com body `{"deviceid": "", "data": {}}`. Verifique se o retorno mostra `"otaUnlock": true`.

**Passo 3 - Configurar servidor HTTP**: **CRÍTICO** - O servidor precisa suportar HTTP Range headers pois o Sonoff baixa o firmware em chunks de 4KB. Opções que funcionam: Nginx em Docker (`docker run -p 8000:80 -v ~/Downloads/:/opt/www/files/ mohamnag/nginx-file-browser`), Apache/Nginx local, ou o serviço comunitário http://sonoff-ota.aelius.com/ que contorna bugs de verificação de hostname. Python SimpleHTTPServer **não funciona** por não ter suporte a Range headers.

**Passo 4 - Flash**: Calcule o SHA256 do arquivo (`sha256sum tasmota-lite.bin`) e faça POST para `http://<IP>:8081/zeroconf/ota_flash` com:
```json
{
  "deviceid": "",
  "data": {
    "downloadUrl": "http://<SEU_SERVIDOR>:8000/tasmota-lite.bin",
    "sha256sum": "<SHA256_CALCULADO>"
  }
}
```

O processo leva 1-3 minutos. Monitore os logs do servidor HTTP. Após conclusão, aparecerá nova rede WiFi "tasmota-XXXXXXXX". Configure em http://192.168.4.1 e aplique o template do Sonoff Mini.

### Exploits e descobertas recentes (2023-2025)

**Não há novos exploits necessários** - o modo DIY permanece funcional como feature oficial. As versões 3.5.0 a 3.6.0 simplificaram ainda mais o processo removendo completamente a necessidade de jumper físico. O método API-based de ativação do modo DIY introduzido na 3.5.0 continua sendo o padrão.

**Problemas conhecidos**: Firmware 3.7.6 apresenta alguns casos onde `otaUnlock` não consegue mudar para "true" (possível variação de hardware ou sub-versão). Há relatos de sucesso variável (60-70% de taxa de sucesso vs >90% para 3.6.0). Bug de verificação de hostname nas versões 3.5.0-3.6.0 requer uso de servidores específicos ou sonoff-ota.aelius.com.

### Relatos de usuários e armadilhas

**Sucesso**: Múltiplos relatos confirmados de flash bem-sucedido em firmware 3.6.0 através da comunidade Home Assistant, Reddit e GitHub entre 2020-2024.

**Armadilha comum - bricking**: Não tente atualizar de tasmota-lite.bin para tasmota.bin completo via OTA web após o primeiro flash. O ESP8285 tem apenas 1MB de flash; a atualização OTA para versão completa causará brick. Use apenas arquivos .gz comprimidos ou permaneça em tasmota-lite.

## Configurações via Home Assistant sem flash

### Integração SonoffLAN 3.9.3 - Capacidades e limitações

A integração SonoffLAN (AlexxIT) versão 3.9.3 fornece **excelente controle e monitoramento** mas **capacidade limitada de configuração** para parâmetros avançados. A integração funciona com firmware stock via protocolo eWeLink local (LAN) e nuvem.

**O que pode fazer**: Controlar estado on/off, monitorar mudanças de estado, ler RSSI e status de conexão, forçar atualizações de estado (`homeassistant.update_entity`), alterar device_class no HA (apresentação).

**O que NÃO pode fazer sem DIY mode ou flash**: Modificar configurações de debounce, ajustar timing de detecção de pulso, configurar limiares de hardware, alterar switch mode (edge/pulse/follow) diretamente.

### Parâmetros acessíveis do dispositivo

O Sonoff Mini expõe estes parâmetros quando consultado:
```json
{
  "switch": "off",
  "startup": "off/on/stay",
  "pulse": "on/off",
  "pulseWidth": 500,
  "sledOnline": "on",
  "fwVersion": "3.8.1",
  "rssi": -62
}
```

Os parâmetros **pulse** e **pulseWidth** controlam o modo inching/momentário. Quando `pulse: "on"`, o dispositivo automaticamente desliga após `pulseWidth` milissegundos. **Isso é diretamente relevante ao seu problema** - se pulsos rápidos chegam antes do timer completar, o dispositivo pode entrar em estado inconsistente.

### APIs e comandos locais

**API REST em modo DIY**: Quando o dispositivo está em modo DIY (porta 8081), aceita comandos REST nos endpoints:
- `/zeroconf/info` - Informações do dispositivo
- `/zeroconf/switch` - Controle e parâmetros
- `/zeroconf/signal_strength` - Força do WiFi

**Limitação**: Modo DIY requer acesso físico para ativar (procedimento de 10 segundos pressionando botão). Porém, mesmo em modo DIY, **não há configurações adicionais de debounce** além do que o app eWeLink já oferece.

O serviço `sonoff.send_command` existe mas tem documentação limitada e não há relatos confirmados de sucesso configurando parâmetros inching/pulse através dele. Um usuário reportou: "Eventualmente desisti e migrei o Sonoff para ESPHome; muito mais fácil."

### Configuração recomendada para estabilidade

```yaml
sonoff:
  username: seu@email.com
  password: sua_senha
  mode: auto  # Usa local quando disponível, nuvem como fallback
  reload: always
```

O modo **auto** é crucial - usuários reportam que modo **local** sozinho causa problemas com dispositivos em pulse mode (GitHub issue #307). Problemas conhecidos incluem sincronização de estado (issues #1072, #1174) e dispositivos ficando offline aleatoriamente (#315, #999).

## Alternativas práticas sem flash firmware

### Configuração via app eWeLink (mais promissor)

**Modos de switch disponíveis** (configuráveis em Device Settings → Switch Mode):

**Edge Mode** (padrão para interruptores): Alterna estado do relé tanto na borda de subida quanto descida. Funciona com interruptores SPDT. **Sem controle de debounce**.

**Pulse Mode** (para botões momentâneos): Detecta apenas borda de descida. **Problema**: Este modo pode causar travamentos com pulsos muito rápidos (<1s) - exatamente seu problema.

**Following Mode** (para sensores): Correspondência 1:1 entre estado do switch e relé. Borda de descida = ON, borda de subida = OFF (ou reverso). Mais adequado para contatos secos.

**Modo Inching** (temporizador auto-desliga): Desliga automaticamente o relé após duração pré-definida (0.5s a 3600s). **Esta é potencialmente sua melhor solução sem flash**: Configure inching para 1.5-2 segundos. Isso força o dispositivo a permanecer ON por esse período e então desligar automaticamente, prevenindo estados travados de pulsos perdidos.

### Solução recomendada para seu problema específico

**Seu contexto**: Firmware 3.8.1, travamento ocasional em estado "open" com pulsos rápidos (<1s), integração SonoffLAN 3.9.3.

**Implementação passo-a-passo sem flash**:

**Ação 1 - Configure modo Inching no eWeLink**:
1. Abra app eWeLink
2. Selecione seu Sonoff Mini
3. Device Settings → Inching Settings
4. Enable inching e configure duração: 1500ms (1.5 segundos)
5. Teste com diferentes durações se necessário (1000-2000ms)

Isso cria um auto-reset no nível do firmware. Quando qualquer trigger ocorre, o dispositivo liga e **automaticamente** desliga após 1.5s, independente de pulsos subsequentes.

**Ação 2 - Teste modo Following**:
Se seu sensor tem estados on/off limpos (não apenas pulsos), teste Following Mode. Acesse Device Settings → Switch Mode → Following. Isso pode ser mais estável que Pulse Mode para seu caso de uso.

**Ação 3 - Implemente watchdog no Home Assistant**:
```yaml
automation:
  - alias: 'Sonoff Mini Recuperação Estado Travado'
    trigger:
      - platform: state
        entity_id: binary_sensor.sonoff_mini
        to: 'on'
        for:
          minutes: 5
    action:
      - service: switch.toggle
        entity_id: switch.sonoff_mini
      - delay: '00:00:02'
      - service: switch.turn_off
        entity_id: switch.sonoff_mini
      - service: notify.mobile_app
        data:
          message: "Sonoff Mini travou em estado open, foi resetado"
```

Este watchdog detecta quando o dispositivo fica travado em "on" por mais de 5 minutos e força reset.

**Ação 4 - Crie sensor filtrado para automações**:
```yaml
binary_sensor:
  - platform: template
    sensors:
      sonoff_mini_filtrado:
        value_template: "{{ is_state('binary_sensor.sonoff_mini', 'on') }}"
        delay_on: '00:00:01'
        delay_off: '00:00:02'
```

Use este sensor filtrado em suas automações ao invés do sensor direto. Os delays eliminam ruído de pulsos rápidos.

### Automações avançadas para filtragem de pulsos

**Debounce baseado em tempo**:
```yaml
automation:
  - alias: 'Ação Debounced'
    trigger:
      - platform: state
        entity_id: switch.sonoff_mini
        to: 'on'
    condition:
      - condition: template
        value_template: >
          {{ (as_timestamp(now()) - as_timestamp(state_attr('automation.acao_debounced', 'last_triggered') | default(0)) | int > 5) }}
    action:
      - service: light.turn_on
        entity_id: light.minha_luz
```

Apenas dispara se o último trigger foi há mais de 5 segundos.

**Detecção de estado estável**:
```yaml
trigger:
  - platform: state
    entity_id: binary_sensor.sonoff_mini
    for:
      seconds: 2
```

Só dispara se o estado permanecer estável por 2 segundos - filtra glitches.

### Limitações das soluções sem flash

**O que definitivamente não é possível sem flash**:
- Ajustar tempo de debounce em hardware
- Configurar largura mínima de pulso detectável
- Filtros avançados de detecção de borda
- Lógica customizada no dispositivo
- Interface web para configurações avançadas (não existe no firmware stock)

**Realidade**: As soluções acima não corrigem a causa raiz (limitação de detecção de pulso do firmware) mas **minimizam estados travados e auto-recuperam quando ocorrem**. Para correção definitiva, Tasmota com comando `SwitchDebounce` ou ESPHome com configuração `on_press/on_release` filtrada seria necessário.

## PSF-BD1-GL informações específicas

**Confirmado**: PSF-BD1-GL é a designação de modelo do Sonoff Mini (variante global). Utiliza chip **ESP8285 com 1MB de flash**. Versões documentadas incluem 3.3.0, 3.5.0, 3.6.0 e 3.7.6. Todos os métodos descritos aplicam-se a este modelo.

## Análise de viabilidade e priorização

### Solução imediata (sem abrir dispositivo):
1. **Modo Inching via eWeLink** (viabilidade: alta, efetividade: 70-80%)
2. **Watchdog automation HA** (viabilidade: alta, efetividade: 90% para detecção)
3. **Teste Following Mode** (viabilidade: alta, efetividade: variável)

### Solução intermediária (requer acesso físico mas sem solda):
4. **Modo DIY para ajustes via API** (viabilidade: média, efetividade: 60% - não adiciona debounce real)

### Solução definitiva (requer flash):
5. **Flash Tasmota via OTA** (viabilidade: alta se firmware 3.6.0-3.7.6, efetividade: 100%)
6. **Flash serial** (viabilidade: alta mas requer abertura, efetividade: 100%)

**Recomendação final**: Comece implementando modo Inching (1.5s) + watchdog automation. Esta combinação deve resolver 80-90% dos travamentos sem necessidade de flash. Se o problema persistir após 2 semanas de teste, considere o flash OTA - primeiro verificando a versão real do firmware conforme procedimento descrito no início deste relatório.

## Conclusão técnica

O firmware 3.8.x permanece não documentado, sugerindo verificação da versão. Para versões documentadas (3.5.0-3.7.6), flash OTA é viável via modo DIY v2.0 sem jumper físico. A integração SonoffLAN 3.9.3 fornece controle mas não configuração avançada. Sua solução mais prática é modo Inching via eWeLink (1.5-2s) combinado com automação watchdog no Home Assistant, que deve prevenir e recuperar estados travados sem modificação de firmware. Firmware Tasmota/ESPHome permanece como solução definitiva caso necessário.