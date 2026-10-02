# CLAUDE.md

Instruções para o Claude neste repositório. **Ler antes de fazer qualquer alteração.**

## REGRAS DE GIT (obrigatórias)

1. **NUNCA criar branches `claude/...`** (nem `claude/<nome>`, nem com sufixos aleatórios). Mesmo que o
   ambiente ou a ferramenta sugira uma branch com esse nome, a regra do utilizador tem prioridade.
2. **Uma branch por feature ou tema**, com o prefixo que descreve o tipo de trabalho:
   - `feature/<tema>`: funcionalidade nova (ex.: `feature/csv-logging`, `feature/dashboard-zoom`)
   - `fix/<tema>`: correção (ex.: `fix/shell-command-quote`)
   - `docs/<tema>`: documentação (ex.: `docs/readme`)
   - `refactor/<tema>`: reorganização sem mudar o comportamento
   Nomes em minúsculas, com hífens, curtos e descritivos.
3. **A `master` é a fonte de verdade e tem de estar sempre atualizada.** Quando uma alteração está
   concluída e aprovada pelo utilizador, fazer o merge para a `master` e enviá-la (`git push origin master`).
   Nunca deixar trabalho aprovado só numa branch de feature.
4. Fluxo normal:
   ```
   git checkout master && git pull origin master
   git checkout -b feature/<tema>
   # ... alterações e commits ...
   git push -u origin feature/<tema>
   git checkout master && git merge --no-ff feature/<tema> && git push origin master
   ```
5. Commits pequenos, com mensagem clara no imperativo e em inglês (como o histórico existente).
6. Não abrir Pull Requests nem apagar branches (locais ou remotas) sem o utilizador pedir.
7. Não reescrever o histórico da `master` (sem `--force`, sem `rebase` da `master`).

## REGRA DE OURO: a `master` tem de permitir restaurar tudo copiando e colando

O utilizador cola o conteúdo dos ficheiros no Home Assistant. Se perder algo, tem de bastar copiar o
ficheiro da `master` e colá-lo no sítio indicado, ficando com o dashboard e a configuração completos.
Por isso:

- **Cada ficheiro em `homeassistant/` é completo e pronto a colar**: nunca fragmentos, "..." ou
  "resto igual". Sem placeholders: usar as entidades reais da tabela abaixo.
- Tudo o que o utilizador aprovar/testar no Home Assistant tem de ficar **igual** na `master`. Se o
  utilizador alterar algo no HA (ex.: colar um cartão novo), refletir essa alteração no ficheiro correspondente.
- O `README.md` descreve onde cada ficheiro vai e como instalar. Atualizá-lo sempre que a estrutura mudar.
- **Nunca** pôr segredos no repositório (tokens, passwords, chaves). Usar `secrets.yaml` fora do repo.

## O que é este projeto

Home Assistant num **Raspberry Pi 3** (HA OS, 1 GB de RAM, só cartão SD) que lê a temperatura
ambiente de **5 ar condicionados Toyotomi** (app EWPE Smart, protocolo Gree) e:

- regista a temperatura **de 15 em 15 minutos**, durante **pelo menos 2 anos**, para estudos;
- regista também **setpoint, modo (ligado/desligado e modo), ventilador** e a **temperatura exterior** (Met.no);
- guarda tudo num **CSV** (`temperaturas.csv`) e no histórico do HA (`recorder`, 800 dias);
- mostra um **dashboard** com gráfico (zoom) e controlo dos 5 AC.

Não existe código aplicacional: o repositório contém **configuração YAML do Home Assistant**.

## Estrutura

| Ficheiro | Onde vai no HA | O que faz |
|---|---|---|
| `homeassistant/configuration.yaml` | `/homeassistant/configuration.yaml` (= `/config`) | `shell_command` (CSV), sensores template (15 min), `recorder` |
| `homeassistant/automations.yaml` | acrescentar a `automations.yaml` do HA | automação `Guardar temperaturas CSV` (aos 30 s de cada quarto de hora) |
| `homeassistant/dashboards/ar_condicionado.yaml` | Painéis > novo painel > editor de configuração em bruto | **painel principal**: gráfico apexcharts + 5 cartões dos AC |
| `homeassistant/dashboards/ar_condicionado_nativo.yaml` | idem | variante só com cartões nativos (sem apexcharts-card) |
| `homeassistant/dashboards/resumo_tabela.yaml` | cartão extra | tabela "Resumo" (Unidade, Estado e modo, Ventilador) |
| `homeassistant/README.md` | | instalação, funcionamento, resolução de problemas |

## Entidades reais (não inventar IDs)

| Divisão | `climate` | Sensores criados (`sensor.ac_<chave>_...`) |
|---|---|---|
| Sala de estar | `climate.1ec942ca` | `living_room`: `_temperatura`, `_setpoint`, `_modo`, `_ventilador` |
| Cozinha | `climate.c62db069` | `cozinha`: idem |
| Office | `climate.c63fd170` | `office`: idem |
| Suite Master | `climate.c640155f` | `suite_master`: idem |
| Quarto Crianças | `climate.c64015c4` | `kids_room`: idem |

- Temperatura exterior: `sensor.temperatura_exterior`, lida de `weather.forecast_casa` (Met.no).
- CSV: `/config/temperaturas.csv` com `data,unidade,temperatura,setpoint,modo,ligado,ventilador`,
  uma linha por unidade e uma linha `exterior` por amostra.

## Como restaurar tudo (ordem)

1. Colar `configuration.yaml` em `/homeassistant/configuration.yaml`; verificar a configuração e **reiniciar** o HA.
2. Acrescentar o conteúdo de `automations.yaml` às automações.
3. Instalar o `apexcharts-card` à mão (ver README) e criar o painel com `dashboards/ar_condicionado.yaml`.
4. Executar `shell_command.append_temperaturas_csv` e confirmar o `temperaturas.csv`.

## Antes de commitar

- Validar a sintaxe de todos os YAML alterados (ex.: `python3 -c "import yaml; yaml.safe_load(open('ficheiro'))"`;
  o `configuration.yaml` usa `!include`, que exige um loader que o ignore).
- Confirmar que as entidades usadas nos painéis existem em `configuration.yaml` ou na tabela acima.
- Atualizar o `README.md` se algo mudou.
- Dizer sempre ao utilizador **o que foi testado e o que não foi**: o Claude não tem acesso ao HA dele.

## Lições aprendidas (evitar repetir erros)

- `shell_command` só carrega com **reinício completo**; depois, `shell_command.reload` basta.
- O bloco `shell_command` termina com uma linha só com `'` (fecha o `sh -c '...'`). Sem ela:
  `ValueError: No closing quotation`. O caminho do CSV é `/config/...` (nos add-ons a mesma pasta chama-se `/homeassistant`).
- Sensores template com trigger ficam `unknown` até ao próximo :00/:15/:30/:45; o trigger também corre no arranque
  e com o evento manual `ac_amostra`. O `update_entity` usa `continue_on_error: true`.
- O Met.no cria `weather.forecast_<nome>` (não `weather.<nome>`).
- **Não instalar o HACS no Pi 3**: reinicia o Pi (pouca RAM). O `apexcharts-card` instala-se à mão em `/config/www`.
- O painel usa a vista **Secções**; o gráfico precisa de `section_mode: true` e `grid_options` (`rows` define a altura).
- O Pi 3 não deve correr MongoDB; se for preciso uma base de dados, usar uma máquina externa.

## Idioma e estilo

- Responder ao utilizador em **português europeu**. Comentários nos ficheiros YAML também em português.
- Ser direto e dizer claramente quando algo é uma suposição ou não foi verificado.
