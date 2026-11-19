# Plano de implementação — Leitura de retorno PIX

> **Para quem executa:** cada tarefa é um ciclo completo de teste e commit. Siga na ordem.
> Os passos usam caixa de seleção (`- [ ]`) para acompanhamento.

**Objetivo:** um serviço HTTP que lê os arquivos de retorno de PIX da pasta do Google Drive do cliente e devolve os pagamentos em JSON para o Protheus dar baixa.

**Arquitetura:** camadas com portas e adaptadores. `domain` tem o modelo e o parser do CNAB, sem nenhum IO. `application` tem o caso de uso e define as portas. `infrastructure` implementa as portas (Google Drive e SQLite). `interface/http` tem a rota. Quem monta as peças concretas é a rota, nunca o caso de uso.

**Tecnologias:** Python 3.12, FastAPI, uv, ruff, pyright, pytest, SQLite, Docker, nginx.

**Especificação:** `specs/001-leitura-retorno-pix/design.md`

**Repositório:** `totvs-oes/svc-py-aviacao-financeiro-leitura-extratos`, branch `main`.

## Restrições globais

Valem para todas as tarefas.

- Python 3.12. Dependências travadas no `uv.lock`.
- Código, nomes, logs e mensagens de commit em inglês. Documentação e texto ao cliente em português.
- Conventional Commits, sem trailer de coautoria de IA.
- Dinheiro em `Decimal`. Nunca `float`, em nenhum ponto.
- `datetime` sempre com fuso. Data de negócio no fuso `America/Sao_Paulo`.
- Valores monetários saem no JSON como string: `"3176.80"`.
- Só a ocorrência `00`, sozinha, significa pago.
- Nenhum pagamento é persistido. O SQLite guarda apenas identidade de arquivo e data de entrega.
- Retenção do registro de entrega: 90 dias.
- Timeout explícito em toda chamada de rede, vindo de `AppSettings`.
- `except Exception` só na fronteira HTTP, sempre com `logger.exception`.
- Segredo lido por `read_secret`, que aceita `NOME_FILE` além de `NOME`. Nunca em código, log ou fixture.
- Sem `print` em código de produção. Sem f-string dentro de chamada de log.
- Complexidade ciclomática máxima 10.
- Piso de cobertura 85%, que nunca desce.
- Nenhum teste toca a rede.
- O repositório é público: nenhum dado real de cliente entra nele.

## Foco de revisão

Cinco situações que a especificação implica, que vão acontecer na vida real e que nenhuma amostra que temos cobre. Cada uma tem o teste indicado na tarefa que é dona do código.

1. **Pagamento recusado pelo banco.** Nenhum dos 30 pagamentos reais tem ocorrência diferente de `00`. Se `pago` vier verdadeiro por engano, o Protheus baixa um título que não foi pago. Teste na tarefa 6.
2. **Arquivo truncado no download.** Linha com menos de 240 colunas precisa virar recusa com motivo, não erro 500. Teste na tarefa 5.
3. **Campo numérico com brancos em vez de zeros.** Acontece em banco que não preenche campo opcional. `int()` em branco derruba a chamada inteira. Teste na tarefa 6.
4. **Valor efetivo diferente do nominal.** Pagamento parcial ou com tarifa descontada. Os dois valores precisam aparecer, nunca um sobrescrever o outro. Teste na tarefa 6.
5. **Mesmo arquivo reenviado com outro id no Drive.** O cliente sobe de novo e vira um id diferente. Precisa aparecer como não entregue, e isso tem que estar explícito no teste para ninguém "consertar" depois. Teste na tarefa 8.

## Estrutura de arquivos

```
server.py                        fábrica da aplicação FastAPI
config.py                        AppSettings e read_secret
domain/
  payment.py                     Payment, Payee, PayerAccount, PixFile, FileKind, PaymentMethod
  exceptions.py                  DomainError, FileRejectedError, IntegrationError, StorageUnavailableError
  cnab/
    occurrences.py               códigos de ocorrência e descrições
    header.py                    leitura e validação do header do arquivo
    banks.py                     tabela de particularidades por banco
    layout.py                    parser dos segmentos A, B e Z
application/
  ports.py                       FileStoragePort, DeliveryLogPort, StoredFileRef
  get_pix_returns.py             caso de uso
infrastructure/
  sqlite_delivery_log.py         adaptador do registro de entrega
  drive_storage.py               adaptador do Google Drive
  logging.py                     configuração de log estruturado
interface/http/
  routes.py                      GET /pix/retornos
  auth.py                        validação da chave de API
  schemas.py                     modelos de resposta
tests/
  fixtures/                      arquivos anonimizados
  test_*.py
```

---

### Tarefa 1: Esqueleto do projeto e portão de qualidade

**Arquivos:**
- Criar: `pyproject.toml`, `.python-version`, `.gitignore`, `.pre-commit-config.yaml`, `.github/workflows/ci.yml`, `.editorconfig`
- Criar: `tests/test_smoke.py`

**Interfaces:**
- Produz: o portão `uv run pytest -q`, `uv run ruff check .`, `uv run ruff format --check .`, `uv run pyright`, que todas as tarefas seguintes usam.

- [ ] **Passo 1: Clonar o repositório e criar a branch**

```bash
git clone git@github.com:totvs-oes/svc-py-aviacao-financeiro-leitura-extratos.git
cd svc-py-aviacao-financeiro-leitura-extratos
git checkout -b feat/pix-return-reader
```

- [ ] **Passo 2: Levar a especificacao para dentro do repositorio**

Copie `design.md` e `tasks.md` para `specs/001-leitura-retorno-pix/` no repositorio do servico. O plano viaja junto com o codigo: quem abrir o repositorio daqui a seis meses precisa encontrar o porque ao lado do que.

- [ ] **Passo 3: Criar `pyproject.toml`**

Projeto `svc-aviacao-financeiro-leitura-extratos`, `requires-python = ">=3.12,<3.13"`.
Dependências: `fastapi`, `uvicorn[standard]`, `pydantic-settings`, `google-api-python-client`, `google-auth`.
Grupo `dev`: `pytest`, `pytest-cov`, `ruff`, `pyright`, `pre-commit`.

Configuração do ruff: `line-length = 100`, regras `E,F,I,N,UP,B,C90,S,G,T20,DTZ,RET,SIM,PL`, `mccabe.max-complexity = 10`.
Em `tests/`, liberar `S101` (uso de `assert`).

Configuração do pytest: `--cov --cov-fail-under=85`.

- [ ] **Passo 4: Escrever o teste de fumaça**

```python
def test_python_version_is_supported() -> None:
    assert sys.version_info[:2] == (3, 12)
```

- [ ] **Passo 5: Rodar o portão completo**

Rode: `uv sync && uv run pytest -q && uv run ruff check . && uv run ruff format --check . && uv run pyright`
Esperado: tudo passa.

- [ ] **Passo 6: Criar o CI em `.github/workflows/ci.yml`**

Em `push` e `pull_request`. Um job que instala o uv, roda `uv sync --frozen` e executa os quatro comandos do passo anterior, nessa ordem.

- [ ] **Passo 7: Criar `.pre-commit-config.yaml`**

Hooks de `ruff check --fix` e `ruff format`.

- [ ] **Passo 8: Commit**

```bash
git add specs pyproject.toml uv.lock .python-version .gitignore .editorconfig .pre-commit-config.yaml .github tests/test_smoke.py
git commit -m "chore: set up project skeleton and quality gate"
```

---

### Tarefa 2: Fixtures anonimizadas

O repositório é público. Os arquivos originais têm CNPJ e razão social de fornecedores reais, a conta corrente da empresa, e CPF com nome de dez funcionárias. Nada disso entra.

**Arquivos:**
- Criar: `scripts/anonymize_fixtures.py`
- Criar: `tests/fixtures/itau_remessa_pix.rem`, `tests/fixtures/itau_retorno_pix.ret`, `tests/fixtures/santander_remessa_pix.rem`, `tests/fixtures/santander_retorno_pix.txt`, `tests/fixtures/santander_varredura_dda.txt`
- Criar: `tests/test_fixtures_are_anonymous.py`

**Interfaces:**
- Produz: os cinco arquivos de fixture que todas as tarefas de 5 em diante usam.

- [ ] **Passo 1: Escrever o teste que prova a anonimização**

A verificação é por lista de permissão, nunca por lista de proibição. Uma lista
do que é proibido teria que conter o dado real, e este repositório é público.
A lista de permissão contém só o que o anonimizador gera, não vaza nada, e
ainda pega o valor que ninguém lembrou de proibir.

```python
FAKE_DOCUMENTS = frozenset(json.loads(Path("tests/fixtures/fake_identities.json").read_text())["documents"])
FAKE_NAMES = frozenset(json.loads(Path("tests/fixtures/fake_identities.json").read_text())["names"])

def test_every_document_in_fixtures_is_fake() -> None:
    for path in _cnab_fixtures():
        for document in extract_documents(path.read_text(encoding="latin-1")):
            assert document in FAKE_DOCUMENTS, f"{path.name} tem documento fora da lista"

def test_every_payee_name_in_fixtures_is_fake() -> None:
    for path in _cnab_fixtures():
        for name in extract_payee_names(path.read_text(encoding="latin-1")):
            assert name in FAKE_NAMES, f"{path.name} tem nome fora da lista"

def test_every_fixture_line_has_240_columns() -> None:
    for path in FIXTURES_DIR.iterdir():
        for line in path.read_text(encoding="latin-1").splitlines():
            assert len(line) == 240, path.name
```

- [ ] **Passo 2: Rodar o teste e ver falhar**

Rode: `uv run pytest tests/test_fixtures_are_anonymous.py -v`
Esperado: FALHA, o diretório de fixtures não existe.

- [ ] **Passo 3: Escrever `scripts/anonymize_fixtures.py`**

Assinatura: `def anonymize(content: str, mapping: Mapping[str, str]) -> str`.

Regra que não pode ser quebrada: **a substituição preserva o comprimento**. O arquivo tem 240 colunas fixas e qualquer deslocamento invalida tudo. Documento trocado por outro de mesmo número de dígitos, nome trocado por outro preenchido com espaços até o mesmo tamanho.

O que trocar: CNPJ e CPF (pagadora e favorecidos), razão social e nome de pessoa, agência e conta da empresa.

O que **não** pode mudar, porque é o que os testes conferem: Seu Número, Nosso Número, valores, datas, códigos de ocorrência, forma de iniciação, totais dos trailers.

O script lê de um diretório passado por argumento. Os originais ficam em `aviao-financeiro/knowledgebase/data_provided_by_the_client.` e **nunca** entram neste repositório.

- [ ] **Passo 4: Gerar as fixtures e rodar o teste**

Rode: `uv run python scripts/anonymize_fixtures.py <dir-originais> tests/fixtures/` e depois `uv run pytest tests/test_fixtures_are_anonymous.py -v`
Esperado: PASSA.

- [ ] **Passo 5: Conferir à mão antes de commitar**

Abra cada fixture e confirme que não sobrou nome de pessoa nem documento real. Este passo não tem automação que substitua: o commit é público e irreversível.

- [ ] **Passo 6: Commit**

```bash
git add scripts/anonymize_fixtures.py tests/fixtures tests/test_fixtures_are_anonymous.py
git commit -m "test: add anonymized CNAB fixtures"
```

---

### Tarefa 3: Configuração e segredos

**Arquivos:**
- Criar: `config.py`, `.env.example`
- Criar: `tests/test_config.py`

**Interfaces:**
- Produz: `AppSettings`, `get_settings() -> AppSettings`, `read_secret(name: str) -> str | None`. As tarefas 8, 9 e 10 consomem.

- [ ] **Passo 1: Escrever os testes**

```python
def test_read_secret_prefers_file_over_environment(tmp_path, monkeypatch) -> None:
    secret_file = tmp_path / "api_key"
    secret_file.write_text("from-file\n")
    monkeypatch.setenv("SERVICE_API_KEY", "from-env")
    monkeypatch.setenv("SERVICE_API_KEY_FILE", str(secret_file))
    assert read_secret("SERVICE_API_KEY") == "from-file"

def test_read_secret_strips_trailing_newline(tmp_path, monkeypatch) -> None: ...

def test_read_secret_returns_none_when_absent(monkeypatch) -> None: ...

def test_settings_reject_empty_api_key(monkeypatch) -> None:
    # pydantic deve recusar a configuracao, nao aceitar chave vazia
```

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_config.py -v`
Esperado: FALHA, `config` não existe.

- [ ] **Passo 3: Implementar `config.py`**

`AppSettings(BaseSettings)` com, no mínimo: `service_api_key: str`, `google_credentials_path: Path`, `drive_pix_folder_id: str`, `drive_timeout_seconds: float = 30.0`, `delivery_log_path: Path`, `delivery_retention_days: int = 90`, `log_level: str = "INFO"`.

`get_settings()` com `functools.lru_cache`.

- [ ] **Passo 4: Rodar e ver passar**

Rode: `uv run pytest tests/test_config.py -v`
Esperado: PASSA.

- [ ] **Passo 5: Escrever `.env.example`**

Uma linha por variável, cada uma com comentário em português dizendo o que faz. Nenhum valor real.

- [ ] **Passo 6: Commit**

```bash
git add config.py .env.example tests/test_config.py
git commit -m "feat: add application settings and secret reading"
```

---

### Tarefa 4: Modelo de domínio, erros e códigos de ocorrência

**Arquivos:**
- Criar: `domain/payment.py`, `domain/exceptions.py`, `domain/cnab/occurrences.py`
- Criar: `tests/test_payment.py`, `tests/test_occurrences.py`

**Interfaces:**
- Produz:
  - `FileKind` (enum: `REMITTANCE`, `RETURN`), `PaymentMethod` (enum: `PIX`, `DDA`)
  - `Payee(name: str | None, document: str | None, pix_key: str | None, initiation_form: str)`
  - `PayerAccount(document: str, branch: str, account: str)`
  - `Payment(client_number: str, bank_number: str | None, payment_date: date, settlement_date: date | None, amount: Decimal, settled_amount: Decimal, occurrences: tuple[str, ...], payee: Payee, authentication: str | None)` com propriedade `is_paid: bool`
  - `PixFile(bank_code: str, file_kind: FileKind, payment_method: PaymentMethod, generated_at: date, payer: PayerAccount, payments: tuple[Payment, ...])` com propriedade `total_amount: Decimal`
  - `PAID_CODE = "00"`, `describe(code: str) -> str`
  - `DomainError`, `FileRejectedError(reason: str)`, `IntegrationError`, `StorageUnavailableError`

Todas as dataclasses são `frozen=True`.

- [ ] **Passo 1: Escrever os testes**

```python
def test_payment_is_paid_only_when_single_occurrence_is_00() -> None:
    assert _payment(occurrences=("00",)).is_paid is True
    assert _payment(occurrences=("PA",)).is_paid is False
    assert _payment(occurrences=("00", "AT")).is_paid is False
    assert _payment(occurrences=()).is_paid is False

def test_total_amount_sums_payments_as_decimal() -> None:
    file = _pix_file(amounts=[Decimal("3176.80"), Decimal("3000.00")])
    assert file.total_amount == Decimal("6176.80")

def test_describe_known_occurrence() -> None:
    assert describe("PA") == "PIX nao efetivado"

def test_describe_unknown_occurrence_returns_the_code() -> None:
    assert describe("ZZ") == "ZZ"
```

Atenção ao terceiro caso do primeiro teste: `00` acompanhado de outro código **não** é pago. O banco devolve até cinco ocorrências.

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_payment.py tests/test_occurrences.py -v`
Esperado: FALHA.

- [ ] **Passo 3: Implementar os três módulos**

A tabela de descrições cobre, no mínimo: `00`, `01`, `02`, `03`, `PA`, `PB`, `PC`, `PD`, `PE`, `PF`, `PG`. Fonte: manual Santander CNAB 240 v11.7, relação 5.

- [ ] **Passo 4: Rodar e ver passar**

Rode: `uv run pytest tests/test_payment.py tests/test_occurrences.py -v`
Esperado: PASSA.

- [ ] **Passo 5: Commit**

```bash
git add domain tests/test_payment.py tests/test_occurrences.py
git commit -m "feat: add payment domain model and occurrence codes"
```

---

### Tarefa 5: Leitura e validação do header

É aqui que mora o anti-objetivo mais importante da especificação: ler um arquivo de DDA como se fosse PIX devolve estrutura válida com conteúdo errado.

**Arquivos:**
- Criar: `domain/cnab/header.py`
- Criar: `tests/test_cnab_header.py`

**Interfaces:**
- Consome: `FileKind`, `PaymentMethod`, `PayerAccount`, `FileRejectedError` da tarefa 4.
- Produz: `FileHeader(bank_code, file_kind, payment_method, generated_at, payer)` e `read_header(content: str) -> FileHeader`, mais `ensure_pix_return(header: FileHeader) -> None`.

Posições, todas 1-based conforme o manual:

| Dado | Registro | Posição |
|---|---|---|
| Código do banco | header de arquivo | 1-3 |
| Tipo de registro | qualquer | 8 |
| Remessa (`1`) ou retorno (`2`) | header de arquivo | 143 |
| Data de geração, `DDMMAAAA` | header de arquivo | 144-151 |
| CNPJ da empresa | header de arquivo | 19-32 |
| Agência | header de arquivo | 53-57 |
| Conta | header de arquivo | 59-70 |
| Forma de lançamento | header de lote | 12-13 |

Forma `45` é PIX. Forma `50` e o segmento `G` indicam DDA.

- [ ] **Passo 1: Escrever os testes**

```python
def test_reads_santander_return_header() -> None:
    header = read_header(fixture("santander_retorno_pix.txt"))
    assert header.bank_code == "033"
    assert header.file_kind is FileKind.RETURN
    assert header.payment_method is PaymentMethod.PIX
    assert header.generated_at == date(2026, 11, 18)

def test_reads_itau_return_header() -> None:
    header = read_header(fixture("itau_retorno_pix.ret"))
    assert header.bank_code == "341"
    assert header.file_kind is FileKind.RETURN

def test_remittance_is_identified_as_remittance() -> None:
    assert read_header(fixture("santander_remessa_pix.rem")).file_kind is FileKind.REMITTANCE

def test_rejects_dda_file() -> None:
    with pytest.raises(FileRejectedError, match="nao e PIX"):
        ensure_pix_return(read_header(fixture("santander_varredura_dda.txt")))

def test_rejects_remittance() -> None:
    with pytest.raises(FileRejectedError, match="nao e retorno"):
        ensure_pix_return(read_header(fixture("santander_remessa_pix.rem")))

def test_rejects_truncated_line() -> None:          # foco de revisao 2
    content = fixture("santander_retorno_pix.txt")[:-50]
    with pytest.raises(FileRejectedError, match="240 colunas"):
        read_header(content)

def test_rejects_empty_content() -> None:
    with pytest.raises(FileRejectedError):
        read_header("")

def test_rejects_content_without_file_header() -> None:
    with pytest.raises(FileRejectedError):
        read_header(_only_trailer_lines())
```

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_cnab_header.py -v`
Esperado: FALHA.

- [ ] **Passo 3: Implementar `domain/cnab/header.py`**

`read_header` recusa quando o conteúdo é vazio, quando alguma linha não tem exatamente 240 colunas, ou quando falta o header de arquivo ou o de lote. A mensagem de `FileRejectedError` diz o motivo em português e nunca cita caminho, host ou credencial.

`ensure_pix_return` recusa quando `file_kind` não é `RETURN` ou `payment_method` não é `PIX`.

- [ ] **Passo 4: Rodar e ver passar**

Rode: `uv run pytest tests/test_cnab_header.py -v`
Esperado: PASSA, os oito testes.

- [ ] **Passo 5: Commit**

```bash
git add domain/cnab/header.py tests/test_cnab_header.py
git commit -m "feat: read and validate CNAB file header"
```

---

### Tarefa 6: Parser dos segmentos e tabela de bancos

**Arquivos:**
- Criar: `domain/cnab/banks.py`, `domain/cnab/layout.py`
- Criar: `tests/fixtures/santander_retorno_pix_com_falha.txt`
- Criar: `tests/test_cnab_banks.py`, `tests/test_cnab_layout.py`

**Interfaces:**
- Consome: `read_header`, `FileHeader` da tarefa 5; o modelo da tarefa 4.
- Produz: `parse_pix_file(content: str) -> PixFile`, `BankQuirks`, `quirks_for(bank_code: str) -> BankQuirks`.

Posições dos segmentos, 1-based:

| Dado | Segmento | Posição |
|---|---|---|
| Seu Número | A | 74-93 |
| Data do pagamento | A | 94-101 |
| Valor nominal | A | 120-134 |
| Nosso Número | A | 135-154 |
| Data real | A | 155-162 |
| Valor real | A | 163-177 |
| Nome do favorecido | A | 44-73 |
| Ocorrências | A | 231-240 |
| Forma de iniciação | B | 15-16 |
| Documento do favorecido | B | 19-32 |
| Chave PIX | B | 128-226 |
| Autenticação | Z | 15-78 |

Valor tem duas casas decimais implícitas. A chave PIX só é preenchida quando a forma de iniciação é `01`, `02`, `03` ou `04`; com `05` o pagamento é por dados bancários e o campo é ignorado.

`BankQuirks(code, name, sends_authentication_segment, blanks_payee_name_on_key_payment, currency_field)`.
Conteúdo inicial da tabela:

| Banco | Segmento Z | Nome em branco por chave | Moeda |
|---|---|---|---|
| `341` Itaú | sim | não | `009` |
| `033` Santander | não | sim | `BRL` |
| `001` Banco do Brasil | a confirmar | a confirmar | a confirmar |

**Banco desconhecido não é recusado.** `quirks_for` devolve um padrão tolerante e o parser registra um aviso no log. Recusar pararia a baixa do dia quando o cliente ligar um banco novo.

- [ ] **Passo 1: Criar a fixture sintética de falha**

`tests/fixtures/santander_retorno_pix_com_falha.txt`, construída à mão a partir da fixture do Santander, com três alterações em pagamentos diferentes:

1. ocorrência `PA` no lugar de `00`;
2. valor real `0000000003000` com valor nominal `0000000003176`, ou seja, efetivo menor que o nominal;
3. campo de valor real preenchido com brancos.

As linhas continuam com 240 colunas e o trailer continua coerente com os valores nominais.

- [ ] **Passo 2: Escrever os testes**

```python
def test_parses_every_real_fixture_and_totals_match_trailer() -> None:
    for name, expected_count, expected_total in [
        ("itau_remessa_pix.rem", 10, Decimal("6020.24")),
        ("itau_retorno_pix.ret", 10, Decimal("6020.24")),
        ("santander_remessa_pix.rem", 14, Decimal("93943.40")),
        ("santander_retorno_pix.txt", 20, Decimal("131139.87")),
    ]:
        parsed = parse_pix_file(fixture(name))
        assert len(parsed.payments) == expected_count
        assert parsed.total_amount == expected_total

def test_client_number_survives_from_remittance_to_return() -> None:
    sent = {p.client_number for p in parse_pix_file(fixture("itau_remessa_pix.rem")).payments}
    received = {p.client_number for p in parse_pix_file(fixture("itau_retorno_pix.ret")).payments}
    assert sent == received

def test_santander_return_records_are_out_of_order() -> None:
    numbers = [p.client_number for p in parse_pix_file(fixture("santander_retorno_pix.txt")).payments]
    assert numbers != sorted(numbers)
    assert len(set(numbers)) == len(numbers)

def test_missing_z_segment_does_not_break_parsing() -> None:
    parsed = parse_pix_file(fixture("santander_retorno_pix.txt"))
    assert all(p.authentication is None for p in parsed.payments)

def test_itau_return_carries_authentication() -> None:
    parsed = parse_pix_file(fixture("itau_retorno_pix.ret"))
    assert all(len(p.authentication or "") == 64 for p in parsed.payments)

def test_blank_payee_name_becomes_none() -> None:
    parsed = parse_pix_file(fixture("santander_retorno_pix.txt"))
    assert any(p.payee.name is None for p in parsed.payments)

def test_pix_key_only_present_for_key_initiation() -> None:
    for p in parse_pix_file(fixture("santander_remessa_pix.rem")).payments:
        if p.payee.initiation_form == "05":
            assert p.payee.pix_key is None
        else:
            assert p.payee.pix_key

def test_rejected_payment_is_not_paid() -> None:              # foco de revisao 1
    parsed = parse_pix_file(fixture("santander_retorno_pix_com_falha.txt"))
    rejected = [p for p in parsed.payments if "PA" in p.occurrences]
    assert len(rejected) == 1
    assert rejected[0].is_paid is False

def test_partial_settlement_keeps_both_amounts() -> None:     # foco de revisao 4
    parsed = parse_pix_file(fixture("santander_retorno_pix_com_falha.txt"))
    partial = [p for p in parsed.payments if p.settled_amount < p.amount]
    assert len(partial) == 1
    assert partial[0].amount == Decimal("3176.80")
    assert partial[0].settled_amount == Decimal("3000.00")

def test_blank_numeric_field_becomes_zero() -> None:          # foco de revisao 3
    parsed = parse_pix_file(fixture("santander_retorno_pix_com_falha.txt"))
    assert any(p.settled_amount == Decimal("0.00") for p in parsed.payments)

def test_unknown_bank_uses_tolerant_defaults() -> None:
    quirks = quirks_for("999")
    assert quirks.sends_authentication_segment is False
```

- [ ] **Passo 3: Rodar e ver falhar**

Rode: `uv run pytest tests/test_cnab_layout.py tests/test_cnab_banks.py -v`
Esperado: FALHA.

- [ ] **Passo 4: Implementar `banks.py` e `layout.py`**

Campo numérico só com brancos vira `Decimal("0.00")`, nunca exceção. Data zerada vira `None`.
`layout.py` não importa nada de `infrastructure` nem de `application`, e não faz IO.

- [ ] **Passo 5: Rodar e ver passar**

Rode: `uv run pytest tests/test_cnab_layout.py tests/test_cnab_banks.py -v`
Esperado: PASSA, os onze testes.

- [ ] **Passo 6: Commit**

```bash
git add domain/cnab tests/fixtures/santander_retorno_pix_com_falha.txt tests/test_cnab_layout.py tests/test_cnab_banks.py
git commit -m "feat: parse CNAB payment segments with per-bank quirks"
```

---

### Tarefa 7: Portas, caso de uso e teste de arquitetura

**Arquivos:**
- Criar: `application/ports.py`, `application/get_pix_returns.py`
- Criar: `tests/test_get_pix_returns.py`, `tests/test_arquitetura.py`

**Interfaces:**
- Consome: `parse_pix_file`, `read_header`, `ensure_pix_return`, `FileRejectedError`.
- Produz:
  - `StoredFileRef(id: str, name: str)`
  - `FileStoragePort` com `list_files(self, *, day: date) -> Sequence[StoredFileRef]` e `download(self, file_id: str) -> str`
  - `DeliveryLogPort` com `previous_delivery(self, file_id: str) -> datetime | None`, `record_delivery(self, file_id: str, name: str, *, at: datetime) -> None`, `purge_older_than(self, cutoff: datetime) -> int`
  - `RejectedFile(id: str, name: str, reason: str)`
  - `DeliveredFile(ref: StoredFileRef, file: PixFile, already_delivered: bool, first_delivered_at: datetime | None)`
  - `PixReturnsResult(day: date, files: tuple[DeliveredFile, ...], rejected: tuple[RejectedFile, ...])`
  - `GetPixReturns(*, storage: FileStoragePort, log: DeliveryLogPort, clock: Callable[[], datetime], retention_days: int)` com `execute(self, day: date) -> PixReturnsResult`

- [ ] **Passo 1: Escrever o teste de arquitetura**

```python
def test_application_never_imports_infrastructure() -> None:
    for path in Path("application").rglob("*.py"):
        source = path.read_text(encoding="utf-8")
        assert "import infrastructure" not in source
        assert "from infrastructure" not in source

def test_domain_imports_nothing_from_the_project() -> None:
    for path in Path("domain").rglob("*.py"):
        source = path.read_text(encoding="utf-8")
        for forbidden in ("application", "infrastructure", "interface", "config"):
            assert f"import {forbidden}" not in source
            assert f"from {forbidden}" not in source
```

- [ ] **Passo 2: Escrever os testes do caso de uso**

Use dublês em memória das duas portas. Nenhum toca rede nem disco.

```python
def test_returns_every_file_of_the_day() -> None: ...

def test_marks_file_as_already_delivered_on_second_call() -> None:
    use_case.execute(DAY)
    result = use_case.execute(DAY)
    assert result.files[0].already_delivered is True
    assert result.files[0].first_delivered_at is not None

def test_second_call_still_returns_the_payments() -> None:
    # nao esconder dado e requisito, nao detalhe: anti-objetivo da especificacao
    use_case.execute(DAY)
    result = use_case.execute(DAY)
    assert result.files[0].file.payments

def test_rejected_file_does_not_stop_the_others() -> None:
    storage = FakeStorage({"ok.txt": VALID, "dda.txt": DDA_CONTENT})
    result = use_case.execute(DAY)
    assert len(result.files) == 1
    assert len(result.rejected) == 1
    assert "nao e PIX" in result.rejected[0].reason

def test_day_without_files_returns_empty_result() -> None:
    assert use_case.execute(DAY).files == ()

def test_purges_delivery_records_older_than_retention() -> None:
    use_case.execute(DAY)
    assert log.purged_cutoff == CLOCK_NOW - timedelta(days=90)

def test_rejected_file_is_not_recorded_as_delivered() -> None: ...
```

- [ ] **Passo 3: Rodar e ver falhar**

Rode: `uv run pytest tests/test_get_pix_returns.py tests/test_arquitetura.py -v`
Esperado: FALHA.

- [ ] **Passo 4: Implementar as portas e o caso de uso**

O caso de uso: lista os arquivos do dia, baixa cada um, valida pelo header, interpreta, consulta a entrega anterior, grava a entrega dos que passaram, purga os registros vencidos e devolve o resultado. `FileRejectedError` em um arquivo não interrompe os demais.

As portas são `typing.Protocol`.

- [ ] **Passo 5: Rodar e ver passar**

Rode: `uv run pytest tests/test_get_pix_returns.py tests/test_arquitetura.py -v`
Esperado: PASSA.

- [ ] **Passo 6: Commit**

```bash
git add application tests/test_get_pix_returns.py tests/test_arquitetura.py
git commit -m "feat: add ports and the pix returns use case"
```

---

### Tarefa 8: Registro de entrega em SQLite

**Arquivos:**
- Criar: `infrastructure/sqlite_delivery_log.py`
- Criar: `tests/test_sqlite_delivery_log.py`

**Interfaces:**
- Consome: `DeliveryLogPort` da tarefa 7.
- Produz: `SqliteDeliveryLog(database_path: Path)`, que implementa a porta.

Tabela `delivery` com `file_id TEXT PRIMARY KEY`, `file_name TEXT NOT NULL`, `first_delivered_at TEXT NOT NULL`, `delivery_count INTEGER NOT NULL`.

Nenhuma coluna guarda pagamento, documento ou valor.

- [ ] **Passo 1: Escrever os testes**

```python
def test_first_delivery_returns_none_then_records(tmp_path) -> None: ...

def test_second_delivery_keeps_the_first_timestamp(tmp_path) -> None:
    log.record_delivery("id1", "a.txt", at=T1)
    log.record_delivery("id1", "a.txt", at=T2)
    assert log.previous_delivery("id1") == T1

def test_second_delivery_increments_the_counter(tmp_path) -> None: ...

def test_purge_removes_only_records_older_than_cutoff(tmp_path) -> None: ...

def test_timestamps_round_trip_with_timezone(tmp_path) -> None:
    log.record_delivery("id1", "a.txt", at=T1)
    assert log.previous_delivery("id1").tzinfo is not None

def test_same_file_resent_with_a_new_drive_id_counts_as_new(tmp_path) -> None:  # foco de revisao 5
    log.record_delivery("id1", "a.txt", at=T1)
    assert log.previous_delivery("id2") is None

def test_schema_has_no_column_for_payment_data(tmp_path) -> None:
    columns = {row[1] for row in connection.execute("PRAGMA table_info(delivery)")}
    assert columns == {"file_id", "file_name", "first_delivered_at", "delivery_count"}
```

O penúltimo teste documenta um comportamento conhecido e aceito: o mesmo arquivo reenviado com outro id é tratado como novo. Está aqui para que ninguém mude isso sem decidir.

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_sqlite_delivery_log.py -v`
Esperado: FALHA.

- [ ] **Passo 3: Implementar `SqliteDeliveryLog`**

Cria o esquema na primeira conexão. Valor sempre por parâmetro, nunca por interpolação. Timestamps em ISO 8601 com fuso.

- [ ] **Passo 4: Rodar e ver passar**

Rode: `uv run pytest tests/test_sqlite_delivery_log.py -v`
Esperado: PASSA.

- [ ] **Passo 5: Commit**

```bash
git add infrastructure/sqlite_delivery_log.py tests/test_sqlite_delivery_log.py
git commit -m "feat: add sqlite delivery log"
```

---

### Tarefa 9: Adaptador do Google Drive

**Arquivos:**
- Criar: `infrastructure/drive_storage.py`, `infrastructure/logging.py`
- Criar: `tests/test_drive_storage.py`

**Interfaces:**
- Consome: `FileStoragePort`, `StoredFileRef` da tarefa 7; `StorageUnavailableError` da tarefa 4.
- Produz: `GoogleDriveStorage(*, credentials_path: Path, folder_id: str, timeout_seconds: float)`.

Autenticação por conta de serviço, escopo `drive.readonly`. A listagem filtra por pasta e por `modifiedTime` do dia pedido, no fuso de São Paulo.

Nenhum teste toca a rede: o cliente da API do Google entra por injeção.

- [ ] **Passo 1: Escrever os testes**

```python
def test_lists_only_files_of_the_requested_day() -> None: ...

def test_downloads_file_content_as_latin1_text() -> None: ...

def test_timeout_is_passed_to_the_client() -> None: ...

def test_api_failure_becomes_storage_unavailable_error() -> None:
    with pytest.raises(StorageUnavailableError):
        storage.list_files(day=DAY)

def test_error_message_does_not_leak_credentials_or_paths() -> None:
    try:
        storage.list_files(day=DAY)
    except StorageUnavailableError as error:
        assert "credentials" not in str(error)
        assert str(CREDENTIALS_PATH) not in str(error)

def test_paginates_until_the_last_page() -> None: ...
```

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_drive_storage.py -v`
Esperado: FALHA.

- [ ] **Passo 3: Implementar o adaptador e o log estruturado**

Toda falha da API do Google vira `StorageUnavailableError` com mensagem limpa, e o detalhe original vai para `logger.exception`.

`infrastructure/logging.py` configura log em uma linha por evento, com `request_id`. Nunca registra conteúdo de arquivo, documento ou valor.

- [ ] **Passo 4: Rodar e ver passar**

Rode: `uv run pytest tests/test_drive_storage.py -v`
Esperado: PASSA.

- [ ] **Passo 5: Commit**

```bash
git add infrastructure/drive_storage.py infrastructure/logging.py tests/test_drive_storage.py
git commit -m "feat: add google drive storage adapter"
```

---

### Tarefa 10: API HTTP

**Arquivos:**
- Criar: `interface/http/routes.py`, `interface/http/auth.py`, `interface/http/schemas.py`, `server.py`
- Criar: `tests/test_api.py`

**Interfaces:**
- Consome: `GetPixReturns` da tarefa 7, os adaptadores das tarefas 8 e 9, `get_settings` da tarefa 3.
- Produz: `create_app() -> FastAPI` e a rota `GET /pix/retornos?data=YYYY-MM-DD`.

Resposta conforme a seção 9 do design. Valores monetários como string. É aqui, e só aqui, que `except Exception` é permitido.

- [ ] **Passo 1: Escrever os testes, com `TestClient`**

```python
def test_returns_200_with_the_files_of_the_day() -> None: ...

def test_monetary_values_are_strings() -> None:
    body = client.get("/pix/retornos?data=2026-11-18", headers=AUTH).json()
    assert body["arquivos"][0]["valorTotal"] == "131139.87"
    assert body["arquivos"][0]["pagamentos"][0]["valor"] == "3176.80"

def test_occurrence_code_and_description_are_both_present() -> None: ...

def test_missing_api_key_returns_401() -> None:
    assert client.get("/pix/retornos?data=2026-11-18").status_code == 401

def test_wrong_api_key_returns_401() -> None: ...

def test_invalid_date_returns_422() -> None:
    assert client.get("/pix/retornos?data=18-11-2026", headers=AUTH).status_code == 422

def test_day_without_files_returns_200_and_empty_list() -> None:
    body = client.get("/pix/retornos?data=2026-01-01", headers=AUTH).json()
    assert body["arquivos"] == []

def test_rejected_file_appears_in_its_own_list() -> None: ...

def test_storage_failure_returns_502_without_leaking_detail() -> None:
    response = client.get("/pix/retornos?data=2026-11-18", headers=AUTH)
    assert response.status_code == 502
    assert "Traceback" not in response.text
    assert "credentials" not in response.text

def test_health_endpoint_needs_no_key() -> None:
    assert client.get("/health").status_code == 200
```

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_api.py -v`
Esperado: FALHA.

- [ ] **Passo 3: Implementar a rota, a autenticação e os schemas**

A chave é comparada com `secrets.compare_digest`. A rota monta `GetPixReturns` com os adaptadores concretos. `/health` não exige chave e não toca o Drive.

- [ ] **Passo 4: Rodar e ver passar, com o portão completo**

Rode: `uv run pytest -q && uv run ruff check . && uv run ruff format --check . && uv run pyright`
Esperado: tudo passa, cobertura acima de 85%.

- [ ] **Passo 5: Commit**

```bash
git add interface server.py tests/test_api.py
git commit -m "feat: expose pix returns over http"
```

---

### Tarefa 11: Docker, compose e nginx

Copiar o que já funciona em `aviacao_geren_fatur_corte_prod`, sem reinventar: `deploy/nginx/` e `deploy/lego/`.

**Arquivos:**
- Criar: `Dockerfile`, `compose.yaml`, `.dockerignore`, `deploy/nginx/*`, `deploy/lego/run.sh`
- Criar: `tests/test_compose_exposure.py`

- [ ] **Passo 1: Escrever o teste de exposição**

```python
def test_only_nginx_publishes_ports() -> None:
    compose = yaml.safe_load(Path("compose.yaml").read_text())
    publishing = {name for name, svc in compose["services"].items() if svc.get("ports")}
    assert publishing == {"nginx"}

def test_application_has_no_secret_in_the_compose_file() -> None:
    raw = Path("compose.yaml").read_text()
    for forbidden in ("SERVICE_API_KEY:", "password", "secret:"):
        assert forbidden not in raw

def test_application_runs_as_non_root() -> None:
    assert "USER app" in Path("Dockerfile").read_text()
```

- [ ] **Passo 2: Rodar e ver falhar**

Rode: `uv run pytest tests/test_compose_exposure.py -v`
Esperado: FALHA.

- [ ] **Passo 3: Escrever o `Dockerfile`**

Base `public.ecr.aws/docker/library/python:3.12-slim-bookworm`, fixada no menor e não no patch. `uv sync --frozen --no-dev --no-cache` montando o uv só no passo de build. Usuário `app` sem privilégio. `CMD` com `uvicorn server:app --proxy-headers`.

- [ ] **Passo 4: Escrever o `compose.yaml`**

Dois serviços: `nginx` e `api`. Só o nginx publica porta. A chave da API e a credencial do Google entram como arquivos de segredo montados. Volume nomeado para o SQLite. `mem_limit` em cada serviço. Rotação de log em 10 MB e 3 arquivos.

- [ ] **Passo 5: Copiar e adaptar a configuração do nginx**

De `aviacao_geren_fatur_corte_prod/deploy/nginx/`. Manter o limite de taxa, o limite de corpo e a subida automática para HTTPS quando o certificado aparecer. Ajustar as rotas para `/pix/` e `/health`.

- [ ] **Passo 6: Subir local e conferir de ponta a ponta**

```bash
docker compose up -d --build
curl -s localhost:8000/health
curl -s -H "X-API-Key: <chave>" "localhost:8000/pix/retornos?data=2026-11-18"
```
Esperado: `/health` responde 200, e a rota responde 200 ou 502 conforme a credencial do Drive estar configurada.

- [ ] **Passo 7: Commit**

```bash
git add Dockerfile compose.yaml .dockerignore deploy tests/test_compose_exposure.py
git commit -m "chore: containerize the service behind nginx"
```

---

### Tarefa 12: Documentação e diagramas

**Arquivos:**
- Criar: `README.md`, `CONTRIBUTING.md`, `CONTEXT.md`, `docs/architecture.md`, `docs/deploy.md`, `docs/engineering-standards.md`
- Criar: `docs/adr/0001-registrar-decisoes-de-arquitetura.md`, `0002-arquitetura-em-camadas.md`, `0003-ingles-no-codigo-e-portugues-na-documentacao.md`, `0004-nao-persistir-dado-financeiro.md`, `0005-seu-numero-como-chave-de-baixa.md`
- Criar: `diagramas/01-fluxo-pix-ponta-a-ponta.mmd` e `.svg`, `diagramas/02-camadas.mmd` e `.svg`
- Modificar: `README.md` do repositório, que hoje tem só o título com o acento quebrado

- [ ] **Passo 1: Escrever o `README.md`**

Seções: o que o serviço faz em um parágrafo, como rodar em desenvolvimento, a rota com um exemplo de requisição e resposta, configuração, como rodar os testes e o portão de qualidade, e links para `docs/`.

- [ ] **Passo 2: Escrever `CONTEXT.md`**

Glossário: remessa, retorno, borderô, Seu Número, Nosso Número, ocorrência, segmento, forma de iniciação, varredura DDA, conciliação. Um conceito, um nome.

- [ ] **Passo 3: Escrever os cinco ADRs**

Formato curto: contexto, decisão, consequências. O 0005 registra por que o Seu Número é a chave, com a evidência dos quatro arquivos.

- [ ] **Passo 4: Gerar os diagramas**

Fonte `.mmd` e `.svg` renderizado lado a lado, como no faturamento e corte. O primeiro é o fluxo de ponta a ponta da seção 7 do design. O segundo são as camadas.

- [ ] **Passo 5: Conferir que a documentação bate com o código**

Rode os comandos do README numa pasta limpa e confirme que funcionam como escrito.

- [ ] **Passo 6: Commit e abrir o PR**

```bash
git add README.md CONTRIBUTING.md CONTEXT.md docs diagramas
git commit -m "docs: add readme, architecture notes, adrs and diagrams"
git push -u origin feat/pix-return-reader
```

---

## Pendências externas que não bloqueiam a implementação

Todas as tarefas acima rodam com as fixtures, sem rede. O que falta de fora só aparece na primeira execução real:

| Pendência | Com quem | Afeta |
|---|---|---|
| Compartilhar a pasta do Drive com a conta de serviço, como leitora | o TI do cliente | Tarefa 9 em produção |
| Confirmar se o Workspace do cliente aceita conta externa | o TI do cliente | Tarefa 9 em produção |
| Chave da API combinada com o Protheus | o desenvolvedor ADVPL | Tarefa 10 em produção |
| Manual de layout do Itaú | o financeiro do cliente | Confirmação da tabela da tarefa 6 |
| Particularidades do Banco do Brasil | o financeiro do cliente | Linha da tabela da tarefa 6 |
