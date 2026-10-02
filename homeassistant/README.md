# Registo de temperatura dos AC (Toyotomi / EWPE Smart) no Home Assistant

Guarda a temperatura ambiente de 5 unidades AC a cada 15 minutos, durante pelo
menos 2 anos, num Raspberry Pi 3 com Home Assistant.

Os AC Toyotomi com a app EWPE Smart usam o protocolo Gree. A integração nativa
**Gree Climate** comunica por LAN e expõe `current_temperature` em cada entidade `climate`.

## Ficheiros

| Ficheiro | Onde vai no HA | Conteúdo |
|---|---|---|
| `configuration.yaml` | `/homeassistant/configuration.yaml` (= `/config`) | sensores template de temperatura, setpoint, modo, ventilador e temperatura exterior (15 min), tomada do office (5 min), integração/utility meters de energia, `recorder` (800 dias) e `shell_command` para os 2 CSV |
| `dashboards/ar_condicionado.yaml` | novo painel (editor de configuração em bruto) | painel principal: gráfico apexcharts (largura total) + 5 cartões de controlo dos AC |
| `dashboards/ar_condicionado_nativo.yaml` | idem (alternativa) | variante só com cartões nativos (sem apexcharts-card), com resumo e temperaturas atuais |
| `dashboards/resumo_tabela.yaml` | cartão extra (opcional) | cartão Markdown "Resumo" em tabela única (Unidade, Estado e modo, Ventilador) |
| `dashboards/office.yaml` | novo painel (editor de configuração em bruto) | painel do office: consumos e controlo da tomada (Office UPS) + AC do office |
| `automations.yaml` | acrescentar a `/homeassistant/automations.yaml` | 2 automações: CSV das temperaturas (15 min) e CSV dos consumos do office (de hora a hora) |

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
   `shell_command.append_temperaturas_csv`. Deve aparecer `/config/temperaturas.csv` com o cabeçalho e 6 linhas (5 unidades + exterior).

## Dashboard

`dashboards/ar_condicionado.yaml`: *Definições → Painéis → Adicionar painel*, abrir, ✏️ *Editar* →
⋮ → *Editor de configuração em bruto*, selecionar tudo (Ctrl+A) e colar o ficheiro.

- Vista **Secções**: o gráfico ocupa a largura total no topo (`section_mode: true`, `rows: 7`) e por
  baixo ficam os 5 cartões `tile` com setpoint, modo e ventilador (requer HA 2024.9+).
- O eixo Y do gráfico é automático, alinhado a múltiplos de 5 (`align_to: 5`, `stepSize: 5`). Para
  mudar a altura, alterar `rows` (cada unidade ≈ 1,7 cm).
- **Zoom:** a roda do rato aproxima/afasta no tempo e o eixo Y reajusta-se (`autoScaleYaxis`); arrastar
  desloca o gráfico; a barra de ferramentas (lupas, mão, *reset*) fica à direita e a legenda à esquerda.
  A roda do rato captura o scroll da página sobre o gráfico (`allowMouseWheelZoom: false` desliga).
  Com dados novos o gráfico pode voltar à vista completa (`update_interval: 15min` reduz isso).
- Os setpoints aparecem a tracejado, na cor da unidade e fora da legenda.
- Sem o `apexcharts-card` o primeiro cartão mostra "Erro de configuração". Alternativa sem cartões
  custom: `ar_condicionado_nativo.yaml`.
- Depende dos sensores criados em `configuration.yaml`. Os sensores template com trigger
  ficam "Desconhecido" até à **próxima passagem por :00/:15/:30/:45** depois de criados
  ou de um reinício (ou até se disparar o evento `ac_amostra`).
- A entidade do Met.no chama-se normalmente `weather.forecast_<nome>` (ex.: `weather.forecast_casa`).
  Se aparecer "Entidade não encontrada", confirmar o ID em *Estados* (filtro `weather.`).
- `resumo_tabela.yaml` é um cartão extra (não incluído no painel principal).

### Instalar o apexcharts-card (sem HACS)

Num Raspberry Pi 3 a instalação do HACS pode reiniciar o Pi (pouca RAM). O cartão instala-se à mão:

1. No add-on *Terminal & SSH*:
   ```
   mkdir -p /config/www
   wget -O /config/www/apexcharts-card.js https://github.com/RomRider/apexcharts-card/releases/latest/download/apexcharts-card.js
   ```
2. Reiniciar o HA (a pasta `www` é nova).
3. Ativar o **Modo avançado** (perfil do utilizador) e em *Definições → Painéis → ⋮ → Recursos*
   adicionar `/local/apexcharts-card.js` como **Módulo JavaScript**.
4. Recarregar o browser com Ctrl+Shift+R.

## Office: tomada inteligente (Office UPS) + AC

Painel `dashboards/office.yaml`: potência em tempo real, tensão, corrente, energia de hoje/do mês/total, controlo da
tomada (ligar/desligar, bloqueio para crianças, arranque, luz) e o AC do office com a temperatura.

**Entidades da tomada (reais):** `switch.office_ups_tomada_1`, `switch.office_ups_bloqueio_para_criancas`,
`select.office_ups_comportamento_de_arranque`, `select.office_ups_modo_de_luz_indicadora`,
`sensor.office_ups_potencia` (W), `sensor.office_ups_tensao` (V), `sensor.office_ups_corrente` (A),
`sensor.office_ups_energia_total` (kWh, contador com resolução de 0,01).

**Criadas em `configuration.yaml`:**

| Entidade | O que é |
|---|---|
| `sensor.office_tomada_potencia`, `_tensao`, `_corrente` | amostras de **5 em 5 min** dos sensores da tomada (ficam no histórico; o sensor original atualiza a cada poucos segundos e encheria o cartão SD) |
| `sensor.office_energia_calculada` | energia (kWh, 3 casas) calculada por integração da potência; o contador da tomada é demasiado grosseiro para consumos por hora |
| `sensor.office_energia_hora`, `_dia`, `_mes` | `utility_meter`: consumo da hora/dia/mês atual (reiniciam no fim do período; o atributo `last_period` guarda o período anterior) |

**CSV horário `/config/office_consumos.csv`** (automação aos XX:00:30, uma linha por hora **fechada**):

```
hora_inicio,energia_hora_kwh,potencia_w,tensao_v,corrente_a,energia_total_kwh,tomada,exterior_c
2026-10-02T14:00+0100,0.039,39.3,240.8,0.163,0.02,on,13.7
```

- `hora_inicio` é o início da hora a que o consumo se refere (14:00 = consumo das 14:00 às 15:00).
- `energia_hora_kwh` vem do `last_period` do utility meter horário. Os restantes valores são instantâneos, no fim da hora. O CSV não leva dados dos AC.
- A energia calculada só conta enquanto o HA está a correr; o `energia_total_kwh` (contador da tomada) é o valor exato acumulado.
- Se a hora ainda não tem dados (primeira linha depois de instalar), `energia_hora_kwh` pode vir a 0.

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
- **"No closing quotation" (`ValueError`) ao executar o comando:** falta a aspa simples `'` que fecha o
  `sh -c '...'`, a última linha do bloco `shell_command` (4 espaços e `'`). É o erro mais comum ao copiar o bloco.
- **Como saber se o serviço existe:** *Ferramentas de programador → Ações*, escrever `shell_command`.
  Deve aparecer `shell_command.append_temperaturas_csv`.
- **"Não existe a pasta /config":** nos add-ons (File editor, Terminal & SSH) a pasta de configuração
  chama-se `/homeassistant`; dentro do Home Assistant Core chama-se `/config`. É a mesma pasta, por isso
  o CSV criado em `/config/temperaturas.csv` aparece no File editor como `temperaturas.csv`, ao lado do
  `configuration.yaml`.
- **Teste mínimo** (isola o problema): acrescentar `teste_shell: 'echo ok > /config/teste.txt'` em
  `shell_command:`, recarregar (`shell_command.reload`), chamar `shell_command.teste_shell` e ver se aparece `teste.txt`.
- O caminho do ficheiro tem de ser `f=/config/temperaturas.csv` (e não `/temperaturas.csv`, que fica fora da pasta de configuração).
- **`office_consumos.csv` sem `energia_hora_kwh`:** o `utility_meter` horário só tem `last_period` depois da primeira passagem por uma hora certa. Esperar até à hora seguinte.
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
