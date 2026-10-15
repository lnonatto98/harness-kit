---
name: azure-devops-efficiency-sync
description: Preenche a aba Efficiency de uma Task do Azure DevOps (TEM-DevOps) a partir do Efficiency Measurement Report gerado por /harness-kit:ai-vs-human-efficiency-auditor e sugere o Log Effort, que deve ser lançado manualmente. Valida o MCP e se a Task está atribuída ao usuário autenticado antes de gravar. Use quando o usuário pedir para atualizar Efficiency, horas humanas/AI ou percentual de código gerado por AI em uma task.
---

# Azure DevOps Efficiency Sync

Lê o relatório do `ai-vs-human-efficiency-auditor`, preenche a aba Efficiency de uma Task do Azure DevOps, pergunta o que o relatório não responde e grava somente após confirmação. Não altera código do repositório.

## Entrada

1. **Relatório do auditor** no contexto atual. Se não houver, execute primeiro `/harness-kit:ai-vs-human-efficiency-auditor` (ele pede a senioridade e roda `git --no-pager diff HEAD`).
2. **ID da Task** (ex.: `5121`). Se não informado, pergunte.
3. **Organização/projeto:** `TelecomExpenseManagement-TEM` / `TEM-DevOps`.

## Validações obrigatórias (antes de qualquer gravação)

1. **MCP e atribuição em uma única consulta.** Carregue as ferramentas com `tool_search_tool` e execute `azure-devops-wit_query` (`action=wiql`, `project=TEM-DevOps`):
   ```
   SELECT [System.Id] FROM WorkItems WHERE [System.Id] = {id} AND [System.AssignedTo] = @Me
   ```
   - Se a chamada falhar ou não responder, pare e responda:
     > Erro: o MCP `azure-devops` não está configurado ou não tem permissão. Verifique a configuração do MCP e a autenticação, depois tente de novo. Nenhuma alteração foi feita.
   - Se o ID não aparecer no resultado, pare e responda:
     > Erro: a Task {id} não está atribuída a você. A gravação foi bloqueada. Nenhuma alteração foi feita.

Não use `list_projects` para validar acesso: é uma chamada extra. Só prossiga se o ID aparecer no resultado.

## Mapeamento (relatório → campo)

### Preenchidos automaticamente (a partir do relatório)

| Campo Azure DevOps (reference name) | Origem no relatório | Regra |
|---|---|---|
| `Custom.Howmanyhourshumanwouldtake` | Estimated Manual Time | horas decimais (ex.: `0.2`) |
| `Custom.HowmanyhoursAIwouldtake` | AI-Assisted Time (Estimated) | horas decimais (ex.: `0.1`) |
| `Custom.PercentageofcodegeneratedbyAI` | AI Contribution → Code Generation | inteiro 0–100 |
| `Custom.DidyouuseAIFirstConcept` | Dados de eficiência de uso com AI no relatório (AI-Assisted Time e Code Generation) | `true` se o relatório tiver dados de eficiência com AI; senão `false` |
| `Custom.Howcomplexwasthetask` | Linhas adicionadas no diff (o relatório não traz essa contagem; rode `git --no-pager diff HEAD --numstat` e ignore a blacklist do auditor: locks, migrations, binários, dados estáticos) | menos de 20 linhas → `Low – Easy, small change` (valor validado na 5121). Outras faixas: pergunte o valor exato, pois o picklist não foi validado |
| `Custom.Deliveredontime` | Data de conclusão vs. término da sprint | `Yes` ou `No` (veja abaixo) |

**Delivered on time** (picklist validado pelo usuário: `Yes` / `No`):
1. Use `System.IterationPath`, `Microsoft.VSTS.Common.ClosedDate` e `System.ChangedDate` da leitura única da Task (passo 2 do Procedimento). Não leia a Task de novo para isso.
2. Descubra o `finishDate` da sprint com o mínimo de consultas: `azure-devops-core_list_project_teams` (`mine=true`) uma vez e, em seguida, `azure-devops-work` (`action=list_team_iterations`, `timeframe=current`) nos times retornados. Use a sprint cujo `path` seja igual ao `System.IterationPath`. Não use o `AreaPath` para escolher o time: o nome do time não é igual ao da área (ex.: `TEM-Dev Team` retornou erro). Se a ferramenta `azure-devops-work` oferecer listagem de iterações do projeto, prefira essa chamada única (confira o schema antes).
3. Data de conclusão = `ClosedDate`, se existir. Senão, use a data de hoje (data local). Se a conclusão for igual ou anterior ao `finishDate`, grave `Yes`. Se for posterior, grave `No` e aplique a regra de motivo abaixo.
4. Se a Task ainda não estiver concluída (sem `ClosedDate`), grave `Yes` somente se hoje estiver dentro do prazo da sprint, e avise que o valor é provisório.
5. Se a sprint não for encontrada, pergunte a data de término com `ask_user`.

**Motivo de atraso (`Custom.ReasonforLateDelivery`, HTML, obrigatório quando Delivered on time = `No`):**
- Gere o motivo automaticamente, nesta ordem:
  1. Se `Custom.Didyouexperienceanyblockers` = `Yes` e `Custom.Ifyespleasedescribeblockers` estiver preenchido, use essa descrição.
  2. Senão, monte um texto factual com dados já lidos: `ClosedDate` (ou data de hoje), `finishDate` da sprint e o nome da sprint. Exemplo: "Task concluída em {data}, após o término da sprint {nome} em {finishDate}." Não use `list_revisions`: é uma consulta pesada e não é necessária.
  3. Sem dados suficientes, use `ask_user` e pergunte o motivo.
- Mostre o motivo na tabela antes/depois. O usuário pode editá-lo antes de confirmar.

### Perguntas ao usuário (sugestão = valor atual da 5121)

Agrupe em um único `ask_user`:

| Campo | Sugestão padrão | Condicional |
|---|---|---|
| `Custom.Didyoutestthesolutionlocally` | `Yes` | |
| `Custom.DidyoudeployandtestthesolutionintheDEVenvironment` | `Yes` | |
| `Custom.Didyouexperienceanyblockers` | `No` | Se `Yes`, pergunte `Custom.Ifyespleasedescribeblockers` (HTML) |
| `Custom.Didyouneedtoaskforclarification` | `No` | |
| `Custom.Howclearweretherequirementsforthistask` | `Very clear` | |
| `Custom.Howconfidentareyouinthequalityofthisdelivery` | `Very confident` | Se baixa/neutra, pergunte `Custom.ClarificationofConfidentLevel` (HTML) |
| `Custom.TestMethodology` | (em branco) | Picklist validado: `Automated`, `Exploratory`, `Manual`, `Regression`, `Smoke`. Ofereça essas opções no `ask_user` com "em branco" como padrão. `Not applicable` é rejeitado pelo servidor |

### Não preencher
- `Custom.AutomationOpportunity`: deixe em branco (decisão do usuário).
- `Custom.TestMethodology`: deixe em branco por padrão (decisão do usuário). Preencha só se o usuário escolher uma opção válida do picklist.
- `Microsoft.VSTS.Scheduling.RemainingWork`, estado, responsável e demais campos fora da aba Efficiency.

### Log Effort (sugestão, não gravada)
A skill **não grava** `Microsoft.VSTS.Scheduling.CompletedWork`. No Azure DevOps, esse campo é a soma dos logs do plugin Kairos `burning-efforts` (`effortLogs`). Gravar o CW direto gera divergência com os logs.
- Sugestão: `AI-Assisted Time (Estimated)` do relatório, salvo se o usuário informar outras horas.
- Ao final, informe o valor sugerido e diga que ele deve ser lançado manualmente no Kairos.

## Regras

- Campos automáticos e perguntados: **substitua** o valor atual. Não some.
- `Microsoft.VSTS.Scheduling.CompletedWork` nunca é gravado pela skill. Só sugira o valor no final.
- Não altere `RemainingWork`, estado, responsável nem campos fora da aba Efficiency.
- Não grave valores de picklist não validados (ex.: complexidade fora de `Low – Easy, small change`). Pergunte ao usuário.
- `Custom.Deliveredontime` usa `Yes` ou `No` (picklist validado pelo usuário). Se o servidor rejeitar, informe o valor e pare.
- `Custom.ReasonforLateDelivery` só é preenchido quando Delivered on time = `No`. Quando for `Yes`, não altere esse campo.
- Cada dado do Azure DevOps é consultado uma vez por execução: validação (WIQL), leitura da Task, time e sprint. Não repita essas chamadas para confirmar.
- `Custom.AutomationOpportunity` pode vir como `false` na leitura. Não o grave.
- Horas sempre em decimal (`0.2h`, nunca minutos).
- Se o relatório não tiver os números, pare e peça o relatório. Não invente valores.

## Procedimento

1. Execute a consulta WIQL de validação (MCP e atribuição). Se falhar, pare e devolva o erro.
2. Leia a Task uma única vez com `azure-devops-wit_work_item` (`action=get`, `id`, `project=TEM-DevOps`, `expand=Relations`), pedindo `rev`, `System.IterationPath`, `System.ChangedDate`, `Microsoft.VSTS.Common.ClosedDate`, `System.Title`, `System.State`, `Custom.AutomationOpportunity` e todos os campos da aba Efficiency listados acima. Use essa leitura para validar, calcular e comparar: não reabra a Task.
   - Se `Relations` não tiver commits vinculados (`ArtifactLink`), o diff local pode não ser o trabalho da Task. Antes de calcular horas e complexidade, confirme com o usuário qual trabalho o relatório descreve.
3. Calcule os campos automáticos, incluindo `Custom.Deliveredontime` pela data da sprint e, se for No, o motivo de atraso.
4. Use `ask_user` para os campos perguntados, com as sugestões da 5121.
5. Monte a tabela antes/depois de todos os campos a gravar, incluindo o motivo de atraso gerado (se No). Não inclua `Custom.AutomationOpportunity`.
6. Peça confirmação com `ask_user` antes de gravar.
7. Grave com `azure-devops-wit_work_item_write` (`action=update`), com teste de concorrência em `/rev`:

   ```json
   [
     {"op": "Test", "path": "/rev", "value": <rev atual>},
     {"op": "Replace", "path": "/fields/Custom.Howmanyhourshumanwouldtake", "value": <manual>},
     {"op": "Replace", "path": "/fields/Custom.HowmanyhoursAIwouldtake", "value": <AI>},
     {"op": "Replace", "path": "/fields/Custom.PercentageofcodegeneratedbyAI", "value": <percentual>},
     {"op": "Replace", "path": "/fields/Custom.DidyouuseAIFirstConcept", "value": <true se houver dados de AI no relatório; senão false>}
   ]
   ```

   O JSON acima é parcial: inclua também `Custom.Howcomplexwasthetask`, `Custom.Deliveredontime`, `Custom.ReasonforLateDelivery` (somente se No) e as respostas perguntadas. Booleanos são `true`/`false` sem aspas. Não inclua `Microsoft.VSTS.Scheduling.CompletedWork`. Confira os nomes de parâmetros no schema da ferramenta antes de chamar.
8. Verifique pelo próprio retorno do `update`, que já traz `rev` e os campos gravados. Não faça `get` extra. Se houver 409 (conflito de revisão), releia a Task, mostre a diferença e peça confirmação de novo.
9. Se o Azure DevOps rejeitar um valor (ex.: picklist), informe o campo e o valor rejeitado. A gravação é atômica: nada é alterado.
10. Reporte em lista curta: campo, antes → depois e `rev` final.
11. Por último, informe o Log Effort sugerido em horas decimais e avise que ele deve ser lançado manualmente no Kairos. Não grave `CompletedWork`.
