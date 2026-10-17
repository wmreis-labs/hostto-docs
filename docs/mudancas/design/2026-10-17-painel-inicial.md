# Design: painel inicial

**Status:** aceita
**Data:** 2026-10-17

O problema de produto está no [PRD](https://wmreis-labs.github.io/wmreis-docs/mudancas/prd/2026-10-17-painel-inicial/).

## Objetivo

A rota `/dashboard` é a primeira tela depois do login. Ela mostra o dia nas casas e, em seguida, o que pede atenção em dinheiro, contrato e flipping. Cada bloco abre a tela que já existe.

## Fora deste desenho

Esta tela não redesenha o calendário de ocupação nem o dashboard do banco. Não desenha gráfico de fluxo. Não lista pessoas nem o total de imóveis. Não percorre o calendário de cada imóvel: usa a lista de próximos eventos e a lista de imóveis que a tela já pode ler. Se uma estadia em curso não vier nessa lista, o bloco do dia não inventa o dado. A lacuna fica para uma decisão de serviço.

## Onde entra

`/dashboard`, no painel já autenticado. É a rota de abertura.

## O que a pessoa faz

O bloco do dia ocupa a largura principal. Três grupos, nesta ordem: quem chega hoje, quem sai hoje, quem permanece. Cada linha traz o nome do imóvel, o título da estadia, as datas e o status. Pendente e confirmado aparecem. Cancelado, bloqueio e concluído ficam de fora desse bloco. A origem da reserva não entra no título. Clicar na linha abre o calendário daquele imóvel, no evento. Um atalho no topo do bloco abre a grade de ocupação.

Ao lado, ou abaixo no celular, três blocos menores e do mesmo peso.

O dinheiro mostra o saldo somado das contas, e as entradas, as saídas e o resultado do mês corrente. O período fica escrito junto dos números. O resultado muda de cor conforme o sinal. Não há gráfico. Clicar abre o dashboard do banco.

Os contratos mostram quantos ainda não foram assinados e uma lista curta com o nome e o status: rascunho, confirmado pelo cliente ou enviado para assinatura. Clicar abre a lista de contratos.

O flipping mostra os projetos em busca ou em andamento, com o nome e o status. Clicar abre o flipping.

## Estados

- **Carregando.** Cada bloco reserva o lugar. Um bloco lento não esconde os outros.
- **Dia sem estadia.** O bloco diz que não há chegada, saída nem hospedagem hoje.
- **Estadia sem imóvel resolvido.** A linha usa o título da estadia e marca o imóvel como não encontrado.
- **Sem conta.** O bloco de dinheiro diz que não há conta e aponta o cadastro.
- **Mês sem movimento.** Saldo permanece. Entradas, saídas e resultado zerados, com o período escrito.
- **Sem contrato pendente.** O bloco diz que não há contrato aguardando assinatura.
- **Sem flipping aberto.** O bloco diz que não há projeto em busca nem em andamento.
- **Falha num bloco.** Esse bloco mostra a mensagem recebida. Os demais continuam.
