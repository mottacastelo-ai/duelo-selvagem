---
name: novo-tema
description: "Adiciona um novo tema selecionável ao Duelo Selvagem (ex: dinossauros, carros, super-heróis, oceano) — baralho, atributos e título próprios, sem substituir os temas já existentes; o jogador escolhe o tema na tela inicial. Use quando o usuário pedir um novo tema, baralho ou variação do jogo."
---

# Novo Tema — Duelo Selvagem

Gera uma variação temática completa do jogo (arquivo único `duelo-selvagem.html` / `index.html`) trocando **apenas** conteúdo — nunca a lógica de rede, estado ou UI. O motor (Supabase Realtime, reconexão, multiplayer 2-4, sons de UI, telas) é agnóstico ao tema e não deve ser tocado.

## Regra inegociável (não pedir, não negociar)

Toda imagem e todo som usados em uma carta precisam ter **licença aberta verificável**: CC0, domínio público, CC BY, CC BY-SA (Wikimedia Commons, NOAA, iNaturalist, Xeno-canto, Freesound, etc.). Fontes CC BY-NC só entram com aprovação explícita do usuário no momento, carta a carta — nunca por padrão. **Nunca** usar mídia sem licença clara, mesmo alegando "uso pessoal": o site é público (GitHub Pages), então o risco de direito autoral não desaparece. Se não achar fonte licenciada para algum item, deixe `sound: null` ou troque a imagem por outra do mesmo tema — não force.

Não inventar dados numéricos (peso, velocidade, datas, etc.). Cada atributo precisa vir de uma fonte checável; anote a fonte em `info.fontes`.

## Arquitetura: temas são plugáveis, nunca substituem um ao outro

O jogo guarda **todos os temas ao mesmo tempo** num objeto `const THEMES = { selvagem: {...}, aereo: {...}, ... }` (buscar `const THEMES = {` no arquivo). Cada entrada tem `{ id, label, emoji, title, attrLabels, attrOrder, deck }`. O jogador escolhe o tema num seletor na tela inicial (`#themePicker`, populado via `renderThemePicker()` a partir de `Object.keys(THEMES)`); quem cria a sala fixa `state.themeId`, que é sincronizado para os demais jogadores via os payloads `roster`/`start`/`stateSync` (broadcast Realtime) — **nunca deixe um tema novo quebrar isso**: todo tema novo precisa entrar no `THEMES` como uma chave nova, nunca sobrescrevendo `selvagem` nem qualquer tema já existente, a menos que o usuário peça explicitamente para substituir/remover um tema.

O resto do motor lê o tema ativo via `currentTheme()` (retorna `THEMES[state.themeId]`) — nunca há mais uma constante solta `DECK`/`ATTR_LABELS`/`ATTR_ORDER` no escopo global.

## O que compõe um "tema" (uma entrada dentro de `THEMES`)

1. **Identidade**: `id` (slug único, chave do objeto `THEMES`), `label` (nome curto pro seletor, ex: "Aéreo"), `emoji` (ícone do seletor), `title` (usado em `<title>` e `.brand-mark` quando esse tema está ativo — trocado em runtime por `applyThemeChrome()`).
2. **Paleta de cores**: hoje é **global** (`:root` no CSS, linhas ~12-25 — `--bg`, `--surface`, `--card-bg`, `--accent`, etc.), compartilhada por todos os temas. Só mexer nela se o usuário pedir explicitamente uma paleta própria por tema (isso exigiria uma extensão do motor, não fazer sem alinhar antes).
3. **Atributos**: `attrLabels` e `attrOrder` dentro da entrada do tema — exatamente 5 atributos numéricos comparáveis, cada um `{ label, unit }`. Ex. para dinossauros: peso, comprimento, ano de existência (mais antigo vence ou mais recente, escolher um critério), força de mordida, altura.
4. **O baralho `deck`** dentro da entrada do tema — um array de N objetos (padrão 32, mas pergunte ao usuário; **sem limite artificial**), cada um neste formato exato:

```js
{
  id: 'slug_unico', name: 'Nome Exibido', species: 'Nome científico/técnico opcional', emoji: '🦖',
  attrs: { atributo1: 50, atributo2: 104, atributo3: 12, atributo4: 77, atributo5: 7100 },
  image: { url: 'https://...', author: 'Autor', license: 'CC BY-SA 4.0' },
  sound: { url: 'https://...', description: 'o que é o som', author: 'Autor/fonte', license: 'CC BY 4.0' }, // ou sound: null
  wikipediaUrl: 'https://pt.wikipedia.org/wiki/...',
  info: {
    habitat: '...', alimentacao: '...', comportamento: '...', conservacao: '...',
    curiosidades: ['fato 1', 'fato 2', 'fato 3', 'fato 4', 'fato 5'],
    fontes: 'Nome das fontes usadas'
  }
}
```

Os nomes de campo dentro de `info` (`habitat`, `alimentacao`, etc.) são fixos no template de exibição do card — se o tema não for de animais, reaproveite os mesmos 4 campos com sentido adaptado (ex. carros: `habitat`→"onde é fabricado/usado", `alimentacao`→"combustível/motor", `comportamento`→"como se comporta na pista/uso", `conservacao`→"raridade/produção"). Não troque a estrutura do objeto, só o conteúdo.

## Fluxo de trabalho (otimizado para gastar poucos tokens)

0. **Checar primeiro o repo de referência do usuário** — `https://github.com/yanexr/trump-cards` (MIT, fornecido pelo Léo). Ele já tem baralhos prontos em `client/assets/carddecks/`:
   - `cars.json` — Carros (116 cartas)
   - `airplanes.json` — Aviões (83 cartas)
   - `rockets.json` — Foguetes (49 cartas)
   - Índice: `client/assets/carddecks/localCardDecks.json` (lista os baralhos disponíveis e a contagem de cartas — confira aqui primeiro se apareceu um baralho novo desde a última vez).

   Se o tema pedido bater com um desses baralhos, **use-o como base em vez de pesquisar do zero**:
   - Puxe o JSON via `gh api repos/yanexr/trump-cards/contents/client/assets/carddecks/<tema>.json --jq '.content' | base64 -d` (ou WebFetch em `https://raw.githubusercontent.com/yanexr/trump-cards/main/client/assets/carddecks/<tema>.json`).
   - `characteristics[]` já dá os 5 atributos prontos (nome + se maior é melhor) → vira `ATTR_LABELS`/`ATTR_ORDER` quase direto (só falta a unidade).
   - Cada carta em `cards[]` já tem `name`, `values[]` (na mesma ordem de `characteristics`), `imageAttr` e `imageLic` — **é a licença real da imagem, já resolvida por eles**. Ainda assim, aplique a regra inegociável: só aceite se `imageAttr`/`imageLic` mapear para CC0/domínio público/CC BY/CC BY-SA; licenças tipo "pixabay" ou outras não listadas exigem a mesma aprovação explícita do usuário que uma CC BY-NC exigiria.
   - `imagePath` é relativo (ex: `Cars/tesla-plaid.jpg`) — a imagem real está em `client/assets/images/<imagePath>` no repo. Referencie via `https://raw.githubusercontent.com/yanexr/trump-cards/main/client/assets/images/<imagePath>` (não precisa baixar nem hospedar de novo).
   - O que o repo **não** tem: som, `id`/slug, emoji, `wikipediaUrl`, e todo o bloco `info` (habitat/alimentação/comportamento/conservação/curiosidades/fontes). Isso ainda precisa ser pesquisado — mas é bem menos trabalho que atributos+imagem+licença do zero.
   - Se o baralho tiver mais cartas do que o usuário pediu, escolha uma seleção representativa (ou pergunte quais critérios usar); se tiver menos, complete o restante com pesquisa nova (passo 2) mantendo o mesmo formato de atributos.

   Se o tema **não** bater com nenhum baralho do repo (ex: dinossauros, super-heróis), pule direto para o passo 2 — pesquisa completa do zero, como já era.

1. **Perguntar ao usuário** (uma vez, via AskUserQuestion se não estiver claro): tema, quantidade de itens, os 5 atributos numéricos desejados, e se quer paleta de cores nova ou manter a atual. (Pode ser feito antes ou depois do passo 0 — se o tema já bate com um baralho do repo, a pergunta de atributos pode nem ser necessária, já que `characteristics[]` resolve isso.)

2. **Pesquisa em paralelo via subagentes** — nunca pesquisar tudo na conversa principal (estoura contexto à toa). Se o passo 0 achou um baralho no repo, os agentes só completam o que falta (som, `info`, e itens extras além do que o baralho já cobre); se não achou, pesquisam tudo do zero. Divida os itens em lotes de 6-8 e dispare um `Agent` (`general-purpose`, `run_in_background: false`, todos na mesma mensagem para rodar em paralelo) por lote. Prompt de cada agente deve incluir:
   - A lista exata de itens do lote — se já vieram do repo, inclua `name`/`attrs`/`image` já resolvidos, pedindo só para completar `id`/`emoji`/`sound`/`wikipediaUrl`/`info`; se não, peça tudo.
   - O formato de objeto JS acima, campo a campo.
   - A regra de licença inegociável (colar o parágrafo acima).
   - Pedir que devolva **apenas** os objetos JS prontos (um por item), com valores reais checados (Wikipedia, fontes oficiais do domínio do tema, Wikimedia Commons para imagem/som), citando a fonte em `fontes`.
   - Instrução de reportar de volta em texto curto (os objetos JS + uma linha dizendo quais itens ficaram sem `sound`).

3. **Montar o DECK**: concatenar os objetos devolvidos pelos agentes (já incluindo os dados vindos do repo de referência quando aplicável). Conferir rapidamente que todo item tem os 5 atributos, `image.license` preenchida, e que nenhum `sound`/`image` tem licença não listada (NC, pixabay, etc.) sem aprovação prévia do usuário.

4. **Editar o arquivo** (nunca reescrever o arquivo inteiro; para um `deck` grande, gerar o array via um script Node no scratchpad e injetar com um patch em vez de escrever centenas de linhas manualmente por `Edit`):
   - Adicionar uma **nova chave** dentro de `const THEMES = { ... }` com `{ id, label, emoji, title, attrLabels, attrOrder, deck }` — **nunca** remover ou sobrescrever uma chave de tema já existente, a menos que o usuário peça isso explicitamente.
   - Não mexer em `<title>`/`.brand-mark` diretamente — eles são setados em runtime por `applyThemeChrome()` a partir do tema ativo.
   - Conferir que `renderThemePicker()` vai listar o novo tema automaticamente (ele itera `Object.values(THEMES)` — não deveria precisar de mudança, mas valide).

5. **Validar sintaxe**: extrair o `<script>` e rodar `node --check` (mesmo fluxo já usado neste projeto) antes de considerar pronto.

6. **Copiar para `index.html`** (espelho servido pelo GitHub Pages) e perguntar se o usuário quer commitar/publicar agora (commit + push + poll do build, seguindo o padrão já usado no projeto). Não commitar nem publicar sem essa confirmação.

## O que NUNCA tocar

Toda a lógica JS abaixo de `THEMES`/`state` (conexão Supabase, reconexão, multiplayer, sincronização de tema via `roster`/`start`/`stateSync`, sons de UI sintetizados, telas de lobby/jogo/fim), o HTML de estrutura das telas, e o `localStorage` de nome/sessão. Um novo tema é 100% uma nova entrada em `THEMES` (dados + rótulos) — zero mudança de comportamento, e zero risco pros temas que já existem.
