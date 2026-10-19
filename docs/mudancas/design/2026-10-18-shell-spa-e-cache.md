# Design: shell único e cache de dados no painel

**Status:** aberta
**Data:** 2026-10-18

## Objetivo

Trocar de menu ou de rota dentro do painel autenticado deve mexer só na área de conteúdo. Menu lateral, barra superior, estado do menu (recolhido, grupos abertos) e dados já lidos permanecem. Hoje a troca parece recarregar a tela. O desenho propõe uma única fonte para o shell e para os dados, em etapas pequenas que não mudam contrato de API.

## Fora deste desenho

Nenhuma rota, payload ou regra de domínio muda. Não há troca de CRA por Vite, nem de MUI. O fluxo de flipping, o calendário e o dashboard bancário mantêm o comportamento descrito nos seus desenhos. Redux continua dono da sessão (`auth`).

## Onde entra

Todas as rotas filhas de `PrivateLayout` em `src/App.tsx`: dashboard, imóveis, pessoas, contratos, flipping, calendário, financeiro, orçamento, configurações e perfil.

## O que a pessoa vê hoje

- Ao abrir uma tela cujo código ainda não foi baixado, aparece "Carregando..." no lugar de tudo, menu incluso, quando a navegação não consegue manter a tela anterior.
- Ao entrar em cada tela, o menu pisca: a tela anterior limpa a configuração do layout e a nova a escreve de novo.
- Voltar para uma lista já aberta mostra o carregamento de novo e refaz a chamada, mesmo segundos depois.
- Filtros, busca e página de uma lista se perdem ao abrir um detalhe e voltar.

## Diagnóstico

Medido no código de `hostto` em 2026-10-18.

1. **Cada página monta o seu próprio layout.** 30 arquivos de `src/pages` renderizam `<BaseLayoutLogged>` com `navigation`, `toolbar` e `secondaryNavigation`. Dentro de `PrivateLayout`, esse componente não desenha nada: copia a configuração para o `LayoutProvider` por efeito e devolve só os filhos. O shell real é um segundo `BaseLayoutLogged` (`asShell`). Resultado: um ciclo de montar, escrever, limpar e reescrever a cada rota, e o shell renderiza várias vezes por troca. O hook `useLayoutConfig` existe e não é usado.
2. **`Suspense` acima das rotas.** O fallback em `App.tsx` envolve `<Routes>`, então o carregamento preguiçoso de uma página compete com o shell inteiro. O `startTransition` do router ameniza, mas não garante.
3. **Dados sem cache na maior parte do app.** Só 6 hooks usam React Query (banco e orçamento). As demais telas buscam em `useEffect` com `useState` e `loading` local, e 32 páginas têm o seu próprio indicador de carga. O `QueryClient` em `src/index.tsx` usa os padrões: `staleTime` zero, então até as telas com Query refazem a chamada a cada montagem.
4. **Estado de lista local à página.** Busca, filtros e paginação ficam em `useState`, então morrem ao desmontar.
5. **Guarda de rota lê `localStorage`.** `RequireAuth` consulta `authTokens` a cada renderização, enquanto o Redux já guarda os tokens. Duas fontes para a mesma sessão.
6. **Páginas muito grandes.** `FlippingDetailPage` tem 2070 linhas, `BudgetEditorPage` 1141, `CalendarPage` 939 e `PropertyDetailPage` 942. Carga, formulário e apresentação vivem juntas, o que impede reaproveitar objeto entre rotas.

## Proposta

### 1. Rota declara o layout; a página só renderiza conteúdo

Cada rota privada ganha `handle` com a configuração do shell (título, menu secundário, ações). O `PrivateLayout` lê `useMatches()` e passa ao shell. A página deixa de renderizar `BaseLayoutLogged`. Para ações que dependem de estado da tela (botão que usa dado carregado), a página usa `useLayoutConfig` com valor memoizado, que passa a ser o único caminho.

Resultado: o shell não é limpo entre rotas e renderiza uma vez por troca. `asShell` e o ramo `layoutActions && !asShell` saem de `BaseLayoutLogged`.

### 2. `Suspense` dentro do shell

O fallback sai de volta de `<Routes>` e entra em volta do `<Outlet />` em `PrivateLayout`, com um esqueleto na área de conteúdo. O shell nunca desmonta por causa de chunk. O fallback do nível raiz fica só para rotas públicas.

### 3. React Query como camada de dados

- `QueryClient` com `staleTime` de 60 s, `gcTime` de 10 min, `refetchOnWindowFocus` desligado e `retry` 1.
- Fábrica de chaves por domínio, em `src/hooks/<domínio>/keys.ts`: `['properties', 'list', filtros]`, `['properties', 'detail', id]`.
- Cada serviço de `src/services/*` ganha um hook de leitura e outros de escrita. Escrita invalida a chave do domínio, como já faz `useBudgetMutations`.
- `placeholderData: keepPreviousData` nas listas com busca e página, para não piscar entre filtros.
- Prefetch no `onMouseEnter` do menu e da linha de lista para o detalhe.
- `useRequest` permanece para ações pontuais (enviar, assinar), que não são leitura.

### 4. Estado de lista no endereço

Busca, filtros e página ficam em `searchParams`, com um hook `useListParams`, do mesmo modo que o dashboard bancário já guarda conta e período. Voltar do detalhe restaura a lista e o link pode ser compartilhado.

### 5. Um só componente de carga e erro

`PageState` (carregando, vazio, erro com tentar de novo) substitui os 32 indicadores locais e o backdrop de tela cheia. Carga inicial ocupa só a área de conteúdo. Recarga em segundo plano mostra uma barra fina no topo.

### 6. Sessão numa fonte só

`RequireAuth` e `RedirectIfAuth` leem o Redux (`PrivateRoute` já faz isso) e saem de `App.tsx`. O `localStorage` fica como persistência, não como fonte de leitura.

### 7. Quebrar as páginas grandes

Ordem: `FlippingDetailPage`, `BudgetEditorPage`, `PropertyDetailPage`, `CalendarPage`. Cada uma divide em hook de dados, seções por aba e componentes de apresentação. Abas viram rotas aninhadas (`/flipping/:id/financeiro`), com `Outlet`, para trocar de aba sem refazer a carga do cabeçalho.

## Estados

- **Troca de menu.** Menu e barra superior ficam. A área de conteúdo mostra o esqueleto só se a tela não tem dado em cache.
- **Volta a uma tela já vista.** Mostra o dado em cache na hora e atualiza em segundo plano se passou do `staleTime`.
- **Falha ao ler.** A área de conteúdo mostra a mensagem recebida e o botão de tentar de novo. O shell continua usável.
- **Sessão expirada.** O interceptor de `baseApi` renova o token. Sem renovação, o Redux limpa a sessão e a guarda leva ao login.
- **Escrita bem-sucedida.** A lista e o detalhe do domínio são invalidados. A tela não recarrega.

## Etapas e ordem

| Etapa | Entrega | Risco |
| --- | --- | --- |
| 1 | `Suspense` dentro do shell e `QueryClient` com `staleTime` | Baixo |
| 2 | Rotas com `handle` e remoção do `BaseLayoutLogged` das páginas, domínio por domínio, começando por dashboard e persons | Médio: 30 páginas e testes de layout |
| 3 | Migrar leituras para React Query: properties, persons, contracts, flipping, calendário, financeiro | Médio: chaves e invalidação |
| 4 | `useListParams` nas listas | Baixo |
| 5 | `PageState` e guarda via Redux | Baixo |
| 6 | Quebra das páginas grandes e abas em rotas | Alto: fazer uma por vez |

Cada etapa sai em PR próprio, com `npm run lint`, `npm test -- --watchAll=false` e `npm run build`.

## Como saber que melhorou

- Trocar de menu não desmonta o shell. Teste de componente: o nó do menu permanece o mesmo entre duas rotas.
- Entrar numa lista já vista não dispara chamada dentro do `staleTime`.
- O número de renderizações do shell por troca de rota cai de várias para uma, medido com o Profiler do React.
- Nenhuma tela mostra "Carregando..." de página inteira depois do primeiro acesso.
