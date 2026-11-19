# svc-py-aviacao-financeiro-leitura-extratos

Serviço que lê os arquivos de retorno bancário de PIX depositados pelo cliente
numa pasta do Google Drive e devolve os pagamentos em JSON, por HTTP, para o
Protheus dar baixa no contas a pagar.

O serviço não dá baixa, não escreve no Protheus e não decide nada de negócio.
Ele lê um arquivo e traduz o layout. Quem chama é o Protheus.

## Estado

Especificação escrita, implementação não iniciada.

| Documento | O que tem |
|---|---|
| [`specs/001-leitura-retorno-pix/design.md`](specs/001-leitura-retorno-pix/design.md) | Por que existe, objetivos e anti-objetivos, arquitetura, layout do CNAB 240, contrato da API, segurança |
| [`specs/001-leitura-retorno-pix/tasks.md`](specs/001-leitura-retorno-pix/tasks.md) | Plano de implementação em 12 tarefas, cada uma com teste e commit |

Comece pelo `design.md`. O `tasks.md` é para executar na ordem.

## Em uma frase

```
banco deposita o retorno no Drive  →  Protheus chama GET /pix/retornos?data=...
                                   →  este serviço lê, valida e devolve o JSON
                                   →  Protheus decide o que baixar
```

## Antes de commitar

Este repositório é público. Nenhum dado real de cliente entra aqui: nem CNPJ,
nem CPF, nem razão social, nem conta corrente. Os arquivos de teste são
anonimizados pela tarefa 2 do plano, e a verificação é por lista de permissão.
