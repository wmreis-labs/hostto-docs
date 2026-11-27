# Design: link de pagamento na reserva

**Status:** aberta
**Data:** 2026-11-27

O problema de produto está no [PRD do Testto Pay](https://wmreis-labs.github.io/wmreis-docs/mudancas/prd/2026-11-21-testto-pay/). A decisão está no [ADR](https://wmreis-labs.github.io/wmreis-docs/mudancas/adr/2026-11-21-testto-pay-pagamentos/). As rotas, os estados da cobrança e a confirmação do pagamento estão no [design do serviço](https://wmreis-labs.github.io/airtestto-docs/mudancas/design/2026-11-21-pagamentos-testto-pay/).

## Objetivo

Na reserva, quem cuida dela gera o link de cartão, copia ou envia ao hóspede e vê se foi pago, sem sair do Hostto.

A tela mostra o estado que a API devolve. Ela não fala com o provedor e não manda valor: o valor do link é o valor em aberto da própria reserva, calculado no backend.

## Fora deste desenho

- **Pix e boleto.** O link é só de cartão.
- **Envio pelo WhatsApp do ATO.** Compartilhar usa o que o aparelho oferece. O envio pelo canal da empresa fica para outra tarefa.
- **Estorno e cancelamento do link.** Ficam no console do Testto Pay.
- **Conciliação e extrato do provedor.** Ficam no console do Testto Pay.

## Onde entra

O bloco **Link de pagamento** fica no painel da reserva no calendário, na seção Financeiro, logo abaixo de Valor, Pago e Pendente.

O bloco aparece em reserva e pré-reserva que não estão canceladas. Nos outros tipos de evento, ele não aparece.

| Papel | O que faz |
| --- | --- |
| `Admin`, `Operacional` e `Financeiro` | Geram o link, copiam e compartilham |
| `Leitor` | Vê o estado e copia o link |

A API confere a mesma regra.

## O que a pessoa faz

### Gerar o link

O botão **Gerar link de pagamento** fica habilitado quando a reserva tem valor em aberto. Sem valor em aberto, o botão fica desabilitado com a dica "A reserva não tem valor em aberto".

O botão abre uma confirmação com:

- **Valor:** o pendente da reserva, só para leitura;
- **Parcelamento máximo:** de 1 a 12, padrão 12;
- **Parcelas sem juros:** de 0 ao parcelamento máximo, padrão 0;
- **Validade:** de 1 a 30 dias, padrão 7.

Confirmar chama `POST /properties/{id}/calendars/{calendarId}/events/{eventId}/charge` só com as opções. A resposta traz a cobrança, e o bloco passa a mostrar o link.

Se o valor da reserva mudou depois do link, a mesma ação gera um link novo. O backend cancela o anterior.

### Copiar e compartilhar

Com o link aberto, o bloco mostra:

- o estado;
- o valor;
- a validade;
- o link, com os botões **Copiar link** e **Compartilhar**.

Compartilhar usa o compartilhamento do aparelho quando ele existe. Quando não existe, copia o link e avisa.

### Acompanhar

Ao abrir a reserva com link aberto, a tela consulta `GET /payments/charges/{chargeId}` para trazer o estado atual. Assim o link vencido aparece como expirado mesmo antes do job diário.

| Estado | O bloco mostra |
| --- | --- |
| `aberta` | "Aguardando pagamento", com o link e a validade |
| `paga` | "Pago", com a data e as parcelas; sem botão de gerar |
| `expirada` | "Link expirado" e o botão **Gerar novo link** |
| `cancelada` | "Link cancelado" e o botão **Gerar novo link** |
| `falhou` | "Falha ao gerar" e o botão **Tentar de novo** |
| `estornada` | "Estornado" |

Quando o pagamento é confirmado, o backend soma o valor em Pago e recalcula o Pendente. A tela mostra esses números ao reabrir a reserva.

## Estados

- **Gerando.** O botão mostra progresso e fica desabilitado até a resposta.
- **Sem valor em aberto (409).** A tela mostra a mensagem da API.
- **Falha no provedor (502).** A tela mostra "O provedor não gerou o link agora; tente de novo".
- **Sem permissão (403).** A tela mostra a mensagem da API.
- **Leitor.** Vê o estado e copia o link, sem os botões de gerar.
