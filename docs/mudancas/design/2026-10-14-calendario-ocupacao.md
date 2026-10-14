# Design: calendário de ocupação

**Status:** aceita
**Data:** 2026-10-14

O problema de produto está no [PRD](https://wmreis-labs.github.io/wmreis-docs/mudancas/prd/2026-10-14-calendario-ocupacao/).

## Objetivo

A rota `/calendar` mostra a ocupação de todos os imóveis. O calendário de um imóvel é o detalhe.

## Fora deste desenho

Criar e editar o evento continuam no fluxo atual. Esta tela não lista contrato nem pessoa para montar a grade. Não há calendário externo. O serviço de eventos permanece como está.

## Onde entra

`/calendar`, no painel já autenticado. No computador e no celular a primeira vista é a grade. A lista continua no controle que já alterna calendário e lista.

## O que a pessoa faz

Na grade, cada linha é um imóvel: a inicial do nome, a foto quando ela chegar, o nome e a cidade. As colunas são 21 dias a partir da segunda-feira da semana em foco. Anterior recua sete dias, próximo avança sete, hoje volta à semana corrente.

A estadia é uma barra da data inicial à data final já gravadas, na cor do status, só com o título. Apontar mostra as datas e o status. Clicar na barra abre o evento. Clicar no imóvel abre o detalhe.

No detalhe, o calendário daquele imóvel oferece mês, semana e dia, e a criação por seleção de intervalo, como hoje. Todos os imóveis volta à grade no período em que a pessoa estava. No computador, a lateral troca de imóvel sem sair do detalhe e sem voltar o mês. No celular, o seletor faz o mesmo.

Na lista, a pessoa escolhe o intervalo e percorre os eventos em páginas. Abrir uma linha abre o mesmo evento.

## Estados

- **Primeira abertura.** A tela espera a lista de imóveis. A grade aparece em seguida, com a inicial do nome. A foto preenche quando a imagem estiver pronta.
- **Período carregando.** A grade ou o calendário permanecem. Uma barra fina indica a carga. A tela inteira não bloqueia.
- **Sem imóvel.** A grade informa que não há imóvel.
- **Sem evento no período.** A linha do imóvel fica vazia.
- **Falha num imóvel.** Os demais continuam na grade. A tela mostra a mensagem recebida.
- **Salvando evento.** A tela bloqueia só enquanto grava, como hoje.
- **Atalho de evento.** Se a abertura traz um evento, a tela vai ao detalhe daquele imóvel e abre o evento, sem montar antes a grade de todos.
