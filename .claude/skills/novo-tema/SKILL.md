---
name: novo-tema
description: "Cria um novo tema completo para o Duelo Selvagem (ex: dinossauros, carros, super-heróis, oceano) — troca o baralho, atributos, título e paleta, mantendo o motor do jogo intacto. Use quando o usuário pedir um novo tema, baralho ou variação do jogo."
---

# Novo Tema — Duelo Selvagem

Gera uma variação temática completa do jogo (arquivo único `duelo-selvagem.html` / `index.html`) trocando **apenas** conteúdo — nunca a lógica de rede, estado ou UI. O motor (Supabase Realtime, reconexão, multiplayer 2-4, sons de UI, telas) é agnóstico ao tema e não deve ser tocado.

## Regra inegociável (não pedir, não negociar)

Toda imagem e todo som usados em uma carta precisam ter **licença aberta verificável**: CC0, domínio público, CC BY, CC BY-SA (Wikimedia Commons, NOAA, iNaturalist, Xeno-canto, Freesound, etc.). Fontes CC BY-NC só entram com aprovação explícita do usuário no momento, carta a carta — nunca por padrão. **Nunca** usar mídia sem licença clara, mesmo alegando "uso pessoal": o site é público (GitHub Pages), então o risco de direito autoral não desaparece. Se não achar fonte licenciada para algum item, deixe `sound: null` ou troque a imagem por outra do mesmo tema — não force.

Não inventar dados numéricos (peso, velocidade, datas, etc.). Cada atributo precisa vir de uma fonte checável; anote a fonte em `info.fontes`.

## O que compõe um "tema"

1. **Título/marca**: `<title>` (linha ~6) e `.brand-mark` na tela inicial (linha ~399).
2. **Paleta de cores** (opcional, só se o usuário quiser visual diferente): variáveis em `:root` (linhas ~12-25) — `--bg`, `--surface`, `--surface-2`, `--card-bg`, `--card-border`, `--ink`, `--paper-ink`, `--accent`, `--accent-dark`, `--win`, `--muted`, `--danger`.
3. **Atributos**: `ATTR_LABELS` e `ATTR_ORDER` (linhas ~1399-1406) — exatamente 5 atributos numéricos comparáveis, cada um `{ label, unit }`. Ex. para dinossauros: peso, comprimento, ano de existência (mais antigo vence ou mais recente, escolher um critério), força de mordida, altura.
4. **O baralho `DECK`** (linhas ~553-1398 hoje) — um array de N objetos (padrão 32, mas pergunte ao usuário; **sem limite artificial**), cada um neste formato exato:

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

1. **Perguntar ao usuário** (uma vez, via AskUserQuestion se não estiver claro): tema, quantidade de itens, os 5 atributos numéricos desejados, e se quer paleta de cores nova ou manter a atual.

2. **Pesquisa em paralelo via subagentes** (nunca pesquisar tudo na conversa principal — estoura contexto à toa). Divida os itens em lotes de 6-8 e dispare um `Agent` (`general-purpose`, `run_in_background: false`, todos na mesma mensagem para rodar em paralelo) por lote. Prompt de cada agente deve incluir:
   - A lista exata de itens do lote.
   - O formato de objeto JS acima, campo a campo.
   - A regra de licença inegociável (colar o parágrafo acima).
   - Pedir que devolva **apenas** os objetos JS prontos (um por item), com valores reais checados (Wikipedia, fontes oficiais do domínio do tema, Wikimedia Commons para imagem/som), citando a fonte em `fontes`.
   - Instrução de reportar de volta em texto curto (os objetos JS + uma linha dizendo quais itens ficaram sem `sound`).

3. **Montar o DECK**: concatenar os objetos devolvidos pelos agentes. Conferir rapidamente que todo item tem os 5 atributos, `image.license` preenchida, e que nenhum `sound` tem licença NC sem aprovação prévia do usuário.

4. **Editar o arquivo** com `Edit` (nunca reescrever o arquivo inteiro):
   - `<title>` e `.brand-mark`.
   - `ATTR_LABELS` / `ATTR_ORDER`.
   - Bloco `const DECK = [ ... ];` inteiro (substituir de uma vez).
   - `:root{...}` se pediu paleta nova.

5. **Validar sintaxe**: extrair o `<script>` e rodar `node --check` (mesmo fluxo já usado neste projeto) antes de considerar pronto.

6. **Copiar para `index.html`** (espelho servido pelo GitHub Pages) e perguntar se o usuário quer commitar/publicar agora (commit + push + poll do build, seguindo o padrão já usado no projeto). Não commitar nem publicar sem essa confirmação.

## O que NUNCA tocar

Toda a lógica JS abaixo de `ATTR_ORDER`/`state` (conexão Supabase, reconexão, multiplayer, sons de UI sintetizados, telas de lobby/jogo/fim), o HTML de estrutura das telas, e o `localStorage` de nome/sessão. Um novo tema é 100% dados + rótulos + cores — zero mudança de comportamento.
