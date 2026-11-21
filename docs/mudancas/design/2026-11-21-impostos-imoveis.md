# Design: impostos dos imóveis

**Status:** aberta
**Data:** 2026-11-21

O problema de produto está no [PRD](https://wmreis-labs.github.io/wmreis-docs/mudancas/prd/2026-11-21-impostos-imoveis-socios/). A decisão está no [ADR](https://wmreis-labs.github.io/wmreis-docs/mudancas/adr/2026-11-21-apuracao-impostos-imoveis/). As rotas e as regras de cálculo estão no [design do serviço](https://wmreis-labs.github.io/airtestto-docs/mudancas/design/2026-11-21-apuracao-impostos-tax/).

## Objetivo

O Hostto ganha três lugares:

- **`/financial/impostos`:** mostra a receita da base, a apuração, a parte de cada sócio e as guias.
- **`/financial/recebimentos`:** mostra os créditos do Inter que ainda não viraram receita de um imóvel.
- **Ficha do imóvel:** mostra os sócios do imóvel.

As telas mostram o número que a API devolve. Nenhuma soma, alíquota ou rateio é feito no front.

## Fora deste desenho

- **Regras de cálculo:** alíquota, presunção, adicional, arredondamento, rateio, vencimento e bloqueio do fechamento ficam no design do serviço.
- **Cadastro da posse:** a tela de pessoa e de imóvel continua onde está. Aqui a ficha só mostra a posse.
- **Despesas comuns e conta do sócio:** seguem em aberto no ADR.
- **Gestão Financeira:** a lista de lançamentos não muda, além do atalho para os recebimentos.

## Onde entra

- O menu ganha o item **Impostos** no grupo Financeiro, depois de Gestão Financeira. O cartão de mesmo nome entra na página `/financial`.
- A área é `financeiro`:
    - `Admin` e `Financeiro` alteram;
    - `Leitor` vê;
    - `Operacional` não vê o item e recebe a tela de sem acesso se abrir o endereço.
- A API confere a mesma regra.

`/financial/impostos` tem cinco abas:

- Receita;
- Apuração;
- Sócios;
- Guias;
- Configuração.

A aba e o ano ficam no endereço (`?aba=apuracao&ano=2026`). O ano padrão é o corrente, e a aba padrão é Apuração. O seletor de ano fica no topo e vale para as quatro primeiras abas.

`/financial/recebimentos` é aberta por dois atalhos:

- o botão "Recebimentos a classificar", no cabeçalho de Gestão Financeira, com a quantidade de pendentes;
- o aviso de pendência de Impostos.

## O que a pessoa faz

### Receita

A aba mostra uma matriz:

- **Linhas:** os imóveis da apuração.
- **Colunas:** os meses do ano.
- **Linha de total:** total do mês.
- **Linha de trimestre:** soma de cada trimestre, debaixo do terceiro mês.

Os números vêm de `GET /tax/base`.

A célula mostra a receita do imóvel no mês. Clicar nela abre um painel lateral com os lançamentos daquele imóvel e daquele mês:

- os lançamentos vêm de `GET /financial`, filtrado pelo imóvel e pela faixa do mês;
- cada linha traz data do crédito, competência, categoria, origem e valor;
- o painel tem um link para Gestão Financeira.

O mês com pendência leva um ícone de aviso no cabeçalho da coluna. A dica diz quantos recebimentos estão pendentes e leva a `/financial/recebimentos` filtrada no mês.

### Apuração

Uma tabela por trimestre, com uma linha para cada mês e uma linha para o trimestre:

| Linha | Colunas |
| --- | --- |
| Mês | Base, PIS, COFINS, estado |
| Trimestre | Base, base presumida, IRPJ, adicional, CSLL, estado |

O estado é `aberto` ou `fechado`.

- **Período fechado:** aparece com a data do fechamento. Se a API devolver diferença, a linha mostra o valor fechado e, ao lado, a diferença com sinal, com a dica "lançamentos mudaram depois do fechamento".
- **Fechar:** o botão aparece na linha aberta para `Admin` e `Financeiro`. Ele pede confirmação mostrando os valores que serão gravados e as guias que serão criadas, e chama `POST /tax/periods/{period}/close`.
- **Recusa:** se a API recusar, a confirmação vira a lista de pendências que voltou, cada uma com o atalho para resolver: recebimentos do mês ou ficha do imóvel.
- **Detalhe:** clicar numa linha abre o detalhe do período, de `GET /tax/periods/{period}`. Ele mostra a base por imóvel, a posse usada, a parte de cada sócio em cada tributo e os parâmetros usados.

### Sócios

A aba mostra uma tabela por sócio, de `GET /tax/partners`:

- **Linhas:** receita dos imóveis dele, PIS, COFINS, IRPJ, CSLL e total de impostos.
- **Colunas:** os meses, os trimestres, o ano e a média mensal.

Um seletor alterna entre ver todos os sócios ou um só.

O botão "Exportar XLSX" usa o exportador que o Hostto já tem. Ele gera uma planilha com uma aba por sócio, com os mesmos números da tela. Período aberto sai marcado como "aberto" na planilha.

### Guias

A aba mostra uma lista das guias do ano, de `GET /tax/guides`:

- **Colunas:** tipo, código de receita, período, valor apurado, valor da guia, vencimento e estado.
- **Filtros:** tipo e estado.
- **Ordem:** do vencimento mais próximo para o mais distante.
- **Avisos:** guia vencida e não paga fica em destaque. Guia cujo valor difere do apurado mostra a diferença.

Ações para `Admin` e `Financeiro`:

- **Emitir:** abre um formulário com código de barras, valor principal, multa, juros e vencimento. Salva com `PATCH /tax/guides/{id}`.
- **Nova guia:** cria guia manual. No tipo `PARCELAMENTO`, aceita a série de parcelas, com quantidade, valor e primeiro vencimento.
- **Conferir pagamento:**
    - mostra os débitos do extrato que a API sugere (`GET /tax/guides/{id}/candidates`), com data, descrição e valor;
    - a pessoa escolhe um e confirma (`POST /tax/guides/{id}/payment`), e a guia passa a paga com a data do débito;
    - sem candidato, o diálogo diz que não há débito com esse valor perto do vencimento.
- **Desfazer pagamento:** volta a guia para emitida.
- **Cancelar:** pede confirmação.

### Configuração

| Bloco | Mostra | Quem altera |
| --- | --- | --- |
| Imóveis da apuração | Lista com vigência; o imóvel é escolhido entre os cadastrados | `Admin` e `Financeiro` |
| Contas de receita | Contas do banco cujos créditos viram recebimento | `Admin` e `Financeiro` |
| Parâmetros | Por tributo e vigência: presunção, alíquota, código de receita e vencimento | `Admin` abre uma vigência nova; os outros só leem |

### Recebimentos a classificar

`/financial/recebimentos` tem duas abas: Pendentes e Regras.

**Pendentes**

A aba lista os créditos de `GET /financial/bank-receipts?status=pendente`:

- **Colunas:** data, pagador, descrição e valor.
- **Filtro:** mês.
- **Outros estados:** um seletor mostra também os classificados e os ignorados, para consulta.
- **Sugestão:** quando a API traz sugestão de imóvel, a linha mostra o nome do imóvel e o botão "Aceitar sugestão".

Classificar abre um formulário:

- **Destino:** "Receita de imóvel" ou "Não é aluguel".
- **Receita de imóvel:**
    - a pessoa escolhe imóvel, categoria (aluguel ou diária) e o mês de competência, que vem preenchido com o mês do crédito;
    - "Repartir" acrescenta partes, e o formulário mostra quanto falta para fechar o valor do crédito;
    - salvar fica bloqueado até as partes somarem o valor.
- **Não é aluguel:** a pessoa escolhe o motivo (evento, aporte, estorno, devolução de sócio, outro).
- **Regra:**
    - a caixa "Usar para os próximos créditos deste pagador" cria a regra junto;
    - o campo da regra vem preenchido com o documento do pagador, ou com o nome quando não há documento;
    - a pessoa pode trocar para "descrição contém".

Salvar chama `classify` ou `ignore`. O crédito sai da lista de pendentes. Um classificado pode ser reaberto, e a API recusa quando o mês já foi fechado. Nesse caso, a tela mostra a mensagem recebida.

**Regras**

A aba lista as regras na ordem:

- **Colunas:** campo, condição, valor, destino (imóvel e categoria, ou motivo) e quantos recebimentos já casaram.
- **Ações:** editar, apagar e subir ou descer a prioridade, só para `Admin` e `Financeiro`.

### Ficha do imóvel

A ficha ganha o cartão **Sócios**:

- nome de cada dono, com link para a pessoa, e a posse em porcentagem;
- se a soma não dá 100%, o cartão mostra um aviso com a soma atual;
- sem dono, o cartão diz que o imóvel não tem sócio cadastrado.

Os dados vêm da rota de donos do imóvel.

## Estados

- **Carregando.** Cada aba reserva o lugar da tabela. O seletor de ano e as abas continuam usáveis.
- **Ano sem receita.** A matriz mostra os imóveis com células vazias, e a apuração mostra os períodos sem valor e sem botão de fechar.
- **Sem imóvel na apuração.** Receita, Apuração e Sócios dizem que nenhum imóvel está na apuração e apontam para Configuração.
- **Pendências.** Um aviso no topo de `/financial/impostos` resume as pendências do ano, com atalho para cada uma:
    - quantos recebimentos estão pendentes;
    - quais imóveis estão sem posse completa.
- **Fechamento recusado.** A confirmação mostra as pendências que a API devolveu e não fecha.
- **Período fechado com diferença.** O valor fechado fica, e a diferença aparece ao lado.
- **Guia vencida.** A linha fica em destaque até ser paga ou cancelada.
- **Sem candidato de pagamento.** O diálogo explica e não deixa confirmar.
- **Leitor.** Vê tudo e não vê os botões de fechar, emitir, criar, conferir pagamento, classificar nem os de regra e configuração.
- **Falha ao ler ou gravar.** A tela mostra a mensagem recebida e mantém aba, ano e filtros.
