# Registo de temperatura dos AC (Toyotomi / EWPE Smart) no Home Assistant

Guarda a temperatura ambiente de 5 unidades AC a cada 15 minutos, durante pelo
menos 2 anos, num Raspberry Pi 3 com Home Assistant.

Os AC Toyotomi com a app EWPE Smart usam o protocolo Gree. A integração nativa
**Gree Climate** comunica por LAN e expõe `current_temperature` em cada entidade `climate`.

## Ficheiros

| Ficheiro | Onde vai no HA | Conteúdo |
|---|---|---|
| `configuration.yaml` | `/homeassistant/configuration.yaml` (= `/config`) | sensores template de temperatura, setpoint, modo, ventilador e temperatura exterior (15 min), `recorder` (800 dias) e `shell_command` para o CSV |
| `dashboards/ar_condicionado.yaml` | novo painel (raw configuration editor) | gráfico de temperaturas, controlo por unidade (modo, ventilador, setpoint), resumo e temperaturas atuais (só cartões nativos) |
| `dashboards/ar_condicionado_apexcharts.yaml` | idem (alternativa) | igual, mas com gráfico apexcharts que mostra também os setpoints a tracejado (requer HACS) |
| `automations.yaml` | acrescentar a `/homeassistant/automations.yaml` | automação que escreve 5 linhas (uma por unidade) no CSV a cada 15 min |

## Unidades

| Divisão | Entidade `climate` | Sensores criados |
|---|---|---|
| Living Room | `climate.1ec942ca` | `sensor.ac_living_room_temperatura`, `_setpoint`, `_modo`, `_ventilador` |
| Cozinha | `climate.c62db069` | `sensor.ac_cozinha_temperatura`, `_setpoint`, `_modo`, `_ventilador` |
| Office | `climate.c63fd170` | `sensor.ac_office_temperatura`, `_setpoint`, `_modo`, `_ventilador` |
| Suite Master | `climate.c640155f` | `sensor.ac_suite_master_temperatura`, `_setpoint`, `_modo`, `_ventilador` |
| Kids Room | `climate.c64015c4` | `sensor.ac_kids_room_temperatura`, `_setpoint`, `_modo`, `_ventilador` |

Temperatura exterior: `sensor.temperatura_exterior`, lida do atributo `temperature`
da entidade `weather.forecast_casa` (integração Met.no).

Os IDs `climate.*` são específicos desta instalação. Noutra instalação, vê-os em
*Ferramentas de programador → Estados* (filtro `climate.`).

## Instalação

0. A temperatura exterior usa a integração **Met.no** (a que o HA cria por defeito), entidade
   `weather.forecast_casa`. Confirmar o ID em *Ferramentas de programador → Estados* (filtro `weather.`)
   e, se for outro, substituir `weather.forecast_casa` em `configuration.yaml`.
1. Adicionar a integração **Gree Climate** (*Definições → Dispositivos e serviços*).
   Os AC têm de estar na mesma rede que o HA. Reservar IP fixo no router.
2. Confirmar `current_temperature` nos atributos de cada `climate.*`
   (*Ferramentas de programador → Estados*).
3. Copiar o conteúdo de `configuration.yaml` para o do HA. Se já existirem
   `template:`, `recorder:` ou `shell_command:`, **juntar** aos existentes em vez de repetir a chave.
4. Acrescentar a automação de `automations.yaml` ao ficheiro do HA.
5. *Ferramentas de programador → YAML → Verificar configuração* e depois **Reiniciar** o HA.
6. Testar o CSV sem esperar 15 min: *Ferramentas de programador → Ações →*
   `shell_command.append_temperaturas_csv`. Deve aparecer `/config/temperaturas.csv` com 5 linhas.

## Dashboard

`dashboards/ar_condicionado.yaml`: *Definições → Painéis → Adicionar painel*, abrir, ✏️ *Editar* →
⋮ → *Editor de configuração em bruto* e colar o ficheiro.

- `ar_condicionado.yaml` usa só cartões nativos (`history-graph`). A variante
  `ar_condicionado_apexcharts.yaml` precisa do `apexcharts-card` (HACS → Frontend);
  sem ele o primeiro cartão mostra "Erro de configuração".
- Os cartões por unidade usam `tile` com controlo de setpoint, modo e ventilador
  (requer um HA recente, 2024.9+).
- Depende dos sensores criados em `configuration.yaml`. Os sensores template com trigger
  ficam "Desconhecido" até à **próxima passagem por :00/:15/:30/:45** depois de criados
  ou de um reinício; os que já existiam recuperam o valor anterior.
- A entidade do Met.no chama-se normalmente `weather.forecast_<nome>` (ex.: `weather.forecast_casa`).
  Se aparecer "Entidade não encontrada", confirmar o ID em *Estados* (filtro `weather.`).

## Como funciona

- Um *template* com `time_pattern` corre nos minutos 0/15/30/45: atualiza as 5
  entidades (`homeassistant.update_entity`) e grava, por unidade, 4 sensores:
  **temperatura** ambiente, **setpoint** (temperatura pedida), **modo** e **ventilador**.
- O **modo** é o estado da entidade `climate`: `off` (desligado), `cool`, `heat`,
  `auto`, `dry` ou `fan_only`. Não há sensor separado de ligado/desligado:
  ligado = modo diferente de `off`.
- O **ventilador** é o atributo `fan_mode`: `auto`, `low`, `medium low`, `medium`,
  `medium high`, `high`.
- O atributo `amostra` (timestamp) força uma linha nova no histórico mesmo quando
  o valor não muda, para não haver buracos na série.
- O `recorder` guarda **só** estes 20 sensores durante 800 dias. `commit_interval: 60`
  reduz escritas no cartão SD.
- A automação corre aos 30 s de cada quarto de hora e acrescenta **uma linha por
  unidade** a `/config/temperaturas.csv`:

```
data,unidade,temperatura,setpoint,modo,ligado,ventilador
2026-09-29T16:45+0100,living_room,24,22,cool,1,medium low
2026-09-29T16:45+0100,cozinha,27,25,off,0,auto
```

  A cada amostra há ainda uma linha `exterior`, só com a coluna `temperatura`:
  `2026-09-29T16:45+0100,exterior,18.4,,,,`

  `ligado` é 1/0. Campos indisponíveis ficam em branco.
  Com a unidade em `off`, o setpoint e o ventilador são os últimos definidos, não valores em uso.

## Obter os dados para estudo

- **CSV:** *File editor* / *Studio Code Server* / Samba (`config/temperaturas.csv`).
- **Base de dados do HA:** `home-assistant_v2.db` (SQLite), tabelas `states` e `states_meta`.
- Histórico gráfico: *Histórico* no HA (estatísticas horárias ficam para sempre).

## Limitações e cuidados

- **Resolução de 1 °C:** os AC reportam inteiros.
- **Temperatura exterior:** o Met.no dá a temperatura prevista/modelada para as coordenadas
  de casa, não a medição de um termómetro. Serve para estudos de tendência, mas pode divergir
  alguns graus do valor real (sol direto, microclima). Atualiza cerca de 1 vez por hora,
  por isso vários quartos de hora seguidos repetem o mesmo valor.
- Validar as leituras contra um termómetro (alguns modelos têm offset).
- **Cartão SD:** pode falhar. Fazer **backups automáticos para fora do Pi** e copiar
  o CSV periodicamente. Usar cartão *high endurance*.
- `recorder.include` só guarda histórico destes sensores. Trocar por `exclude` para manter o resto.

## Resolução de problemas: CSV / `shell_command`

- **"serviço desconhecido: shell_command.append_temperaturas_csv"**: o bloco `shell_command:` não
  foi carregado. Confirmar que está no `configuration.yaml` (uma só vez, sem indentação), que a
  verificação de configuração passa e que o HA foi **reiniciado** (recarregar YAML não chega).
- **Como saber se o serviço existe:** *Ferramentas de programador → Ações*, escrever `shell_command`.
  Deve aparecer `shell_command.append_temperaturas_csv`.
- **"Não existe a pasta /config":** nos add-ons (File editor, Terminal & SSH) a pasta de configuração
  chama-se `/homeassistant`; dentro do Home Assistant Core chama-se `/config`. É a mesma pasta, por isso
  o CSV criado em `/config/temperaturas.csv` aparece no File editor como `temperaturas.csv`, ao lado do
  `configuration.yaml`.
- **Teste mínimo** (isola o problema): acrescentar `teste_shell: 'echo ok > /config/teste.txt'` em
  `shell_command:`, reiniciar, chamar `shell_command.teste_shell` e ver se aparece `teste.txt`.
- **Sensores template "unknown" / sem o atributo `amostra`:** o template com trigger nunca correu.
  O trigger inclui o arranque do HA e o evento manual `ac_amostra` (*Ferramentas de programador →
  Eventos → Disparar evento*). Depois de alterar `template:`, usar a ação `template.reload`
  (*Template: Recarregar*) e disparar o evento. Se continuar `unknown`, ver o registo
  (filtrar por `template`).
- Todas as ocorrências de `weather.casa` devem ser `weather.forecast_casa` (3 no `configuration.yaml`).
- Erros de execução aparecem em *Definições → Sistema → Registos* (filtrar por `shell_command`).

## Base de dados MongoDB (opcional)

O HA não escreve em MongoDB nativamente e o Pi 3 (1 GB RAM, ARMv8.0) não é
adequado para correr o MongoDB. Se quiseres Mongo, corre-o fora do Pi
(outra máquina, NAS ou Atlas) e envia as leituras por um ponte (MQTT ou webhook).
