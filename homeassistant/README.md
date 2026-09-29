# Registo de temperatura dos AC (Toyotomi / EWPE Smart) no Home Assistant

Guarda a temperatura ambiente de 5 unidades AC a cada 15 minutos, durante pelo
menos 2 anos, num Raspberry Pi 3 com Home Assistant.

Os AC Toyotomi com a app EWPE Smart usam o protocolo Gree. A integração nativa
**Gree Climate** comunica por LAN e expõe `current_temperature` em cada entidade `climate`.

## Ficheiros

| Ficheiro | Onde vai no HA | Conteúdo |
|---|---|---|
| `configuration.yaml` | `/homeassistant/configuration.yaml` (= `/config`) | sensores template (15 min), `recorder` (800 dias) e `shell_command` para o CSV |
| `automations.yaml` | acrescentar a `/homeassistant/automations.yaml` | automação que escreve uma linha no CSV a cada 15 min |

## Unidades

| Divisão | Entidade `climate` | Sensor criado |
|---|---|---|
| Living Room | `climate.1ec942ca` | `sensor.ac_living_room_temperatura` |
| Cozinha | `climate.c62db069` | `sensor.ac_cozinha_temperatura` |
| Office | `climate.c63fd170` | `sensor.ac_office_temperatura` |
| Suite Master | `climate.c640155f` | `sensor.ac_suite_master_temperatura` |
| Kids Room | `climate.c64015c4` | `sensor.ac_kids_room_temperatura` |

Os IDs `climate.*` são específicos desta instalação. Noutra instalação, vê-os em
*Ferramentas de programador → Estados* (filtro `climate.`).

## Instalação

1. Adicionar a integração **Gree Climate** (*Definições → Dispositivos e serviços*).
   Os AC têm de estar na mesma rede que o HA. Reservar IP fixo no router.
2. Confirmar `current_temperature` nos atributos de cada `climate.*`
   (*Ferramentas de programador → Estados*).
3. Copiar o conteúdo de `configuration.yaml` para o do HA. Se já existirem
   `template:`, `recorder:` ou `shell_command:`, **juntar** aos existentes em vez de repetir a chave.
4. Acrescentar a automação de `automations.yaml` ao ficheiro do HA.
5. *Ferramentas de programador → YAML → Verificar configuração* e depois **Reiniciar** o HA.
6. Testar o CSV sem esperar 15 min: *Ferramentas de programador → Ações →*
   `shell_command.append_temperaturas_csv`. Deve aparecer `/config/temperaturas.csv`.

## Como funciona

- Um *template* com `time_pattern` corre nos minutos 0/15/30/45: atualiza as 5
  entidades (`homeassistant.update_entity`) e grava o valor em 5 sensores com
  `state_class: measurement`.
- O atributo `amostra` (timestamp) força uma linha nova no histórico mesmo quando
  a temperatura não muda, para não haver buracos na série.
- O `recorder` guarda **só** estes 5 sensores durante 800 dias. `commit_interval: 60`
  reduz escritas no cartão SD.
- A automação corre aos 30 s de cada quarto de hora e acrescenta uma linha a
  `/config/temperaturas.csv` (`data,living_room,cozinha,office,suite_master,kids_room`),
  com o campo em branco quando o sensor está indisponível.

## Obter os dados para estudo

- **CSV:** *File editor* / *Studio Code Server* / Samba (`config/temperaturas.csv`).
- **Base de dados do HA:** `home-assistant_v2.db` (SQLite), tabelas `states` e `states_meta`.
- Histórico gráfico: *Histórico* no HA (estatísticas horárias ficam para sempre).

## Limitações e cuidados

- **Resolução de 1 °C:** os AC reportam inteiros.
- **Kids Room:** no arranque `current_temperature` (23) era igual ao setpoint (23).
  Confirmar que varia; se ficar colado ao setpoint, essa unidade não expõe o sensor de sala.
- Validar as leituras contra um termómetro (alguns modelos têm offset).
- **Cartão SD:** pode falhar. Fazer **backups automáticos para fora do Pi** e copiar
  o CSV periodicamente. Usar cartão *high endurance*.
- `recorder.include` só guarda histórico destes sensores. Trocar por `exclude` para manter o resto.

## Base de dados MongoDB (opcional)

O HA não escreve em MongoDB nativamente e o Pi 3 (1 GB RAM, ARMv8.0) não é
adequado para correr o MongoDB. Se quiseres Mongo, corre-o fora do Pi
(outra máquina, NAS ou Atlas) e envia as leituras por um ponte (MQTT ou webhook).
