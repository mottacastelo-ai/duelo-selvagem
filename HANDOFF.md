# Duelo Selvagem — Documento de Handoff para Claude Code

## 1. O que é o projeto

Jogo de cartas de dois jogadores, no estilo Super Trunfo, para ser jogado por dois celulares na mesma rede Wi-Fi, cada um abrindo um link no navegador (sem instalar app). Tema: animais selvagens. Primeiro público-alvo: Léo Motta e seus dois filhos (11 e 8 anos).

Nota sobre propriedade intelectual: "Super Trunfo" e "Trunfo" são marcas registradas da Grow no Brasil (o jogo é uma edição do britânico Top Trumps). A mecânica de comparar atributos numéricos e o maior valor vencer **não é protegida por patente** — é um mecanismo genérico de jogo. Por isso o projeto usa nome e identidade visual próprios ("Duelo Selvagem"), evitando qualquer uso do nome ou trade dress da Grow.

## 2. Decisões de arquitetura (e por quê)

| Decisão | Alternativas descartadas | Motivo |
|---|---|---|
| Web app (link no navegador, sem instalar) | App nativo (APK) com auto-descoberta via UDP broadcast | Léo preferiu não depender de instalação, mesmo topando o APK inicialmente — a rota web venceu na comparação final |
| Relay simples via Supabase Realtime (Broadcast) + código de sala de 4 caracteres | (a) WebRTC P2P + QR Code para sinalização manual; (b) WebRTC P2P + servidor de sinalização + código curto; (c) servidor local (Node.js) rodando num PC na mesma rede | Simplicidade de implementação e confiabilidade — sem negociação de ICE/NAT, sem depender de nenhum computador extra ligado. Jogo é por turnos, então a latência extra do relay é irrelevante. Léo já tem familiaridade com Supabase (usado no Sabendo.app) |
| Um único arquivo HTML (vanilla JS, sem build step) | React/framework com bundler | Projeto pequeno, sem necessidade de build tooling; facilita hospedagem estática simples (GitHub Pages, Vercel, ou até envio direto do arquivo) |

## 3. Estado atual do código

Arquivo único: `index.html` (também presente como `duelo-selvagem.html` nos outputs do Claude.ai).

**Credenciais Supabase já configuradas no arquivo:**
- Project URL: `https://fdyjquhwbmsqgiwsyenn.supabase.co`
- Chave publicável (nova nomenclatura, equivalente à antiga `anon key`): `sb_publishable_CLo9EY8vv36kwdDaDhOihw_o4pCKPyM`

**Implementado:**
- Tela inicial: criar sala (gera código de 4 caracteres) / entrar com código.
- Conexão via `supabase.channel('game-<CODIGO>')` com Broadcast (canal público, sem `private: true`).
- Distribuição de cartas: o host embaralha e reparte; o convidado recebe sua metade via evento `deal`.
- Lógica de rodada completa: escolha de atributo → comparação → vitória / derrota / **empate com acúmulo de cartas na mesa** (pote), replicando a regra oficial do gênero de jogo (quem empatou mantém o direito de escolher na rodada seguinte; o vencedor da próxima rodada leva tudo).
- Fim de jogo quando um lado fica com todas as cartas.
- Logs de depuração no console (`[Duelo Selvagem] ...`) em pontos-chave: status do canal, recebimento de `join`, recebimento de `deal`.

**Baralho atual (baralho de teste, 4 animais — dados verificados via National Geographic, Smithsonian National Zoo, WWF e Wikipedia):**

| Animal | Peso (kg) | Velocidade máx. (km/h) | Expectativa de vida (anos, natureza) |
|---|---|---|---|
| Guepardo | 50 | 104 | 12 |
| Elefante-africano | 6000 | 40 | 65 |
| Leão | 190 | 80 | 13 |
| Girafa | 1200 | 55 | 22 |

Meta: expandir para 32 animais (padrão do gênero), sempre com dados de fontes verificáveis — nunca inventados.

## 4. Problema em aberto (prioridade máxima)

**Sintoma relatado por Léo Motta:** ao testar com dois celulares na mesma rede, o host fica preso em "Aguardando o outro jogador entrar…" e o convidado fica preso em "Conectado. Aguardando o baralho…" — ou seja, o convidado conectou e enviou o evento `join`, mas o host nunca reagiu a ele (ou a mensagem nunca chegou).

**Hipóteses a investigar, em ordem de probabilidade:**

1. **Toggle "Allow public access" desabilitado no projeto Supabase.** Canais de Broadcast públicos (sem `private: true`, que é o que este código usa) só funcionam sem RLS se essa opção estiver habilitada em Project Settings → Realtime. Projetos criados recentemente podem vir com configurações mais restritivas por padrão. Verificar em:
   `https://supabase.com/dashboard/project/fdyjquhwbmsqgiwsyenn/settings/realtime`
2. **Erro silencioso na subscrição do canal do lado do host** — o código já foi corrigido para expor `status` e `err` no callback de `channel.subscribe()`; conferir o console do navegador do host por mensagens `CHANNEL_ERROR`, `TIMED_OUT` ou `CLOSED`.
3. **Race condition ou nome de canal divergente** — conferir se `state.roomCode` é idêntico dos dois lados (o código do convidado faz `.toUpperCase()`; confirmar que o host não introduziu espaços ou caracteres invisíveis ao exibir o código).
4. **Realtime Inspector do Supabase** (`Realtime → Inspector` no painel) permite ver ao vivo se as mensagens `join`/`deal`/`choose`/`reveal` estão de fato chegando ao servidor — é o diagnóstico mais definitivo.

**Sugestão de teste mais eficiente:** antes de testar em dois celulares, testar com duas abas do Chrome no mesmo computador (uma "cria sala", outra "entra na sala"), com DevTools (F12) aberto em cada aba, observando os logs `[Duelo Selvagem] ...` no console.

## 5. Próximos passos após corrigir a conexão

1. Validar uma partida completa de ponta a ponta (incluindo um cenário de empate, para testar a lógica do "pote").
2. Expandir o baralho de 4 para 32 animais, em lotes, sempre citando a fonte de cada dado.
3. Considerar hospedagem fixa (GitHub Pages ou Vercel) para gerar um link permanente, em vez de reenviar o arquivo HTML.
4. Revisar responsividade e testes de usabilidade com os filhos de Léo Motta (11 e 8 anos) — botões grandes, texto legível, feedback visual claro de quem venceu a rodada.

---

# Prompt pronto para colar no Claude Code

```
Estou continuando um projeto chamado "Duelo Selvagem" — um jogo de cartas de dois
jogadores (estilo Top Trumps/Super Trunfo, mas com identidade própria) para dois
celulares na mesma rede Wi-Fi, via navegador (sem instalar app), usando Supabase
Realtime Broadcast como camada de comunicação entre os dois aparelhos.

Leia o arquivo HANDOFF.md nesta pasta para o contexto completo: decisões de
arquitetura, credenciais do Supabase já configuradas, estado atual do código e,
mais importante, um bug em aberto que é a prioridade agora.

O problema: ao testar com dois celulares, o host fica preso em "Aguardando o
outro jogador entrar…" e o convidado em "Conectado. Aguardando o baralho…" — o
evento 'join' enviado pelo convidado não está fazendo o host reagir.

Preciso que você:
1. Verifique, usando meu navegador já logado, se o toggle "Allow public access"
   está habilitado em Project Settings → Realtime do meu projeto Supabase
   (URL do projeto e verificação do Realtime Inspector estão no HANDOFF.md).
2. Escreva um script de teste (Node.js, usando @supabase/supabase-js) que
   simule os dois lados da conexão (host e guest) trocando os eventos
   join/deal/choose/reveal, para confirmarmos programaticamente onde a cadeia
   quebra — sem precisar de dois celulares.
3. Depois de identificar e corrigir a causa raiz, sirva o index.html localmente
   (ex: npx serve) e valide uma partida completa entre duas abas do navegador,
   incluindo pelo menos um empate, para testar a lógica de acúmulo de cartas
   na mesa.
4. Me reporte exatamente qual foi a causa do bug antes de seguir para
   qualquer outra melhoria — não quero pular pra próxima etapa sem entender
   o que quebrou.

Não invente dados nem assuma comportamento do Supabase sem verificar na
documentação oficial ou testando de fato.
```
