# Design — Leitura de retorno PIX

Serviço que lê os arquivos de retorno bancário de PIX e devolve os pagamentos
em JSON para o Protheus dar baixa no contas a pagar.

Status: proposta, aguardando aprovação.
Data: 2026-11-19.

## 1. Por que existe

Hoje a baixa dos pagamentos de PIX no Protheus é manual: o financeiro do cliente confere
borderô por borderô, por número e banco. São cerca de 600 PIX por mês.

O banco já devolve, todo dia, um arquivo com o que foi pago. O que falta é
alguém ler esse arquivo e entregar o conteúdo ao Protheus num formato que ele
entenda. É isso e só isso que este serviço faz.

A baixa em si, a decisão do que baixar e a escrita no Protheus são da equipe
ADVPL. Este serviço não decide nada de negócio.

## 2. Objetivos

- Ler os arquivos de retorno de PIX depositados pelo cliente numa pasta do
  Google Drive.
- Interpretar o layout CNAB 240 e devolver os pagamentos em JSON.
- Atender os bancos que o cliente usa hoje, sem reescrever nada a cada banco novo.
- Ser chamado pelo Protheus por HTTP, com autenticação.
- Registrar o que já foi entregue, sem esconder dado de ninguém.

## 3. Não-objetivos

- Não dá baixa, não escreve no Protheus, não chama o Protheus.
- Não decide se um pagamento deve ou não ser baixado.
- Não lê a pasta sozinho, não tem agendador, não roda por conta própria.
- Não processa DDA nesta fase. A estrutura prevê a entrada, a implementação não.
- Não guarda dado financeiro. Nenhum pagamento é persistido.

## 4. Anti-objetivos

O que faria este projeto fracassar mesmo funcionando:

- **Entregar um pagamento como pago quando o banco recusou.** O arquivo traz
  os dois misturados. Se a informação do código de ocorrência se perder no
  caminho, o Protheus baixa título não pago e o erro só aparece na conciliação.
- **Esconder um arquivo que o Protheus não processou.** Por isso a API devolve
  tudo o que existe na data e apenas marca o que já foi entregue.
- **Interpretar o layout errado em silêncio.** Ler um arquivo de DDA como se
  fosse PIX devolve estrutura válida com conteúdo errado, que é pior que erro.
- **Vazar a credencial do Google.** Quem tiver aquela chave lê a pasta
  financeira do cliente inteira.

## 5. Restrições

| Restrição | Consequência |
|---|---|
| A pasta é um Google Drive compartilhado de fora da nossa organização | Precisa de conta de serviço, e o cliente tem que compartilhar a pasta com o e-mail dela |
| Não temos acesso ao painel do provedor da VM | As portas 80 e 443 provavelmente seguem fechadas; o serviço nasce em HTTP atrás do nginx, igual ao faturamento e corte |
| O endpoint de baixa do Protheus ainda não existe | Não bloqueia: o Protheus é quem chama, não o contrário |
| O manual do Itaú ainda não chegou | Os campos do Itaú estão validados por engenharia reversa sobre dois arquivos reais, não por documentação |
| Quem implementa está começando | Cada peça de infraestrutura precisa se pagar; a estrutura precisa guiar, não só permitir |

## 6. Abordagens consideradas

**A. Espelho reduzido do `aviacao_geren_fatur_corte_prod`.** As mesmas camadas
(`domain`, `application`, `infrastructure`, `interface`), o mesmo ferramental
(`uv`, `ruff`, `pyright`, `pytest`, `pre-commit`, CI no GitHub), os mesmos
documentos (constituição, padrões de engenharia, ADRs, `specs/`, `diagramas/`).
Sem Celery, sem Postgres, sem Alembic.

**B. Três módulos sem camadas.** `cnab/`, `drive/`, `api/`, mesmo ferramental.
Menos cerimônia no primeiro dia.

**C. Script com um arquivo.** Descartada: contraria o pedido explícito de
robustez e não sobrevive à entrada do DDA.

**Escolhida: A.** Dois motivos. O encaixe é real, não decorativo: existem
exatamente dois sistemas externos (Drive e SQLite) e o parser é função pura sem
IO, que é o caso de livro para portas e adaptadores. E a consistência com o
projeto que já roda significa que a IA que for ajudar na implementação tem um
exemplo pronto e funcionando para copiar, o que vale mais que qualquer
explicação escrita.

## 7. Arquitetura

```
interface/http/     rota, autenticação, schemas de entrada e saída
      ↓
application/        caso de uso ObterRetornosPix, definição dos ports
      ↓
domain/             modelo de pagamento, layout CNAB, tabela de bancos, erros
      ↑
infrastructure/     adaptador do Google Drive, adaptador do SQLite, log
```

`application` recebe ports e nunca importa `infrastructure`. Quem monta as
peças concretas é a rota. Um `tests/test_arquitetura.py` quebra o build se
alguém furar essa regra, igual ao projeto de faturamento.

O parser do CNAB mora em `domain` e não faz IO: recebe texto, devolve objetos.
É o que permite testar os quatro arquivos reais sem rede, e o que vai permitir
o DDA entrar depois sem encostar em nada do que já existe.

### Fluxo de uma chamada

```
Protheus → GET /pix/retornos?data=2026-11-19  (header com a chave)
    → lista os arquivos da subpasta PIX com essa data no Drive
    → para cada arquivo: baixa o conteúdo
                         confere pelo header que é CNAB 240, retorno e PIX
                         interpreta segmentos A, B e Z
    → consulta no SQLite quais já foram entregues
    → grava a entrega
    → devolve JSON com todos os arquivos e todos os pagamentos
```

## 8. O layout: um só, com particularidades por banco

Os quatro arquivos reais (Itaú e Santander, remessa e retorno) foram lidos com
um único conjunto de posições, e os totais bateram com os trailers. O manual do
Banco do Brasil confirma as mesmas posições. Não existem quatro layouts: existe
o CNAB 240 da Febraban, serviço 20, forma de lançamento 45.

Campos que o serviço extrai:

| Campo | Onde | Observação |
|---|---|---|
| Seu Número | segmento A, 74-93 | Sequencial do Protheus. É a chave que liga o pagamento ao título |
| Nosso Número | segmento A, 135-154 | Atribuído pelo banco |
| Data real do pagamento | segmento A, 155-162 | `DDMMAAAA` no arquivo, ISO no JSON |
| Valor real | segmento A, 163-177 | Duas casas decimais implícitas, `Decimal` no código |
| Valor nominal | segmento A, 120-134 | Para detectar divergência |
| Ocorrências | segmento A, 231-240 | Até cinco códigos de dois caracteres |
| Nome do favorecido | segmento A, 44-73 | Vem em branco no Santander quando o PIX foi por chave |
| CNPJ ou CPF do favorecido | segmento B, 19-32 | Conferência, nunca chave |
| Forma de iniciação | segmento B, 15-16 | `01` telefone, `02` e-mail, `03` CPF/CNPJ, `04` aleatória, `05` dados bancários |
| Chave PIX | segmento B, 128-226 | Só quando a iniciação é por chave |
| Autenticação | segmento Z, 15-78 | Opcional. O Itaú envia, o Santander não |
| Conta pagadora | header do arquivo | Banco, agência e conta da empresa |

Particularidades conhecidas, que vão numa tabela declarativa única:

| | Itaú (341) | Santander (033) | BB (001) |
|---|---|---|---|
| Segmento Z | envia | não envia | a confirmar |
| Nome no retorno | preenchido | branco em PIX por chave | a confirmar |
| Tipo da moeda (A 102-104) | `009` | `BRL` | a confirmar |
| Ordem dos registros no retorno | igual à remessa | reordenada | a confirmar |

Banco novo é uma entrada nessa tabela. Nunca um `if` dentro do parser.

## 9. Contrato da API

`GET /pix/retornos?data=YYYY-MM-DD`

A data é a de aparição do arquivo na pasta, não a do conteúdo. Os dois valores
divergem: um arquivo com `191126` no nome trouxe movimento do dia 18. A data do
conteúdo vai no JSON, em `dataGeracao`.

Autenticação por chave fixa em header, combinada com o desenvolvedor ADVPL e lida de arquivo
de segredo.

```json
{
  "data": "2026-11-19",
  "arquivos": [
    {
      "nome": "FORN_R5D5_01_191126P_MOV.TXT",
      "idDrive": "1a2b3c...",
      "banco": "033",
      "tipoArquivo": "RETORNO",
      "formaPagamento": "PIX",
      "dataGeracao": "2026-11-18",
      "jaEntregue": false,
      "primeiraEntregaEm": null,
      "contaPagadora": { "cnpj": "12345678000199", "agencia": "01234", "conta": "000005678" },
      "quantidadePagamentos": 20,
      "valorTotal": "131139.87",
      "pagamentos": [
        {
          "seuNumero": "0000087585",
          "nossoNumero": "900000001",
          "dataPagamento": "2026-11-18",
          "dataEfetiva": "2026-11-18",
          "valor": "3176.80",
          "valorEfetivo": "3176.80",
          "ocorrencias": ["00"],
          "pago": true,
          "favorecido": {
            "nome": "FORNECEDOR EXEMPLO LTDA    ",
            "documento": "98765432000188",
            "chavePix": null,
            "formaIniciacao": "05"
          },
          "autenticacao": null
        }
      ]
    }
  ]
}
```

Valores monetários trafegam como string para não perder precisão em quem
consome. `pago` é verdadeiro apenas quando a única ocorrência é `00`.

### Códigos de ocorrência

Só `00` significa que o dinheiro saiu. A tabela completa está no manual do
Santander, relação 5. Os relevantes:

| Código | Significado |
|---|---|
| `00` | Crédito ou débito efetivado |
| `01` | Insuficiência de fundos |
| `02` | Cancelado pelo pagador |
| `PA` | PIX não efetivado |
| `PC` | Conta do recebedor encerrada |
| `PF` | CPF/CNPJ do recebedor não bate com o titular |

O serviço devolve o código e a tradução. Quem decide o que fazer é o Protheus.

### Erros

| Situação | Resposta |
|---|---|
| Chave ausente ou errada | 401 |
| Data em formato inválido | 422 |
| Nenhum arquivo na data | 200 com `arquivos` vazio |
| Arquivo não é CNAB 240, ou não é retorno, ou não é PIX | O arquivo entra na resposta em `arquivosRecusados`, com o motivo. Os demais são processados |
| Drive fora do ar ou credencial inválida | 502 |

Nenhuma mensagem de erro expõe stack, host, caminho ou credencial.

## 10. Registro de entrega

SQLite num arquivo, em volume montado. Uma tabela com o identificador do
arquivo no Drive, o nome, a data da primeira entrega e a contagem de entregas.

**Nenhum pagamento é gravado.** Sem CNPJ, sem valor, sem nome de fornecedor. Os
dados passam pela API e vão embora na resposta. Com isso não existe dado pessoal
em repouso, o backup deixa de ser um problema de LGPD, e a limpeza vira higiene.

Registros com mais de noventa dias são apagados na própria chamada. Sem serviço
separado, sem agendador.

## 11. Segurança

Em ordem de estrago evitado:

1. **Chave da conta de serviço do Google.** Montada como arquivo de segredo,
   nunca no repositório, na imagem, no log ou em fixture. Mesmo padrão do
   `config.read_secret` do faturamento e corte.
2. **Permissão de leitura apenas.** A conta de serviço entra na pasta como
   leitora. Hoje o convite é de contribuidor; precisa ser pedido ao TI do cliente.
3. **Chave da API** em header, de arquivo de segredo, comparada com
   `secrets.compare_digest`.
4. **Só o nginx publica porta.** A aplicação fica na rede interna do compose.
   Limite de taxa e de corpo no nginx. Um teste no CI trava a tabela de
   exposição, como em `tests/test_compose_exposure.py`.
5. **HTTPS automático quando as portas abrirem.** O `lego` do faturamento e
   corte, copiado. Com `ACME_DOMAINS` vazio ele fica parado e o nginx segue em
   HTTP.
6. **Log sem dado financeiro.** Nome do arquivo, contagem, duração, erro.
   Nunca o conteúdo. `request_id` em toda linha.
7. **Container com usuário sem privilégio**, imagem base fixada no menor,
   dependências travadas no `uv.lock`.

## 12. Padrões de código

Herdados do `docs/engineering-standards.md` do faturamento e corte:

- Código, log e commits em inglês. Documentação e texto ao cliente em português.
- Conventional Commits, sem trailer de coautoria de IA.
- Timeout explícito em toda chamada de rede, vindo das configurações.
- `except Exception` só na fronteira HTTP, sempre com `logger.exception`.
- Erro de domínio separado de erro de integração.
- `Decimal` para dinheiro, desde a primeira linha. Nunca `float`.
- `datetime` sempre com fuso. Data de negócio no fuso de São Paulo.
- Sem `print` em código de produção.
- Sem f-string em chamada de log.
- Complexidade ciclomática barrada acima de 10.
- Piso de cobertura de 85%, que nunca desce.
- Teste de comportamento observável, nunca de estrutura interna.
- Correção de defeito começa por um teste que falha.

## 13. Testes

Os quatro arquivos reais entram como fixture: remessa e retorno do Itaú,
remessa e retorno do Santander. Nenhum teste toca a rede.

O que precisa estar coberto:

- Os quatro arquivos são lidos e os totais batem com os trailers.
- Um arquivo de DDA colocado na pasta PIX é recusado com motivo.
- Um arquivo de remessa é recusado, porque não é retorno.
- Pagamento com ocorrência diferente de `00` vem com `pago` falso.
- Pagamento por chave no Santander, com nome em branco, é lido sem erro.
- Ausência do segmento Z não quebra a leitura.
- Valor divergente entre nominal e efetivo aparece nos dois campos.
- Segunda chamada na mesma data devolve os mesmos arquivos, agora com
  `jaEntregue` verdadeiro.
- Chamada sem chave devolve 401.
- O Drive indisponível devolve 502 e não vaza detalhe.

## 14. Fora de escopo, mas previsto na estrutura

O DDA entra depois, como um parser novo atrás do mesmo port, com sua própria
rota. Nada do PIX muda quando isso acontecer. A subpasta DDA já existe na pasta
do cliente.

## 15. O que fica a critério de quem implementa

- Organização interna dos módulos dentro de cada camada.
- Nomes de função e de variável, respeitando o glossário.
- Formato exato das mensagens de log.
- Biblioteca de acesso ao Drive, desde que atrás do port.
- Estratégia de paginação na listagem do Drive, se necessária.

## 16. Pendências externas

| Pendência | Com quem | Bloqueia |
|---|---|---|
| Compartilhar a pasta com a conta de serviço, como leitora | o TI do cliente | Sim, o acesso ao Drive |
| Confirmar se o Workspace do cliente permite compartilhar com conta externa | o TI do cliente | Sim |
| Chave da API, combinada com o Protheus | o desenvolvedor ADVPL | Não, pode ser gerada e informada depois |
| Qual data o Protheus vai mandar | o desenvolvedor ADVPL | Não, já definido como a de aparição na pasta |
| Manual de layout do Itaú | o financeiro do cliente | Não, os campos estão validados contra arquivos reais |
