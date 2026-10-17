# Design: dashboard bancário

**Status:** aceita
**Data:** 2026-10-17

O problema de produto está no [PRD](https://wmreis-labs.github.io/wmreis-docs/mudancas/prd/2026-10-17-dashboard-bancario/).

## Objetivo

A rota `/financial/banking` é a única tela do banco. Ela mostra a conta, o período, o saldo, o fluxo e o extrato, e uma matriz do que se repete por pagador e por recebedor.

## Fora deste desenho

O serviço de banco permanece como está. A tela não soma todas as contas, não exporta planilha, não edita categoria e não calcula saldo corrido. Não há total mensal pronto na API: a matriz agrupa os lançamentos já recebidos.

## Onde entra

`/financial/banking`, no painel já autenticado. A conta, a data inicial e a data final ficam no endereço. Abrir o detalhe de uma conta leva a essa mesma rota, com a conta escolhida.

## O que a pessoa faz

No topo, escolhe a conta e o período. Os atalhos são hoje, sete dias, este mês e mês anterior. O intervalo livre substitui o atalho. O período escrito vale para os números, os gráficos e o extrato.

O saldo atual aparece maior que o resto, com a data em que foi lido. Ao lado, três números do período: entradas, saídas e resultado. O resultado muda de cor conforme o sinal. Quantidade de lançamentos e médias ficam numa linha de apoio, sem o mesmo tamanho do saldo.

Dois gráficos ficam abaixo: a evolução do saldo e o fluxo de caixa, entradas contra saídas. Eixos e dica usam a moeda da conta. O fluxo tem legenda. Sem dados, o gráfico diz que não há movimento no período. Clicar num dia restringe o extrato àquele dia.

O extrato lista os lançamentos do período, do mais recente para o mais antigo, agrupados pelo dia, com subtotal do dia. A busca olha descrição e documento. Um filtro separa entrada e saída. O valor leva sinal e cor só no número. A tabela diz quantos lançamentos estão na tela e quantos existem no período. Carregar mais traz a página seguinte.

O clique num lançamento abre o detalhe: valor, datas, descrição, documento, categoria e conta. O conteúdo técnico do lançamento não ocupa essa vista.

A matriz usa os últimos doze meses, à parte do atalho do topo. São duas tabelas. Recebimentos agrupam créditos pelo pagador. Pagamentos agrupam débitos pelo recebedor. A linha é a pessoa ou a empresa: o documento identifica, o nome rotula. Sem documento, a linha usa o nome e fica marcada como sem documento. Lançamento sem a outra parte vai para “Sem identificação”, com o título ou a descrição. A célula mostra a soma e a quantidade. Linha presente em dois meses ou mais fica em destaque. Mês sem lançamento permanece vazio. Clicar na célula filtra o extrato naquela contraparte e naquele mês.

## Estados

- **Sem conta.** A tela informa que não há conta e aponta o cadastro.
- **Carregando.** Números e gráficos reservam o lugar. O extrato não bloqueia a tela inteira.
- **Período sem movimento.** Números zerados, gráficos com a mensagem de vazio e extrato sem linhas.
- **Falha ao ler.** A tela mostra a mensagem recebida e mantém a conta e o período escolhidos.
- **Contraparte sem documento.** A linha entra na matriz com o nome e a marca de sem documento.
- **Doze meses longos.** A matriz espera o extrato desse intervalo. Se a carga falhar, as duas tabelas mostram a mensagem recebida e o restante da tela continua.
