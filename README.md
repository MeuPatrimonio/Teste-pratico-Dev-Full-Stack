# Teste prático Desenvolvedor(a) Full Stack

## Widgets de mercado: Heatmap setorial e Notícias por ativo

> **Documento do processo seletivo**
>
> Backend obrigatório: PHP 8.3+
>
> Funcionalidade-base: dois widgets de um terminal financeiro `heatmap` e `news` do repositorio 

---

## Referência visual

As imagens abaixo são a **implementação de referência**. O candidato deve reproduzir o comportamento e o contrato de dados, não o estilo visual pixel a pixel. Fonte, cor exata e espaçamento são livres.

![Heatmap do mercado brasileiro ao lado do widget de notícias no modo por ativo, exibindo manchetes de PETR3.SA](docs/screenshots/heatmap-br-news-symbol.png)

*Figura 1 Heatmap com o mercado `BR` selecionado e notícias no modo por ativo. O título do widget acompanha o símbolo ativo (`NEWS PETR3.SA`), que foi definido ao clicar em Petrobras no heatmap.*

![Mesmo heatmap com o widget de notícias no modo GLOBAL, exibindo manchetes de mercado de diversos veículos](docs/screenshots/heatmap-br-news-global.png)

*Figura 2 Mesmo heatmap, notícias no modo `GLOBAL`. O símbolo ativo agora é `ABEV3.SA`, mas o modo global ignora o símbolo e devolve manchetes de mercado. Veículos diferentes (WSJ, Bloomberg, Investopedia, The Motley Fool) comprovam a mescla de fontes.*

### O que as imagens comprovam

| Elemento visível | Requisito correspondente |
|---|---|
| Botões `US` `EU` `BR` no heatmap | Seleção de mercado seção 2.2 |
| Retângulos agrupados sob rótulos de setor (`FINANCE`, `ENERGY MINERALS`, …) | Agrupamento por setor 2.2.1 |
| `PETR3.SA` ocupando a maior área de Energy Minerals | Área proporcional ao valor de mercado 2.2.2 |
| Verde em `+2,64%`, vermelho em `-3,11%`, tons saturando perto de ±3% | Escala divergente limitada 2.2.3 |
| Sufixo `.SA` em todos os tickers | Normalização de símbolo B3 2.4 |
| Rótulo `TECHNOLOGY SERVI…` cortado na própria coluna | Recorte do rótulo por coluna 2.2.6 |
| Retângulos pequenos sem texto algum | Texto condicional ao espaço 2.2.7 |
| Título do widget mudando de `PETR3.SA` para `ABEV3.SA` entre as figuras | Clique no heatmap propaga o símbolo ativo 2.2.8 |
| Botões `PETR3.SA` `GLOBAL` `BR` nas notícias | Três modos 2.3 |
| Veículo e data/hora sob cada manchete | Metadados do item 2.3.6 |

---

## Endpoints referência rápida

> **Leia esta seção antes de começar.** Ela resume o que chamar e o que expor. As regras completas de validação, cache, degradação e segurança estão nas seções 2.4, 5.1 e 5.2 e prevalecem sobre este resumo.

### A. Endpoints externos que o seu backend PHP deve consumir

Nenhum exige chave de API. Todos são instáveis.

| # | Provedor | Requisição | Alimenta |
|---|---|---|---|
| 1 | TradingView Scanner | `POST https://scanner.tradingview.com/america/scan` | Heatmap `US` |
| 2 | TradingView Scanner | `POST https://scanner.tradingview.com/brazil/scan` | Heatmap `BR` |
| 3 | Yahoo Finance RSS | `GET https://feeds.finance.yahoo.com/rss/2.0/headline?s={symbol}&region=US&lang=en-US` | Notícias por ativo |
| 4 | Google News RSS | `GET https://news.google.com/rss/search?q={query}&hl=en-US&gl=US&ceid=US:en` | Notícias nos três modos |

Os endpoints 1 e 2 exigem `Referer` e `Origin` de `https://www.tradingview.com`; sem eles a borda responde `403`.

**Comandos para conferir cada provedor antes de escrever código:**

```bash
# 1/2 Heatmap BR (troque brazil por america para o mercado US)
curl -sS -X POST 'https://scanner.tradingview.com/brazil/scan' \
  -H 'Content-Type: application/json' \
  -H 'Referer: https://www.tradingview.com/' \
  -H 'Origin: https://www.tradingview.com' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/126.0 Safari/537.36' \
  -d '{
        "columns": ["description","close","change","market_cap_basic","sector","volume","exchange"],
        "filter": [
          {"left":"type","operation":"equal","right":"stock"},
          {"left":"typespecs","operation":"has","right":["common"]},
          {"left":"country","operation":"equal","right":"Brazil"},
          {"left":"exchange","operation":"equal","right":"BMFBOVESPA"}
        ],
        "sort": {"sortBy":"market_cap_basic","sortOrder":"desc"},
        "range": [0, 5]
      }'

# 3 Notícias por ativo (Yahoo)
curl -sS -H 'User-Agent: Mozilla/5.0' \
  'https://feeds.finance.yahoo.com/rss/2.0/headline?s=PETR3.SA&region=US&lang=en-US'

# 4 Notícias por ativo (Google News)
curl -sS 'https://news.google.com/rss/search?q=PETR3.SA+stock&hl=en-US&gl=US&ceid=US:en'
```

> **Verificação feita na redação deste documento:** o scanner respondeu `200`, o Google News respondeu `200` e o **Yahoo respondeu `429 Too Many Requests`**. Isso não é defeito do enunciado: é exatamente o cenário que a seção 5.2 manda tratar. Se o Yahoo estiver bloqueado na sua rede, o modo por ativo ainda deve funcionar com a fonte restante.

### B. Endpoints internos que você deve expor

| Método e rota | Responsabilidade | Acesso |
|---|---|---|
| `GET /api/health` | Prontidão, sem depender de provedor externo | Público |
| `GET /api/v1/heatmap?market=us\|br` | Células do heatmap | **Autenticado**, com limite de taxa |
| `GET /api/v1/news?symbol={ticker}` | Notícias do ativo | **Autenticado**, com limite de taxa |
| `GET /api/v1/news` | Notícias globais | **Autenticado**, com limite de taxa |
| `GET /api/v1/news?region=br` | Notícias do mercado brasileiro | **Autenticado**, com limite de taxa |
| `POST /api/v1/auth/login` | Criar sessão | Público, com limite de taxa |
| `POST /api/v1/auth/logout` | Encerrar sessão | Autenticado |
| `GET /api/v1/auth/me` | Usuário atual | Autenticado |
| `GET /api/v1/admin/providers` | Saúde agregada dos provedores | Administrador |
| `GET /api/v1/admin/provider-calls` | Auditoria paginada e filtrada | Administrador |

`GET /api/health` é o **único** endpoint público além do login. Os dois widgets só respondem a quem tem sessão ver seção 2.1.1.

### C. Mapa completo: botão da interface → sua rota → provedor

```text
BOTÃO NA TELA         SUA ROTA PHP                         PROVEDOR EXTERNO
──────────────────────────────────────────────────────────────────────────────────────
HEATMAP  [US]    →    GET /api/v1/heatmap?market=us   →    scanner.tradingview.com/america/scan
HEATMAP  [BR]    →    GET /api/v1/heatmap?market=br   →    scanner.tradingview.com/brazil/scan
HEATMAP  [EU]    →    (bônus, seção 9)                →    um scanner por país europeu

NEWS  [PETR3.SA] →    GET /api/v1/news?symbol=PETR3.SA →   feeds.finance.yahoo.com  +
                                                           news.google.com  "PETR3.SA stock"
NEWS  [GLOBAL]   →    GET /api/v1/news                →    news.google.com  "stock market"  +
                                                           news.google.com  "federal reserve economy"
NEWS  [BR]       →    GET /api/v1/news?region=br      →    news.google.com  "Ibovespa mercado financeiro"  +
                                                           news.google.com  "B3 bolsa de valores Brasil"
──────────────────────────────────────────────────────────────────────────────────────
                      ▲                                     ▲
                      │ única origem que o                  │ única camada autorizada
                      │ navegador conhece                   │ a sair para a internet
```

O botão `EU` aparece nas capturas porque existe na implementação de referência, mas **não é obrigatório** nesta entrega: o mercado europeu exige uma requisição por país e conversão cambial, o que não cabe no timebox. Implementá-lo vale bônus.

---

## 1. Objetivo do teste

Este teste avalia a capacidade de construir um agregador de dados de mercado completo e reproduzível, em que **o backend PHP é o único ponto de saída para a internet** e o navegador nunca fala com provedores externos. O fluxo obrigatório é:

```text
visitante autentica → sessão criada no servidor
    → backend consulta provedores públicos → normaliza e cacheia
    → serve os dois widgets apenas a quem tem sessão válida
    → registra cada chamada externa
    → administrador consulta saúde e auditoria
```

Serão avaliados:

- PHP moderno e API HTTP;
- consumo resiliente de APIs de terceiros instáveis;
- parsing seguro de XML/RSS de origem não confiável;
- normalização e saneamento de dados vindos de fora;
- autenticação, autorização e segurança de sessão;
- modelagem e consultas em PostgreSQL ou MySQL;
- frontend tipado com visualização de dados;
- DDD pragmático e TDD;
- cache, degradação graciosa e rastreabilidade;
- Docker Compose e reprodutibilidade;
- CI e documentação técnica.

O objetivo **não** é premiar volume de código. Dois widgets pequenos, seguros, testados offline e fáceis de executar valem mais que uma arquitetura excessiva.

---

## 2. Especificação funcional

### 2.1 Atores

| Ator | Permissões |
|---|---|
| Visitante | **Apenas autenticar-se.** Não enxerga nenhum dado de mercado |
| Usuário autenticado | Visualizar os dois widgets e encerrar a sessão |
| Administrador | Possui as permissões do usuário e pode consultar a saúde dos provedores e a auditoria das chamadas externas |

O ambiente deve possuir, criados por seed, **um usuário comum e um administrador** de demonstração. Sem os dois não é possível avaliar a diferença entre `401`, `403` e acesso concedido. As senhas de desenvolvimento podem estar documentadas no README, desde que sejam claramente fictícias e válidas somente para o ambiente local.

### 2.1.1 Acesso restrito aos widgets

Os dois widgets **exigem sessão autenticada**. Um visitante sem sessão:

1. ao acessar a página da aplicação, é redirecionado para a tela de login;
2. ao chamar `GET /api/v1/heatmap` ou `GET /api/v1/news` diretamente, recebe `401` em JSON;
3. não recebe nenhum dado de mercado em corpo, cabeçalho ou mensagem de erro.

A proteção deve estar no backend. Esconder o widget no frontend mantendo o endpoint aberto será tratado como ausência de autorização.

> **Armadilha de projeto leia antes de implementar o cache.**
>
> Autenticação controla **acesso**, não **particionamento de dados**. Os dados de mercado são idênticos para todos os usuários logados: o heatmap do `BR` é o mesmo para o usuário comum e para o administrador.
>
> Portanto o cache de provedor deve continuar **compartilhado entre todos os usuários** e chaveado por operação `heatmap:br`, `news:PETR3.SA`, `news:global` e **nunca** por `user_id` ou por identificador de sessão.
>
> Chavear o cache por usuário multiplica as chamadas externas pelo número de pessoas logadas, anula a proteção da seção 5.2 e é o caminho mais curto para o deployment ser bloqueado pelos provedores. Esse erro será penalizado nos pontos de resiliência mesmo que a aplicação pareça funcionar.

### 2.2 Widget 1 Heatmap setorial

Mapa de árvore (treemap) das maiores empresas listadas, agrupadas por setor.

Requisitos:

1. agrupar os ativos por setor;
2. dimensionar a área de cada retângulo pelo valor de mercado;
3. colorir pela variação percentual do dia, em escala divergente limitada a ±3%;
4. exibir no máximo os 150 maiores por valor de mercado, descartando registros sem valor de mercado;
5. descartar listagens de balcão (campo `exchange` igual a `OTC`);
6. exibir o rótulo do setor recortado à sua própria coluna, sem transbordar para a vizinha;
7. exibir ticker e variação apenas quando o retângulo comportar o texto;
8. permitir clique no ativo para selecioná-lo como símbolo ativo da aplicação;
9. redesenhar ao redimensionar o contêiner.
10. permitir alternar entre os mercados `US` e `BR`, mantendo cache independente por mercado.

O símbolo ativo selecionado no item 8 é o mesmo consumido pelo widget de notícias. Clicar em `PETR3.SA` no heatmap deve fazer o widget vizinho passar a exibir notícias de `PETR3.SA`, como mostram as Figuras 1 e 2.

### 2.3 Widget 2 Notícias por ativo

Lista de manchetes com três modos, conforme os botões das Figuras 1 e 2:

| Modo | Rótulo do botão | Conteúdo |
|---|---|---|
| Por ativo | o próprio ticker ativo, por exemplo `PETR3.SA` | Manchetes do ativo selecionado |
| Global | `GLOBAL` | Manchetes de mercado, ignorando o símbolo ativo |
| Brasil | `BR` | Manchetes do mercado brasileiro, em português |

Requisitos:

1. no modo por ativo, mesclar duas fontes independentes e usar o símbolo ativo da aplicação como padrão;
2. nos modos global e Brasil, mesclar duas consultas temáticas independentes e ignorar o símbolo ativo;
3. deduplicar por título normalizado minúsculas, não alfanuméricos colapsados em espaço, 80 primeiros caracteres;
4. ordenar do mais recente para o mais antigo;
5. limitar a 40 itens;
6. exibir título, veículo e data/hora local;
7. abrir o link em nova aba com `rel="noreferrer"`;
8. renderizar como texto inerte, **sem âncora**, qualquer item cujo link não seja `http` ou `https`;
9. tolerar falha parcial: se uma das duas fontes falhar, a outra ainda deve ser exibida.

### 2.4 Provedores externos

O candidato deve consumir estes provedores públicos e sem chave:

| Provedor | Endpoint | Uso |
|---|---|---|
| TradingView Scanner | `POST https://scanner.tradingview.com/america/scan` | Heatmap `US` |
| TradingView Scanner | `POST https://scanner.tradingview.com/brazil/scan` | Heatmap `BR` |
| Yahoo Finance RSS | `GET https://feeds.finance.yahoo.com/rss/2.0/headline?s={symbol}&region=US&lang=en-US` | Notícias por ativo |
| Google News RSS | `GET https://news.google.com/rss/search?q={query}&hl=en-US&gl=US&ceid=US:en` | Notícias nos três modos |

Corpo de referência da requisição do heatmap `US`:

```json
{
  "columns": ["description", "close", "change", "market_cap_basic", "sector", "volume", "exchange"],
  "filter": [
    { "left": "type", "operation": "equal", "right": "stock" },
    { "left": "typespecs", "operation": "has", "right": ["common"] }
  ],
  "sort": { "sortBy": "market_cap_basic", "sortOrder": "desc" },
  "range": [0, 1500]
}
```

Para o heatmap `BR`, acrescentar dois filtros ao mesmo corpo:

```json
{ "left": "country",  "operation": "equal", "right": "Brazil" },
{ "left": "exchange", "operation": "equal", "right": "BMFBOVESPA" }
```

O endpoint exige `Referer` e `Origin` plausíveis; sem eles a borda responde `403`. A resposta traz `data[].s` no formato `BOLSA:TICKER` e `data[].d` como vetor posicional na ordem das colunas pedidas.

#### Normalização obrigatória do mercado brasileiro

O scanner devolve o ticker puro (`PETR3`), mas a aplicação trabalha com a convenção do Yahoo (`PETR3.SA`), que é a usada no widget de notícias e visível nas Figuras 1 e 2. Portanto:

1. acrescentar o sufixo `.SA` a todo ticker vindo do scanner brasileiro;
2. descartar o mercado fracionário da B3 tickers terminados em dígito seguido de `F`, por exemplo `PETR3F`, que o scanner devolve **antes** do ticker principal por aparecerem com o mesmo valor de mercado;
3. tratar a moeda como `BRL` e não converter para dólar.

Sem o item 2, a maior célula do setor Energy Minerals aparece duplicada. Sem o item 1, clicar na célula do heatmap faz o widget de notícias consultar um símbolo que o Yahoo não resolve.

#### Consultas de notícias por modo

| Modo | Fonte 1 | Fonte 2 |
|---|---|---|
| Por ativo | Yahoo RSS com `s={symbol}` | Google News com `q={symbol} stock` |
| Global | Google News com `q=stock market` | Google News com `q=federal reserve economy` |
| Brasil | Google News com `q=Ibovespa mercado financeiro` | Google News com `q=B3 bolsa de valores Brasil` |

As consultas temáticas podem ser ajustadas, desde que cada modo use duas fontes independentes e o README documente as escolhas.

> **Atenção:** estes provedores são instáveis por natureza. Durante a avaliação é esperado encontrar `429`, `403` e *timeout* o Yahoo respondeu `429` durante a redação deste enunciado. Lidar com isso **é o teste**, não um imprevisto.

### 2.5 Fronteira de rede regra arquitetural

```text
Navegador
    │  fala exclusivamente com a própria origem
    ▼
Nginx  ──►  PHP  ◄── único processo autorizado a sair para a internet
                │
                ├── TradingView Scanner (america e brazil)
                ├── Yahoo Finance RSS
                └── Google News RSS
```

O navegador **não pode** emitir nenhuma requisição para domínio de terceiros. Essa restrição é uma regra arquitetural do teste, não uma recomendação, e deve ser sustentada por:

- `Content-Security-Policy` com `connect-src 'self'`;
- ausência de scripts, fontes ou folhas de estilo carregados de CDN no perfil de avaliação;
- ausência de qualquer requisição a domínio externo na aba Rede do navegador.

Motivo: um terminal que consulta provedores direto do navegador expõe o padrão de uso do cliente a terceiros, perde o cache compartilhado, não consegue aplicar limite de taxa e quebra assim que um provedor remove o CORS.

### 2.6 Requisitos de auditoria

Cada tentativa de chamada a provedor externo deve gerar um registro durável:

| Evento | Momento |
|---|---|
| `provider.succeeded` | Provedor respondeu e o resultado foi aceito |
| `provider.failed` | Provedor recusou, excedeu o tempo ou devolveu corpo inutilizável |
| `provider.skipped` | Chamada não foi feita por estar em janela de resfriamento |

Cada registro deve permitir responder:

- qual provedor e qual operação;
- quando ocorreu, em UTC definido pelo servidor;
- qual requisição HTTP originou a tentativa;
- se terminou com sucesso, falha ou bloqueio;
- quanto tempo levou;
- quantos itens foram aproveitados, quando aplicável.

O registro **não pode** conter o corpo da resposta do provedor, títulos de notícia, URLs de manchete, cotações, IP do visitante, `User-Agent` do visitante, cookies, senha ou token.

Esses registros são **telemetria de infraestrutura**, não arquivo de dados de mercado. Quem quiser os dados consulta os widgets; a auditoria responde apenas “o provedor respondeu?”.

### 2.7 Regras que não devem ser alteradas

Não fazem parte do desafio:

- inventar métricas financeiras que o provedor não devolve;
- recalcular valor de mercado, variação ou setor;
- traduzir ou reescrever manchetes;
- reordenar o heatmap por critério diferente de valor de mercado;
- alterar a escala de cor para além do limite de ±3%.

Qualquer comportamento estranho nessas áreas deve ser documentado, não “corrigido” sem especificação.

---

## 3. Desafio

Completar o fluxo vertical da aplicação:

1. autenticar o usuário e manter a sessão no servidor;
2. restringir os dois widgets a usuários autenticados, no backend;
3. implementar o backend PHP que consulta os três provedores;
4. normalizar, cachear e degradar graciosamente quando um provedor falhar;
5. servir os dois widgets a partir da própria origem;
6. registrar cada tentativa de chamada externa sem vazar conteúdo;
7. autorizar separadamente o acesso administrativo;
8. disponibilizar ao administrador a saúde dos provedores e a listagem paginada da auditoria;
9. entregar todo o ambiente em Docker Compose.

---

## 4.Formato da entrega

Fluxo mínimo de avaliação:

```bash
git clone <URL_DO_REPOSITORIO>
cd <REPOSITORIO>
docker compose build
docker compose up -d
```

Depois desses comandos, a aplicação deve estar acessível na URL documentada no README. Não haverá deploy em nuvem nem configuração manual do ambiente pelo avaliador.

### 4.1 Material de avaliação que a organização deve fornecer

Antes da abertura da vaga, a organização deve preparar e versionar um manifesto de *fixtures* com:

1. uma resposta real e íntegra do scanner `america/scan`;
2. uma resposta real e íntegra do scanner `brazil/scan`, contendo ao menos um par principal/fracionário, por exemplo `PETR3` e `PETR3F`;
3. um RSS real do Yahoo Finance para `PETR3.SA`;
4. um RSS real do Google News para o modo global e outro para o modo `BR`;
5. uma resposta `429` com corpo real;
6. um XML sintaticamente malformado;
7. um XML contendo `DOCTYPE` e `ENTITY` externa apontando para arquivo local;
8. um item de feed cujo link use o esquema `javascript:`;
9. hash SHA-256, finalidade e resultado observável esperado de cada arquivo.

Sem esse material o desafio não deve ser iniciado: a suíte de testes precisa rodar offline e de forma determinística, e o candidato não pode depender de o provedor estar instável no dia certo.

---

# 5. Entregas obrigatórias

## 5.1 API PHP

A API é responsável por egressão de rede, normalização, cache, identidade, sessão, autorização e auditoria.

PHP 8.3 ou superior. O framework é livre Laravel, Symfony, Slim ou PSR-15 sem framework. A escolha deve ser justificada no README. O que será avaliado é a separação de camadas, não o framework.

Contrato mínimo sugerido:

| Método e rota | Responsabilidade | Acesso |
|---|---|---|
| `GET /api/health` | Prontidão do serviço | Público |
| `GET /api/v1/heatmap` | Células do heatmap | Autenticado, com limite de taxa |
| `GET /api/v1/news` | Manchetes por ativo, globais ou do Brasil | Autenticado, com limite de taxa |
| `POST /api/v1/auth/login` | Criar sessão | Público, com limite de taxa |
| `POST /api/v1/auth/logout` | Encerrar sessão | Autenticado |
| `GET /api/v1/auth/me` | Retornar usuário atual | Autenticado |
| `GET /api/v1/admin/providers` | Saúde agregada dos provedores | Administrador |
| `GET /api/v1/admin/provider-calls` | Auditoria paginada e filtrada | Administrador |

Não é exigido cadastro público de usuários: o usuário comum e o administrador vêm por seed. Se o candidato implementar cadastro, ele não pode aceitar `role` vindo do cliente.

Comportamento HTTP obrigatório:

- sessão válida e dados disponíveis, de qualquer procedência: `200`;
- widget chamado sem sessão: `401` em JSON, sem tocar a rede e sem revelar dado de mercado;
- página da aplicação acessada sem sessão: redirecionamento para a tela de login;
- parâmetro inválido: `422` em JSON;
- limite de taxa excedido: `429` com `Retry-After`;
- chamada administrativa sem sessão: `401` em JSON;
- chamada administrativa por usuário comum: `403` em JSON;
- mutação sem CSRF válido: `419` ou equivalente documentado;
- nenhuma fonte disponível e nenhum cache utilizável: `503` em JSON, com erro específico da capacidade afetada.

Ordem de avaliação obrigatória: **sessão primeiro, validação depois**. Um visitante anônimo que envie `?symbol=<script>` deve receber `401`, não `422`: responder `422` confirma a existência e o formato do parâmetro a quem ainda não provou identidade.

O limite de taxa dos widgets deve ser contado por usuário autenticado, não apenas por IP. Vários usuários atrás do mesmo NAT corporativo não devem competir pela mesma cota.

### Contrato de resposta dos widgets

Toda resposta de widget deve declarar a procedência do dado:

```json
{
  "data": [],
  "meta": {
    "freshness": "live",
    "generated_at": "2026-11-09T14:12:03Z",
    "providers": ["tradingview"],
    "degraded": false
  }
}
```

`freshness` aceita exatamente `live`, `cached` e `stale`. O mesmo valor deve ser espelhado no cabeçalho `X-Data-Freshness`.

### Contrato de erro

Mensagem genérica do tipo “todos os provedores estão indisponíveis” será considerada defeito de observabilidade. O erro deve nomear a capacidade:

```json
{
  "error": {
    "code": "heatmap_unavailable",
    "message": "O heatmap está temporariamente indisponível. As notícias continuam disponíveis.",
    "retry_after_seconds": 30
  }
}
```

Motivo: uma mensagem global faz o operador investigar a infraestrutura inteira quando apenas um provedor caiu, e faz o usuário acreditar que a aplicação inteira morreu quando metade dela está funcionando.

### Parâmetros aceitos

| Endpoint | Parâmetro | Regra |
|---|---|---|
| `/api/v1/heatmap` | `market` | Opcional, allowlist `us` e `br`, padrão `us` |
| `/api/v1/heatmap` | `limit` | Opcional, inteiro de `1` a `150`, padrão `150` |
| `/api/v1/news` | `symbol` | Opcional; quando ausente e sem `region`, modo global |
| `/api/v1/news` | `region` | Opcional, allowlist `br`; mutuamente exclusivo com `symbol` |
| `/api/v1/news` | `limit` | Opcional, inteiro de `1` a `40`, padrão `40` |

O símbolo deve ser validado por allowlist estrita letras maiúsculas, dígitos, ponto, hífen e circunflexo, até 12 caracteres **antes** de ser interpolado em qualquer URL. A expressão de referência é `/^[A-Z0-9.^-]{1,12}$/`, que aceita `PETR3.SA`, `ABEV3.SA` e `AAPL` e rejeita qualquer outra coisa. Símbolo fora do padrão retorna `422` sem tocar a rede.

Motivo: `symbol` entra na URL do provedor. Sem allowlist, o parâmetro vira vetor de SSRF e de injeção de parâmetros na consulta do terceiro.

Enviar `symbol` e `region` na mesma requisição retorna `422`: são modos mutuamente exclusivos, e aceitar os dois esconderia qual venceu.

## 5.2 Resiliência, cache e degradação

Contrato obrigatório:

| Mecanismo | Exigência |
|---|---|
| Timeout | Explícito por provedor, nunca infinito |
| Cache positivo | TTL por operação, documentado |
| Cache de emergência | Último valor bom-sucedido preservado além do TTL |
| Resfriamento | Após `429` ou `403`, janela em que o provedor não é chamado |
| Falha parcial | Uma fonte de notícias caída não derruba a outra |
| Limite de memória | Número de chaves em cache limitado, com descarte do menos usado |

Ordem de resolução obrigatória:

```text
cache válido ──► responde live/cached
     │ ausente
     ▼
provedor em resfriamento? ──sim──► registra provider.skipped ──┐
     │ não                                                     │
     ▼                                                         │
chama provedor                                                 │
     ├── sucesso ──► atualiza cache + emergência ──► live      │
     └── falha ────► registra provider.failed ─────────────────┤
                                                               ▼
                                              cache de emergência existe?
                                                   ├── sim ──► stale
                                                   └── não ──► 503 específico
```

Responder `503` quando existe cache de emergência utilizável será tratado como defeito.

## 5.3 Persistência SQL

Usar **PostgreSQL ou MySQL**. O candidato deve documentar a escolha e eventuais diferenças de portabilidade.

### Tabela `users`

Deve possuir, no mínimo: identificador; nome; e-mail normalizado e único; hash da senha; papel suficiente para distinguir usuário e administrador; timestamps.

### Tabela `provider_calls`

Deve possuir, no mínimo:

- identificador interno;
- `call_id` público e único, para idempotência de regravação;
- provedor, restrito por allowlist;
- operação, restrita por allowlist;
- resultado: `succeeded`, `failed` ou `skipped`;
- status HTTP devolvido pelo provedor, quando houve resposta;
- duração em milissegundos;
- `request_id` para correlação;
- código de falha categorizado, quando aplicável: `timeout`, `rate_limited`, `forbidden`, `bad_payload` ou `unknown`;
- metadados técnicos em JSON, limitados por allowlist;
- data/hora de ocorrência definida pelo servidor, em UTC;
- timestamps.

Registros são imutáveis depois da criação. Campos textuais devem ter limites explícitos. A exclusão de usuário não deve apagar auditoria silenciosamente.

Índices mínimos:

- unicidade de `users.email`;
- unicidade de `provider_calls.call_id`;
- índice em `provider_calls.provider`;
- índice em `provider_calls.outcome`;
- índice em `provider_calls.occurred_at`.

Se a estratégia para alto volume fizer parte da solução, índices compostos devem acompanhar os filtros e a ordenação, por exemplo `(provider, occurred_at, id)`. Índices adicionais precisam ser justificados por consulta.

Migrations e rollback devem funcionar. A listagem administrativa deve paginar no banco, não carregar tudo em memória.

### Consulta administrativa

```http
GET /api/v1/admin/provider-calls?provider=yahoo_rss&outcome=failed&from=2026-11-01T00:00:00Z&to=2026-11-30T23:59:59Z&page=1&per_page=25
```

- filtros opcionais: `provider`, `operation`, `outcome`, `failure_code`, `from` e `to` em UTC/ISO-8601;
- `page` inicia em 1;
- `per_page` padrão 25 e máximo 100;
- ordenação estável por `occurred_at DESC, id DESC`;
- resposta com `data` e metadados `current_page`, `per_page`, `total` e `last_page`.

## 5.4 Autenticação e autorização

Requisitos obrigatórios:

- senha armazenada com hash seguro nativo do PHP ou do framework;
- renovação do identificador da sessão após login;
- invalidação da sessão e renovação do token CSRF no logout;
- cookie de sessão `HttpOnly`, com `SameSite=Lax` ou mais restritivo e `Secure` quando houver HTTPS;
- proteção CSRF nas mutações autenticadas;
- limite de taxa no login;
- **endpoints dos widgets protegidos no backend**, não apenas escondidos no frontend;
- endpoints administrativos protegidos no backend por Policy, Gate ou middleware equivalente;
- expiração de sessão tratada no frontend com redirecionamento, sem tela quebrada;
- mensagens de login que não facilitem enumeração de contas.

Token de autenticação persistido em `localStorage` será considerado decisão insegura. Não é necessário implementar recuperação de senha, MFA ou login social.

## 5.5 Frontend

O frontend deve implementar:

- tela de login como porta de entrada: sem sessão, nenhum widget é montado nem chamado;
- os dois widgets conforme as seções 2.2 e 2.3;
- alternância de modo no widget de notícias;
- indicação visível quando o dado servido for `stale`;
- estados de carregamento, vazio, erro e sucesso em ambos os widgets;
- estado de sessão restaurado ao recarregar;
- redirecionamento para login quando a sessão expirar;
- página administrativa com saúde dos provedores e filtros de auditoria por provedor, resultado e período;
- contratos tipados para as respostas da API;
- rótulos associados aos campos, foco visível, mensagens de erro anunciadas e navegação por teclado.

A referência é React com TypeScript. Vue com TypeScript é aceito. Renderização server-side em PHP com TypeScript apenas no treemap também é aceita, desde que os contratos sejam tipados. Não implemente a mesma interface duas vezes.

Para o treemap é permitido usar biblioteca de layout. O algoritmo de empacotamento não é o objeto da avaliação; o recorte de rótulo, a escala de cor e o comportamento no redimensionamento são.

## 5.6 DDD pragmático

A separação esperada é:

```text
Domínio
  ├── provedores e operações permitidos
  ├── política de metadados seguros da auditoria
  ├── regra de frescor: live, cached, stale
  └── invariantes da célula de heatmap e do item de notícia

Aplicação
  ├── GetSectorHeatmap
  ├── GetSymbolNews
  ├── RecordProviderCall
  └── ListProviderCalls

Infraestrutura
  ├── clientes HTTP dos três provedores
  ├── parser de RSS
  ├── cache
  ├── banco SQL
  ├── sessão
  └── logger estruturado

Apresentação
  ├── controllers e requests HTTP
  └── componentes dos widgets
```

Essa árvore é referência, não exigência de nomes ou quantidade de classes. Estruturas equivalentes serão aceitas quando preservarem responsabilidades e invariantes.

Espera-se que allowlists, saneamento, política de frescor e autorização não estejam espalhados em controllers e componentes. O domínio não deve conhecer `cURL`, `SimpleXML` nem Eloquent.

Não são exigidos event sourcing, command bus, microsserviços, repository genérico ou aggregates artificiais.

## 5.7 Segurança

Esta seção tem peso elevado porque os dois widgets consomem **conteúdo não confiável de terceiros**.

### 5.7.1 Parsing de XML

O RSS vem de fora e não é confiável. Obrigatório:

- carregar o XML sem resolver entidades externas e sem acesso a rede pelo parser;
- não utilizar `LIBXML_NOENT`;
- rejeitar ou ignorar documentos que declarem `DOCTYPE`;
- limitar o tamanho do corpo aceito antes do parsing;
- limitar a profundidade e a quantidade de nós processados;
- comprovar por teste que um feed com `ENTITY` apontando para arquivo local não lê o arquivo e não faz requisição de rede.

Motivo: entidade externa em XML é leitura arbitrária de arquivo e SSRF no processo PHP. É a falha mais provável desta aplicação.

### 5.7.2 Links de terceiros

- aceitar somente os esquemas `http` e `https`;
- rejeitar `javascript:`, `data:`, `vbscript:` e esquemas relativos ambíguos;
- renderizar o item como texto inerte quando o link for rejeitado, sem descartá-lo da lista;
- usar `rel="noreferrer"` em toda âncora externa;
- escapar título e veículo na renderização.

### 5.7.3 Egressão e cabeçalhos

- `Content-Security-Policy` com `connect-src 'self'` e sem `unsafe-inline` em `script-src`;
- `X-Content-Type-Options: nosniff`;
- `Referrer-Policy` restritiva;
- nenhum recurso de CDN no perfil de avaliação.

### 5.7.4 Geral

- segredos fora do Git e das imagens;
- sem analytics ou rastreadores de erro externos no perfil de avaliação;
- allowlist para provedor, operação e chaves de metadados;
- não aceitar papel, usuário ou horário como autoridade do cliente;
- não registrar senha, token, cookie, corpo integral da resposta do provedor nem IP do visitante;
- não desabilitar CSRF, CORS ou validação globalmente;
- não retornar stack trace ao navegador;
- revisar todo o histórico antes de compartilhar o repositório.

## 5.8 TDD e testes automatizados

A suíte deve rodar **offline**, usando as fixtures da seção 4.1. Teste que depende de provedor real será desconsiderado.

### Backend

Implementar, no mínimo:

**Acesso e sessão**

1. visitante sem sessão recebe `401` em `/api/v1/heatmap` e em `/api/v1/news`, sem tocar a rede;
2. visitante anônimo com parâmetro inválido recebe `401`, não `422`;
3. usuário autenticado recebe `200` nos dois widgets;
4. login válido renova o identificador da sessão;
5. após `logout`, o mesmo cookie deixa de dar acesso aos widgets;
6. usuário comum impedido de acessar a listagem administrativa;
7. administrador consultando auditoria com paginação e filtros;
8. limite de taxa devolve `429` com `Retry-After`.

**Heatmap**

9. normaliza a resposta do scanner e mapeia corretamente as colunas posicionais;
10. descarta registros sem valor de mercado e listagens `OTC`;
11. respeita o limite de 150 e a ordenação por valor de mercado;
12. mercado `BR` acrescenta `.SA` e descarta o mercado fracionário (`PETR3F`).

**Notícias**

13. mesclam duas fontes e deduplicam por título normalizado;
14. ordenam do mais recente para o mais antigo;
15. sobrevivem à falha de uma das duas fontes;
16. símbolo fora da allowlist retorna `422` sem tocar a rede;
17. `symbol` e `region` enviados juntos retornam `422`.

**Segurança do conteúdo de terceiros**

18. XML com `DOCTYPE`/`ENTITY` externa não lê arquivo local nem acessa a rede;
19. XML malformado não derruba a requisição;
20. item com link `javascript:` é entregue marcado como inseguro.

**Cache e degradação**

21. cache válido é servido sem nova chamada ao provedor;
22. o cache é compartilhado entre usuários: dois usuários autenticados diferentes pedindo o mesmo mercado geram **uma** chamada externa;
23. falha do provedor com cache de emergência responde `200` com `freshness: stale`;
24. falha do provedor sem cache responde `503` com código específico da capacidade;
25. `429` do provedor abre janela de resfriamento e a chamada seguinte registra `provider.skipped`.

**Auditoria**

26. toda tentativa gera registro de auditoria com duração e `request_id`;
27. auditoria não persiste título, URL de manchete nem corpo da resposta.

O caso 22 é o que separa uma implementação correta de uma que apenas parece funcionar: ele falha quando o candidato chaveia o cache por usuário ou por sessão.

### Frontend

Cobrir estes dois fluxos obrigatórios:

1. login bem-sucedido → heatmap renderiza células agrupadas por setor → clique propaga o símbolo para o widget de notícias;
2. notícias renderizam âncora para link seguro e texto inerte para link inseguro.

Sessão expirada redirecionando para o login, estado `stale` sinalizado na interface e filtro administrativo são casos adicionais recomendados.

### Evidência de TDD

O histórico progressivo e a entrevista devem permitir discutir pelo menos um ciclo real:

1. **RED:** teste relevante falhando;
2. **GREEN:** implementação mínima passando;
3. **REFACTOR:** melhoria mantendo a suíte verde.

Não crie commits artificiais apenas para simular TDD. O valor de regressão, isolamento e legibilidade dos testes terá mais peso que o nome ou a quantidade de commits.

## 5.9 Logs operacionais

O candidato deve distinguir:

1. **Auditoria funcional:** linha durável em `provider_calls`, consultável pelo administrador.
2. **Log operacional:** JSON em `stdout`/`stderr` dos containers, para diagnóstico.

Exemplo de log operacional:

```json
{
  "event": "provider_call.completed",
  "request_id": "req-7b61e8e2dce54b29",
  "call_id": "019a5c1e-0000-7000-8000-000000000001",
  "provider": "tradingview",
  "operation": "heatmap.scan",
  "outcome": "failed",
  "failure_code": "rate_limited",
  "provider_status": 429,
  "duration_ms": 312,
  "freshness_served": "stale"
}
```

Uma linha JSON por evento, com horário UTC, nível, serviço, ambiente e campos de correlação. Nginx não deve registrar corpo, cookie nem `Authorization`. O `X-Request-ID` vindo da internet deve ser validado e limitado ou substituído na borda.

O README deve mostrar como consultar logs com `docker compose logs` e explicar como diagnosticar um provedor em resfriamento.

## 5.10 CI/CD

Criar um pipeline que execute, no mínimo:

1. instalação reproduzível das dependências;
2. testes PHP;
3. testes frontend;
4. análise estática ou lint;
5. build de produção;
6. falha do pipeline quando uma etapa obrigatória falhar.

A referência é GitHub Actions. Se a plataforma impedir a execução no repositório privado, será aceita a execução local do mesmo script, com o arquivo de pipeline e o log completo.

O pipeline não deve depender de serviço externo, conter segredos no YAML, realizar deploy real ou exigir infraestrutura paga. **Não pode chamar os provedores de mercado.**

## 5.11 README e Postman

### README

Deve conter: objetivo e escopo; requisitos locais; instalação e execução; configuração do banco; variáveis de ambiente sem valores secretos; comandos de testes e build; contrato resumido da API; decisões de arquitetura e DDD; estratégia de autenticação e autorização; TTLs de cache e política de resfriamento por provedor; política de retenção da auditoria; como consultar auditoria e logs; comportamento em degradação; considerações de segurança, incluindo a defesa contra XXE; limitações conhecidas; tempo gasto; próximos passos.

### Postman

Collection e environment sem segredos, cobrindo: heatmap; notícias por ativo; notícias global; símbolo inválido; login; recuperação do usuário autenticado; auditoria autorizada e negada; logout.

A collection deve manter o cookie jar e documentar a obtenção e o reenvio do token CSRF. Credenciais exclusivamente locais e fictícias são fixtures de desenvolvimento, não segredos reais.

OpenAPI pode ser entregue adicionalmente, mas não substitui a collection obrigatória.

## 5.12 Docker Compose e execução local

### Arquitetura de referência

```text
                        porta pública :8080
                               │
                               ▼
                     ┌──────────────────┐
                     │ web              │
                     │ Nginx + assets   │
                     └────────┬─────────┘
                              │ FastCGI/rede interna
                              ▼
                     ┌──────────────────┐
                     │ api              │
                     │ PHP-FPM          │
                     └────────┬─────────┘
                              │ SQL/rede interna
                              ▼
                     ┌──────────────────┐
                     │ db               │
                     │ Postgres/MySQL   │
                     └──────────────────┘

                     ┌──────────────────┐
                     │ migrate          │
                     │ job de uma vez   │
                     └────────┬─────────┘
                              └── usa a mesma imagem e o mesmo banco
```

### Responsabilidade de cada container

#### `web`

- ser o único serviço exposto ao host;
- servir os assets compilados;
- encaminhar requisições PHP para `api`;
- encaminhar ou gerar `X-Request-ID`;
- aplicar os cabeçalhos de segurança da seção 5.7.3;
- registrar access/error logs em `stdout` e `stderr`;
- não conter regra de negócio nem credencial de banco.

#### `api`

- executar PHP-FPM;
- validar, autenticar e autorizar;
- executar os casos de uso e a egressão para os provedores;
- persistir usuários e auditoria;
- escrever logs estruturados em `stdout`/`stderr`;
- não expor porta ao host;
- não executar servidor de desenvolvimento do frontend no perfil padrão.

#### `db`

- executar PostgreSQL ou MySQL;
- usar volume nomeado;
- possuir health check nativo (`pg_isready` ou `mysqladmin ping`);
- permanecer acessível apenas na rede interna.

#### `migrate`

- reutilizar a imagem da API;
- aguardar o banco saudável;
- executar migrations e seed local mínimo de forma idempotente;
- criar o administrador de demonstração sem duplicar em segunda execução;
- nunca executar comandos destrutivos;
- terminar com código zero antes de a API ficar pronta;
- não permanecer como daemon.

Node.js deve ser usado em estágio de build multi-stage. Container de desenvolvimento do frontend é aceitável apenas em profile separado e não deve ser necessário para a avaliação.

Requisitos obrigatórios:

- `docker compose build` constrói as imagens sem etapas manuais;
- `docker compose up -d` inicia Nginx, aplicação e banco;
- migrations e dados mínimos aplicados automática e idempotentemente;
- a aplicação aguarda o banco, sem tentativa manual;
- banco, API e entrada HTTP possuem health checks úteis;
- apenas o serviço `web` publica porta no host;
- comunicação interna por nome de serviço, nunca `localhost` ou IP fixo;
- nenhuma credencial real no Compose ou no repositório;
- `.env.example` pode conter senha de banco e credencial administrativa exclusivamente locais e fictícias;
- frontend de produção compilado durante o build;
- logs em `stdout`/`stderr`;
- ambiente funcional sem PHP, Composer, Node.js ou banco instalados no host;
- `docker compose down -v` permite reconstrução limpa;
- README informa URL, usuário de demonstração e comandos de diagnóstico.

Ordem mínima de prontidão:

```text
db healthy → migrate Exited (0) → api healthy → web healthy
```

O Compose deve expressar essa ordem com `service_healthy` e `service_completed_successfully`, ou mecanismo equivalente sem `sleep N`. O job `migrate` deve usar `restart: "no"`.

Contrato mínimo de saúde:

- `db`: probe nativo com o usuário configurado;
- `api`: probe interno que confirme PHP e a migration de `provider_calls` disponíveis;
- `web`: `GET /api/health` atravessando Nginx e PHP, com `200` e `{"status":"ok"}`;
- a resposta de saúde **não** deve depender dos provedores externos;
- resposta sem versão, ambiente, credenciais ou stack trace;
- interval, timeout, retries e start period explícitos.

Motivo da penúltima regra: health check que chama provedor externo transforma instabilidade de terceiro em container reiniciando em laço.

O avaliador não corrigirá Dockerfile, permissões, dependências, migrations ou variáveis de ambiente para conseguir iniciar a solução.

### Verificação esperada pelo candidato

Antes da entrega, validar em um clone limpo:

```bash
docker compose build
docker compose up -d
docker compose ps --all
curl --fail http://localhost:8080/api/health

# sem sessão: os dois widgets devem responder 401
curl -i 'http://localhost:8080/api/v1/heatmap?market=br'
curl -i 'http://localhost:8080/api/v1/news?symbol=PETR3.SA'

# com sessão: autentique com o usuário de demonstração e reutilize o cookie jar
# (o passo de obtenção do token CSRF deve estar documentado no seu README)
curl -c jar.txt -b jar.txt -sS -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"<usuario-de-demonstracao>","password":"<senha-local-ficticia>"}'
curl -b jar.txt -i 'http://localhost:8080/api/v1/heatmap?market=br'
curl -b jar.txt -i 'http://localhost:8080/api/v1/news?symbol=PETR3.SA'
curl -b jar.txt -i 'http://localhost:8080/api/v1/news?region=br'

docker compose exec -T api <comando-de-testes-documentado>
docker compose down -v
```

Após `up`, `web`, `api` e `db` devem estar saudáveis e `migrate` deve aparecer como `Exited (0)`.

---

# 6. Entregáveis

1. API PHP com os dois widgets, autenticação e auditoria;
2. clientes dos provedores com timeout, cache, emergência e resfriamento;
3. migrations de usuários e `provider_calls`;
4. frontend tipado com os dois widgets e a tela administrativa;
5. testes backend e frontend rodando offline;
6. pipeline de CI;
7. README da solução;
8. collection e environment Postman sem segredos;
9. Dockerfile e Docker Compose funcionais;
10. histórico Git progressivo e explicação de um ciclo RED/GREEN/REFACTOR;
11. lista objetiva de limitações e próximos passos;
12. URL do repositório privado compartilhado ou arquivo Git bundle.

---

# 7. Critérios objetivos de aceite

A entrega será considerada funcional quando:

1. um clone limpo iniciar com `docker compose build` e `docker compose up -d`;
2. migration e rollback funcionarem no banco escolhido;
3. um visitante sem sessão ser redirecionado para o login ao abrir a aplicação;
4. `GET /api/v1/heatmap` e `GET /api/v1/news` responderem `401` sem sessão, sem devolver dado de mercado;
5. o usuário de demonstração conseguir autenticar e, só então, visualizar os dois widgets;
6. o heatmap renderizar células agrupadas por setor, dimensionadas por valor de mercado e coloridas pela variação, nos mercados `US` e `BR`;
7. o heatmap `BR` exibir tickers com sufixo `.SA` e sem duplicatas do mercado fracionário;
8. clicar numa célula do heatmap mudar o símbolo do widget de notícias;
9. o widget de notícias alternar entre os três modos e exibir manchetes em cada um;
10. dois usuários autenticados diferentes compartilharem o mesmo cache de provedor;
11. nenhuma requisição a domínio de terceiros partir do navegador;
12. símbolo inválido retornar `422` para usuário autenticado e `401` para anônimo;
13. feed com entidade externa não ler arquivo local nem acessar a rede;
14. link `javascript:` ser renderizado como texto inerte, não como âncora;
15. cache válido ser servido sem nova chamada ao provedor;
16. falha de provedor com cache de emergência responder `200` com `freshness: stale`;
17. falha total responder `503` nomeando a capacidade afetada;
18. `429` do provedor abrir resfriamento e gerar `provider.skipped`;
19. falha de uma fonte de notícias não impedir a exibição da outra;
20. toda tentativa externa gerar registro de auditoria correlacionável por `request_id`;
21. a auditoria não conter título, URL de manchete, cotação, corpo de resposta, IP ou cookie;
22. usuário comum ser impedido de consultar a auditoria;
23. administrador conseguir filtrar e paginar os registros;
24. logout invalidar a sessão e revogar o acesso aos widgets;
25. a suíte executar sem depender de provedor externo;
26. testes, build e pipeline finalizarem com sucesso;
27. README e Postman reproduzirem o fluxo, incluindo login;
28. `web`, `api` e `db` ficarem saudáveis e `migrate` terminar com `Exited (0)`;
29. apenas Nginx estar exposto ao host no perfil padrão;
30. o repositório não conter segredos reais em nenhum commit do histórico.

---

# 8. Fora do escopo

Não é necessário:

- implementar outros widgets do terminal;
- implementar cotações, gráficos, opções, cripto ou macro;
- calcular indicadores técnicos;
- implementar o mercado europeu do botão `EU`;
- traduzir manchetes ou aplicar análise de sentimento;
- implementar WebSocket ou streaming em tempo real;
- construir dashboard de métricas ou observabilidade;
- provisionar infraestrutura em nuvem;
- implementar a mesma interface em React e Vue;
- atingir 100% de cobertura;
- criar microsserviços;
- suportar MySQL e PostgreSQL em execução simultânea;
- implementar filas ou Redis.

---

# 9. Bônus

Bônus não compensam ausência dos requisitos fundamentais.

- mercado `EU` do heatmap, com uma requisição por país e normalização cambial por adapter até 3 pontos;
- segundo provedor de heatmap com adapter intercambiável e fake nos testes até 1 ponto;
- disjuntor com meia-abertura e recuperação progressiva, com teste determinístico até 2 pontos;
- cache compartilhado entre réplicas, com invalidação documentada até 1 ponto;
- OpenAPI validado automaticamente até 1 ponto;
- teste de carga reproduzível sobre o cache até 1 ponto;
- métricas reais com OpenTelemetry ou Prometheus até 2 pontos;
- imagens executadas por usuário sem privilégios e com SBOM até 1 ponto;
- profile Docker Compose separado para observabilidade até 1 ponto;
- testes automatizados de acessibilidade até 1 ponto;
- demonstração de como estruturaria os widgets em Vue até 1 ponto.

**Limite de bônus: 15 pontos. Nota final = `min(100, pontuação-base + bônus)`.**

---

# 10. Rubrica de avaliação

Pontuação base: **100 pontos**. Nota recomendada para aprovação: **70 pontos**, desde que nenhum bloqueador seja acionado.

| Área | Pontos | Evidências esperadas |
|---|---:|---|
| Fluxo funcional | 14 | Heatmap correto (5); notícias corretas (5); área administrativa (4) |
| PHP e API HTTP | 13 | Contratos e validação (4); frescor e erro específico (4); sessão/CSRF (3); controllers finos (2) |
| Frontend e TypeScript | 10 | Treemap e lista (4); estados e sinalização de `stale` (3); contratos tipados e acessibilidade (3) |
| SQL | 9 | Migrations/rollback (2); chaves e índices (3); idempotência de `call_id` (2); paginação e filtros (2) |
| DDD pragmático | 6 | Invariantes e allowlists no domínio (3); domínio livre de infraestrutura (2); simplicidade (1) |
| TDD e testes | 15 | Casos backend obrigatórios (10); dois fluxos frontend (3); suíte offline e determinística (2) |
| Segurança | 12 | Defesa contra XXE (4); saneamento de links (3); egressão e CSP (3); segredos e logs seguros (2) |
| Resiliência, cache e logs | 9 | Cache e emergência (3); resfriamento (2); falha parcial (2); correlação e auditoria × log (2) |
| Docker Compose e CI/CD | 8 | Build e topologia (4); prontidão e migrations (2); pipeline offline (2) |
| README e Postman | 4 | README reproduzível (2); collection com sessão/CSRF (2) |
| **Total** | **100** | |

## 10.1 Faixas de avaliação

- **90–100:** solução pronta para evoluir, com decisões claras e poucos riscos.
- **80–89:** solução sólida, com lacunas pequenas e identificadas.
- **70–79:** atende ao núcleo, mas possui dívidas técnicas ou operacionais relevantes.
- **60–69:** implementação parcial ou frágil em mais de uma competência central.
- **Abaixo de 60:** não demonstra domínio suficiente do fluxo completo.

## 10.2 Bloqueadores e limites de nota

| Situação | Consequência recomendada |
|---|---|
| XXE explorável no parsing de RSS | Reprovação |
| Segredo real ou token reutilizável versionado | Reprovação |
| Senha persistida sem hash seguro | Reprovação |
| Link `javascript:` renderizado como âncora clicável | Nota máxima 59 |
| Navegador chamando provedor externo diretamente | Nota máxima 59 |
| Aplicação não inicia ou um dos widgets não funciona | Nota máxima 59 |
| Endpoint administrativo sem autorização | Nota máxima 59 |
| Símbolo interpolado em URL sem allowlist | Nota máxima 59 |
| Ausência total de testes backend | Nota máxima 49 |
| Suíte dependente de provedor externo | Nota máxima 69 |
| Token de sessão em `localStorage` sem justificativa e mitigação | Nota máxima 69 |
| Auditoria contendo conteúdo de notícia, cotação ou IP do visitante | Nota máxima 69 |
| `503` devolvido havendo cache de emergência utilizável | Perda dos pontos de resiliência |
| Health check dependente de provedor externo | Perda dos pontos de prontidão |
| Pipeline inexistente | Perda dos pontos de CI/CD |
| `docker compose build` ou `up -d` não funciona em clone limpo | Nota máxima 59 |
| Repositório privado não compartilhado com os avaliadores | Entrega não avaliável até a correção |

Os limites são aplicados depois da soma da rubrica e dos bônus. O mesmo defeito perde os pontos dos critérios afetados, mas não recebe segundo desconto numérico além do teto aplicável.

---

# 11. Roteiro sugerido para a entrevista técnica

Reservar de 45 a 60 minutos para:

1. **Demonstração 10 minutos**

   O candidato executa os dois widgets e força um cenário de degradação.

2. **Arquitetura e DDD 10 minutos**

   Explica limites entre domínio, aplicação, infraestrutura e apresentação, e por que o domínio não conhece o parser.

3. **Banco de dados 8 minutos**

   Justifica chaves, índices, JSON, idempotência, paginação e diferenças entre MySQL e PostgreSQL.

4. **Segurança 10 minutos**

   Explica a defesa contra XXE, o saneamento de links e a fronteira de egressão.

5. **Incidente 10 minutos**

   Cenário: o heatmap ficou vazio para todos os usuários e a auditoria mostra aumento de `provider.failed` com `failure_code=rate_limited`. O candidato usa `request_id`, logs, health checks e SQL para diagnosticar e propor mitigação.

6. **Docker e entrega 7 minutos**

   Explica build, inicialização, health checks, migrations e diagnóstico dos containers.

### Perguntas úteis

- Por que o navegador não deve falar direto com o provedor, se o dado é público?
- O que exatamente `LIBXML_NOENT` faria com um feed malicioso?
- Como você provou, em teste, que a entidade externa não foi resolvida?
- Por que um item com link inseguro continua na lista em vez de ser descartado?
- Qual a diferença prática entre `cached` e `stale` para quem está olhando a tela?
- Quando responder `503` é correto, e quando é defeito?
- Como impedir que uma rajada de usuários derrube seu limite junto ao provedor?
- Que informação você deliberadamente decidiu não registrar na auditoria?
- Por que o health check não deve consultar o provedor externo?
- Como adaptaria migrations e consultas de PostgreSQL para MySQL?
- Qual teste trouxe mais confiança e qual risco permaneceu sem cobertura?

---

# 12. Mapeamento com a descrição da vaga

| Item da vaga | Evidência no teste |
|---|---|
| Manutenção corretiva e evolutiva | Normalização e degradação de provedores instáveis sem regressão |
| Novas funcionalidades com a liderança | Especificação funcional, decisões registradas e discussão técnica |
| PHP e API REST | Egressão, contratos versionados, sessão e casos de uso |
| APIs REST externas | Núcleo do desafio: três provedores, timeout, cache, resfriamento e fakes |
| JavaScript, TypeScript e React | Treemap, lista de notícias, estados e tela administrativa tipada |
| Vue | Discussão comparativa na entrevista e bônus opcional |
| MySQL/PostgreSQL | Migrations, chaves, índices, JSON, paginação, idempotência e portabilidade |
| Ambientes e deploy | Docker Compose reproduzível, health checks e execução por clone limpo |
| Segurança, performance e escalabilidade | XXE, saneamento de links, CSP, limite de taxa, cache e índices |
| Versionamento, testes e CI/CD | Histórico Git, TDD, pipeline offline e build |
| Monitoramento e incidentes | Auditoria durável, logs estruturados, correlação e cenário de falha |
| README e Postman | Entregáveis obrigatórios |
| DDD e TDD | Invariantes de domínio, casos de uso e ciclo RED/GREEN/REFACTOR |
| Melhoria contínua | Limitações, próximos passos e bônus técnicos |

---

# 13. Orientação ao avaliador

Priorize, nesta ordem:

1. segurança no tratamento de conteúdo de terceiros;
2. respeito à fronteira de egressão;
3. correção dos dois widgets;
4. comportamento sob falha de provedor;
5. clareza dos limites arquiteturais;
6. testes capazes de detectar regressões sem depender da internet;
7. simplicidade compatível com o timebox;
8. qualidade da documentação e das justificativas.

Quantidade de abstrações, volume de código, cobertura isolada ou número de bônus não devem valer mais do que uma solução pequena, executável e bem fundamentada.

Atenção especial a duas armadilhas frequentes neste desafio:

- **Parsing “que funciona”.** Um feed bem formado passa em qualquer implementação. A diferença entre um candidato que entende o risco e um que não entende só aparece nas fixtures maliciosas.
- **Degradação invisível.** Uma aplicação que responde `200` com lista vazia quando o provedor caiu parece correta na demonstração e é inútil em produção. Verifique `freshness`, o cabeçalho e o erro específico.
# Teste-pr-tico-Desenvolvedor-a-Full-Stack-
# Teste-pr-tico-Desenvolvedor-a-Full-Stack-
