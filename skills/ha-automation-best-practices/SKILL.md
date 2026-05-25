---
name: ha-automation-best-practices
description: "Melhores práticas para automações, scripts, blueprints, templates e helpers no Home Assistant (2025+). Use esta skill antes de criar QUALQUER automação, script, blueprint, template, macro ou helper."
version: 1.0.0
author: Bsector
license: MIT
metadata:
  hermes:
    tags: [home-assistant, automation, scripting, blueprints, templates, helpers, smart-home]
    related_skills: [home-assistant]
---

# Home Assistant: Melhores Práticas (2025+)

Válido para Home Assistant 2025.1 até 2026.5. Não use padrões anteriores a 2025 — muitos foram deprecados ou substituídos.

---

## ⚡ REGRA DE OURO

**Sempre buscar a documentação mais atualizada.** Antes de criar qualquer automação, script, blueprint, template ou helper, verifique a documentação oficial em `https://www.home-assistant.io/docs/` e o blog de releases em `https://www.home-assistant.io/blog/`. Padrões pré-2025 frequentemente usam sintaxe deprecated.

---

## 1. AUTOMAÇÕES

### 1.1 Estrutura Canônica (2025+)

```yaml
automation:
  - id: "identificador_unico"       # OBRIGATÓRIO — use sempre
    alias: "Nome Amigável"
    description: "O que esta automação faz"
    mode: single                     # single | restart | queued | parallel
    max_exceeded: silent             # silent | warning | log level
    triggers:
      - trigger: state
        entity_id: binary_sensor.motion_kitchen
        to: "on"
        id: "motion_kitchen"         # use IDs para múltiplos triggers
    conditions:                      # PLURAL — 'condition' singular é legacy
      - condition: state
        entity_id: sun.sun
        state: "below_horizon"
    actions:                         # PLURAL — 'action' singular é legacy
      - action: light.turn_on        # 'action:', NÃO 'service:'
        target:                      # target:, NÃO entity_id: direto
          entity_id: light.kitchen
```

### 1.2 Checklist Obrigatório

- [ ] `id:` único presente em TODA automação
- [ ] `triggers:` (plural), não `trigger:` singular
- [ ] `conditions:` (plural), não `condition:` singular
- [ ] `actions:` (plural), não `action:` singular
- [ ] `action: domain.service`, não `service: domain.service`
- [ ] `target: entity_id:`, não `entity_id:` direto na action
- [ ] `data:` com templates inline, nunca `data_template:`
- [ ] NUNCA usar `device_id` — sempre `entity_id`

### 1.3 Triggers — Melhores Práticas

**Use time_pattern, não template para horários:**
```yaml
# ERRADO — avaliado 1440x/dia
triggers:
  - trigger: template
    value_template: "{{ now().minute == 1 }}"

# CERTO — avaliado 24x/dia
triggers:
  - trigger: time_pattern
    minutes: "1"
```

**Use trigger_variables para templates nos triggers:**
```yaml
trigger_variables:
  room: "living_room"
triggers:
  - trigger: mqtt
    topic: "{{ room }}/switch/ac"
```

**Múltiplos triggers com choose (evite delays):**
```yaml
triggers:
  - trigger: time
    at: "21:00:00"
    id: inicio
  - trigger: time
    at: "23:59:00"
    id: fim
actions:
  - choose:
      - conditions:
          - condition: trigger
            id: inicio
        sequence:
          - action: switch.turn_on
            target:
              entity_id: switch.difusor
      - conditions:
          - condition: trigger
            id: fim
        sequence:
          - action: switch.turn_off
            target:
              entity_id: switch.difusor
```

### 1.4 Conditions — Melhores Práticas

**Formato curto (template) para condições simples:**
```yaml
conditions:
  - "{{ is_state('binary_sensor.porta', 'on') }}"
  - "{{ is_state('binary_sensor.porta', 'on') or is_state('binary_sensor.janela', 'on') }}"
```

**Use groups para simplificar condições multi-entidade:**
```yaml
# Define o grupo UMA vez
group:
  family_members:
    name: Família
    entities:
      - device_tracker.iphone_pai
      - device_tracker.iphone_mae

# Usa o grupo em qualquer automação
conditions:
  - condition: state
    entity_id: group.family_members
    state: "home"
```

**Operadores lógicos — shorthand para OR/AND:**
```yaml
conditions:
  - or:
      - condition: state
        entity_id: binary_sensor.door
        state: "on"
      - condition: state
        entity_id: binary_sensor.window
        state: "on"
```

### 1.5 Modos de Execução

| Modo | Quando usar | Cuidado |
|------|-------------|---------|
| `single` (default) | Ações que não devem empilhar (notificações, comandos únicos) | Ignora re-triggers silenciosamente. Use `max_exceeded: silent` para suprimir warnings. |
| `restart` | Motion lights, redefinição de timer | **PERIGO:** `restart` + `delay` pode travar a automação. Prefira triggers com choose. |
| `queued` | Ações que devem executar em ordem (cena complexa) | Respeita ordem FIFO. Limite com `max:`. |
| `parallel` | Ações independentes e simultâneas | Pode gerar race conditions. Use com cautela. |

**Throttling com single + delay (anti-flood):**
```yaml
automation:
  - alias: "Luz com motion — anti-flood"
    mode: single
    max_exceeded: silent
    triggers:
      - trigger: state
        entity_id: binary_sensor.motion
        to: "on"
    actions:
      - action: light.turn_on
        target:
          entity_id: light.exterior
      - delay: 300
      - action: light.turn_off
        target:
          entity_id: light.exterior
```

---

## 2. SCRIPTS

### 2.1 Estrutura Canônica

```yaml
script:
  meu_script:
    mode: restart
    max: 10
    fields:                         # parâmetros de entrada
      brightness:
        selector:
          number:
            min: 0
            max: 255
        default: 128
    variables:
      my_var: "valor inicial"
    sequence:
      - alias: "Passo 1 — acender luz"
        action: light.turn_on
        target:
          entity_id: light.sala
```

### 2.2 Fields — Parâmetros Reutilizáveis

```yaml
script:
  flash_light:
    fields:
      light:
        selector:
          entity:
            domain: light
      count:
        default: 3
        selector:
          number:
            min: 1
            max: 10
    sequence:
      - repeat:
          count: "{{ count }}"
          sequence:
            - action: light.toggle
              target:
                entity_id: "{{ light }}"
            - delay: 1
```

### 2.3 Sequence — Elementos Essenciais

**Choose (if/elif/else):**
```yaml
- choose:
    - conditions:
        - condition: template
          value_template: "{{ now().hour < 9 }}"
      sequence:
        - action: script.morning_routine
    - conditions:
        - condition: template
          value_template: "{{ now().hour < 18 }}"
      sequence:
        - action: script.day_routine
  default:
    - action: script.night_routine
```

**Repeat — for_each com objetos:**
```yaml
- repeat:
    for_each:
      - language: English
        message: Hello World
      - language: Dutch
        message: Hallo Wereld
    sequence:
      - action: notify.phone
        data:
          title: "Message in {{ repeat.item.language }}"
          message: "{{ repeat.item.message }}!"
```

**Wait com timeout encadeado:**
```yaml
- wait_template: "{{ is_state('binary_sensor.door_1', 'on') }}"
  timeout: 10
  continue_on_timeout: false
- wait_for_trigger:
    - trigger: state
      entity_id: binary_sensor.door_2
      to: "on"
  timeout: "{{ wait.remaining }}"
```

**Parallel — ações simultâneas:**
```yaml
- parallel:
    - action: notify.person1
      data:
        message: "Aviso 1"
    - action: notify.person2
      data:
        message: "Aviso 2"
```

### 2.4 Script vs Automação vs Blueprint

| Tipo | Quando usar | Exemplo |
|------|-------------|---------|
| **Automação** | Lógica reativa trigger-driven | Acender luz com movimento |
| **Script** | Sequência procedural reutilizável | Rotina "boa noite" chamada de vários lugares |
| **Blueprint** | Template parametrizável compartilhável | "Sensor Light" genérico para qualquer cômodo |

---

## 3. BLUEPRINTS

### 3.1 Estrutura Canônica

```yaml
blueprint:
  name: Motion-activated Light
  description: >
    Acende luz ao detectar movimento.
    Markdown **suportado** na descrição.
  domain: automation
  author: Seu Nome
  homeassistant:
    min_version: "2025.1.0"
  input:
    motion_entity:
      name: Motion Sensor
      selector:
        entity:
          filter:
            device_class: motion
            domain: binary_sensor
    light_target:
      name: Light
      selector:
        target:
          entity:
            domain: light
    no_motion_wait:
      name: Wait time
      default: 120
      selector:
        number:
          min: 0
          max: 3600
          unit_of_measurement: seconds

mode: restart
max_exceeded: silent

triggers:
  - trigger: state
    entity_id: !input motion_entity
    from: "off"
    to: "on"

actions:
  - action: light.turn_on
    target: !input light_target
  - wait_for_trigger:
      - trigger: state
        entity_id: !input motion_entity
        from: "on"
        to: "off"
  - delay: !input no_motion_wait
  - action: light.turn_off
    target: !input light_target
```

### 3.2 Input Sections (2024.6+)

```yaml
blueprint:
  input:
    my_section:
      name: Minha Seção
      icon: mdi:cog
      description: "Opções específicas"
      collapsed: false
      input:
        my_input:
          name: Example input
        my_input_2:
          name: Segundo input
```

### 3.3 Selectors Essenciais

| Selector | Quando usar |
|----------|-------------|
| `entity` | Selecionar entidade com filtros (domain, device_class) |
| `target` | Target (entity/area/device/label) — preferido para actions |
| `number` | Valor numérico com min/max/unit |
| `duration` | Duração (dias, horas, min, seg) |
| `boolean` | Toggle on/off |
| `text` | Campo de texto |
| `time` | Horário |
| `template` | Editor de template |
| `action` | Editor de actions completo |
| `condition` | Editor de condições |
| `device` | Selecionar dispositivo (evitar — prefira entity) |

### 3.4 Inputs em Templates

```yaml
# !input é YAML tag, não template variable
# Para usar em template, exponha como variável:
variables:
  my_input: !input my_input

# Agora pode usar em templates:
actions:
  - action: notify.notify
    data:
      message: "{{ my_input }}"
```

---

## 4. TEMPLATES

### 4.1 Sintaxe Jinja2

| Delimitador | Uso |
|-------------|-----|
| `{{ ... }}` | Expressão / imprimir valor |
| `{% ... %}` | Statement (if, for, set) |
| `{# ... #}` | Comentário |

### 4.2 Funções Essenciais

**Estado:**
```yaml
{{ states('sensor.temperature') }}                    # valor do estado
{{ state_attr('light.sala', 'brightness') }}          # atributo
{{ is_state('light.sala', 'on') }}                    # compara estado
{{ is_state_attr('light.sala', 'brightness', 255) }}  # compara atributo
{{ has_value('sensor.temp') }}                        # tem valor (não unknown/unavailable)
```

**Trigger (apenas em automações):**
```yaml
{{ trigger.id }}              # id do trigger que disparou
{{ trigger.entity_id }}       # entidade
{{ trigger.to_state.state }}  # novo estado
{{ trigger.from_state.state }} # estado anterior
{{ this.entity_id }}          # entity_id da própria automação
```

**Data/Hora:**
```yaml
{{ now() }}                              # datetime atual com timezone
{{ now().strftime('%H:%M') }}            # formatado
{{ today_at('08:00') }}                  # hoje às 08:00
{{ as_timestamp(now()) }}                # timestamp unix
{{ as_datetime('2025-01-01') }}          # string → datetime
```

**Filtros úteis:**
```yaml
{{ value | float(default=0) }}
{{ value | int }}
{{ value | round(2) }}
{{ value | abs }}
{{ 'ON' | lower }}
{{ list | join(', ') }}
{{ list | length }}
{{ list | select('eq', 'valor') | list }}
{{ list | reject('eq', 'valor') | list }}
```

**Operadores:**
```yaml
{{ 'hello' ~ ' ' ~ 'world' }}     # concatenação
{{ value in list }}               # pertinência
{{ value is defined }}            # teste de existência
{{ value is none }}               # teste de null
```

### 4.3 Exemplos Práticos

**Bateria baixa (lista de entidades):**
```yaml
{% set low = states.sensor
  | selectattr('attributes.device_class', 'eq', 'battery')
  | selectattr('state', '<=', 20)
  | map(attribute='name') | list %}
{% if low %}
  Bateria baixa: {{ low | join(', ') }}
{% endif %}
```

**Média de temperatura:**
```yaml
{% set temps = [
  states('sensor.temp_sala') | float,
  states('sensor.temp_quarto') | float
] %}
{{ (temps | sum / temps | length) | round(1) }}
```

**Expansão de grupo com filtro temporal:**
```yaml
{{ expand('group.family_members')
   | map(attribute='last_changed')
   | select('gt', now() - timedelta(minutes=6))
   | list | count > 0 }}
```

### 4.4 Template Triggers (Sensores Atualizados Sob Demanda)

```yaml
template:
  - trigger:
      - platform: state
        entity_id: sensor.temperatura_externa
    sensor:
      - name: "Temperatura Ajustada"
        state: "{{ states('sensor.temperatura_externa') | float + 2 }}"
```

---

## 5. MUDANÇAS BREAKING / DEPRECATED (2024-2026)

### Tabela de Substituição

| ❌ Antigo (NÃO USE) | ✅ Atual (USE) |
|---------------------|----------------|
| `trigger:` (singular) | `triggers:` (plural) |
| `condition:` (singular) | `conditions:` (plural) |
| `action:` (singular) | `actions:` (plural) |
| `service: domain.action` | `action: domain.action` |
| `data_template:` | `data:` (avalia templates automaticamente) |
| `entity_id:` direto na action | `target: entity_id:` |
| `device_id` em triggers | `entity_id` com trigger nativo |
| Template legacy (`platform: template`) | Template moderno (`template:`) |

### Prazos Críticos

| Data | O que muda |
|------|-----------|
| **2025.12** | Template entities legacy marcadas como deprecated |
| **2026.6** | Template entities legacy **deixam de funcionar** |

**Migração de template legacy → moderno:**
```yaml
# ❌ LEGACY (será removido em 2026.6)
sensor:
  - platform: template
    sensors:
      meu_sensor:
        value_template: "{{ states('sensor.temp') }}"

# ✅ MODERNO
template:
  - sensor:
      - name: "Meu Sensor"
        state: "{{ states('sensor.temp') }}"
```

### Novidades 2025-2026

- **Home Assistant Labs** (2025.12): features experimentais — purpose-specific triggers/conditions
- **Infrared platform** (2026.4): nativa para IR/RF, entidades de primeira classe
- **Matter lock manager** (2026.4): gerenciamento de PIN codes
- **RF platform** (2026.5): dispositivos 433/315/868/915 MHz nativos
- **Documentação de templating reescrita** (2026.5): guia progressivo completo
- **Autocomplete nos code editors** (2026.5)

---

## 6. HELPERS

### 6.1 Helpers Essenciais e Seus Padrões

| Helper | Domain | Pattern |
|--------|--------|---------|
| **Toggle** | `input_boolean` | Flags de controle (guest_mode, debug_flag, vacation_mode) |
| **Number** | `input_number` | Thresholds ajustáveis via UI |
| **Text** | `input_text` | Valores de texto configuráveis |
| **Select** | `input_select` | Dropdown de modos (house_mode: Home/Away/Night/Guests) |
| **DateTime** | `input_datetime` | Data/hora selecionável |
| **Button** | `input_button` | Botão que dispara ação |
| **Timer** | `timer` | Sobrevive a reinicializações — melhor que `delay` |
| **Counter** | `counter` | Incremental/decremental |
| **Schedule** | `schedule` | Períodos programados |
| **Group** | `group` | Agrupa entidades para condições simplificadas |

### 6.2 Helpers Derivativos (Sensores)

| Helper | Uso |
|--------|-----|
| **Template** | Derived state reutilizável (ex: `sensor.alguem_em_casa`) |
| **Threshold** | Sensor binário baseado em threshold numérico |
| **Derivative** | Taxa de variação de sensor |
| **Integral** | Soma/acumulação ao longo do tempo |
| **Utility Meter** | Medidor de consumo com tarifas |
| **Min/Max** | Valor mínimo/máximo/médio entre sensores |
| **Combine** | Combina estados de múltiplas entidades |
| **Trend** | Sensor binário de tendência (subindo/descendo) |

### 6.3 Patterns de Uso

**Flag de debug:**
```yaml
input_boolean:
  debug_flag:
    name: Modo Debug
    icon: mdi:bug

script:
  debug_notify:
    sequence:
      - if:
          - condition: state
            entity_id: input_boolean.debug_flag
            state: "on"
        then:
          - action: notify.mobile_app
            data:
              message: "Debug: {{ debug_message }}"
```

**Derived state (evita repetir lógica):**
```yaml
template:
  - sensor:
      - name: "Alguém em Casa"
        state: >
          {{ 'on' if expand('group.family_members')
             | selectattr('state', 'eq', 'home')
             | list | count > 0 else 'off' }}
```

**Timer (sobrevive a restart):**
```yaml
automation:
  - alias: "Luz com timer"
    triggers:
      - trigger: state
        entity_id: binary_sensor.motion
        to: "on"
    actions:
      - action: light.turn_on
        target:
          entity_id: light.corredor
      - action: timer.start
        target:
          entity_id: timer.corredor
        data:
          duration: "00:05:00"

  - alias: "Desligar ao fim do timer"
    triggers:
      - trigger: event
        event_type: timer.finished
        event_data:
          entity_id: timer.corredor
    actions:
      - action: light.turn_off
        target:
          entity_id: light.corredor
```

---

## 7. ORGANIZAÇÃO DE CÓDIGO

### 7.1 Packages (Recomendado)

```
packages/
  iluminacao.yaml
  climatizacao.yaml
  seguranca.yaml
  notificacoes.yaml
```

Ative em `configuration.yaml`:
```yaml
homeassistant:
  packages: !include_dir_named packages
```

### 7.2 Automações Granulares

**ERRADO:** uma automação monolítica com 50 triggers e 200 linhas de actions.

**CERTO:** várias automações pequenas, uma por função:
- `alarm_disarm.yaml`
- `alarm_arm_home.yaml`
- `alarm_trigger.yaml`

---

## 8. CHECKLIST ANTI-PITFALL

Antes de finalizar QUALQUER automação/script/blueprint/template, verifique:

- [ ] Nunca usei `device_id` em triggers ou actions
- [ ] Usei `triggers:`, `conditions:`, `actions:` (plural)
- [ ] Usei `action:` (não `service:`) nas actions
- [ ] Usei `target: entity_id:` (não `entity_id:` direto)
- [ ] Não usei `data_template:` (use `data:` com templates inline)
- [ ] Incluí `id:` único em toda automação
- [ ] Usei `mode:` adequado ao caso de uso
- [ ] Evitei delays longos (use triggers com choose ou timer)
- [ ] Confirmei nomes de entidade em Dev Tools > States (não no nome do UI)
- [ ] Templates no formato moderno (`template:`), não legacy (`platform: template`)
- [ ] Preferi entity_id a device em todos os lugares
- [ ] Usei groups para simplificar condições multi-entidade
- [ ] Consultei a documentação mais recente, NÃO tutoriais antigos

---

## 9. REFERÊNCIAS RÁPIDAS

| O que | Onde |
|-------|------|
| Documentação oficial | https://www.home-assistant.io/docs/ |
| Blog de releases | https://www.home-assistant.io/blog/ |
| Fórum da comunidade | https://community.home-assistant.io/ |
| Automações | https://www.home-assistant.io/docs/automation/ |
| Scripts | https://www.home-assistant.io/docs/scripts/ |
| Blueprints | https://www.home-assistant.io/docs/blueprint/ |
| Templates | https://www.home-assistant.io/docs/configuration/templating/ |
| Helpers | Settings → Devices & Services → Helpers |
| Dev Tools (testar templates) | Developer Tools → Template |
| Dev Tools (ver estados) | Developer Tools → States |

---

## 10. REGRA FINAL

> **NUNCA crie automações, scripts, blueprints, templates, macros ou helpers baseados em documentação anterior a 2025.** Sempre verifique a documentação oficial mais recente e o blog de releases antes de implementar qualquer coisa. Se a fonte que você está consultando usa `service:`, `data_template:`, `entity_id:` direto na action, ou templates legacy com `platform: template` + `sensors:`, descarte-a — está desatualizada.
