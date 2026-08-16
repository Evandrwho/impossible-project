# ingest-service

Serviço que recebe o vídeo bruto enviado por um estúdio e transforma esse arquivo único em todas as versões (`renditions`) que o catálogo vai poder oferecer aos usuários. Sem esse serviço, não existe título publicável — é o primeiro elo da cadeia entre "chegou o episódio" e "está disponível pra assistir".

Parte do **Mini-Netflix**, o projeto impossível que acompanha o [plano de estudos de Go sênior](../plano-go-v2.html). Cada decisão técnica daqui existe pra fixar um conceito específico do plano — ver seção "Por que existe" abaixo.

## Responsabilidade

- Receber o `master` (arquivo original em altíssima qualidade) de um episódio ou filme.
- Gerar a `bitrate ladder`: um conjunto de `renditions` (combinações de codec, resolução e bitrate) a partir do master.
- Publicar, via Outbox Pattern, o evento de que cada rendition ficou pronta — pra quem consome (QC, catalog-service) seguir o pipeline.

O que **não** é responsabilidade deste serviço: validar qualidade da rendition (isso é do QC), decidir se o título fica visível no catálogo (isso é do `catalog-service`), nem lidar com licenciamento/DRM (isso é do `license-service`). `ingest-service` só transforma o arquivo — o resto do pipeline decide o que fazer com o resultado.

## Por que existe (o problema real)

Em dia normal, 1 título chega por vez, e transcodificar cada uma das ~20 renditions numa goroutine solta (`go transcode(r)`) parece funcionar. O problema aparece no pico: um drop de temporada inteira (10 episódios × 20 renditions = 200 encodes de uma vez) ou uma migração de codec em massa no catálogo geram milhares de goroutines simultâneas, cada uma segurando buffers de frame pesados na memória — o nó estoura em OOMKill e o pipeline trava pra todo mundo, não só pra quem causou o pico.

A solução é um **worker pool com número fixo de workers** (dimensionado pelos cores do nó de encode), consumindo jobs de uma fila. Paralelismo e memória ficam previsíveis independente de quantos títulos chegam de uma vez, e cancelamento gracioso garante que um encode em andamento termina antes do nó ser drenado.

Esse é o Módulo 01 (Concorrência) do plano de estudos — ver `plano-go-v2.html` pra a narrativa completa, o comparativo jeito errado/jeito certo, e os passos de implementação em blocos de ~1h.

## Estrutura

```
ingest-service/
├── cmd/
│   └── ingest-service/
│       └── main.go       # entrypoint — só wiring, zero lógica de negócio
├── internal/
│   └── encode/
│       └── doc.go        # onde vive EncodeJob, EncodeResult e worker (ainda vazio)
├── go.mod
├── README.md
└── CLAUDE.md
```

Layout baseado no **Standard Go Project Layout** (`golang-standards/project-layout`, convenção de comunidade, não é regra oficial do time Go), na versão enxuta — sem `pkg/`, `configs/` ou `deployments/` que esse serviço ainda não precisa:

- **`cmd/ingest-service/main.go`** — o binário. Só monta as dependências (config, logger, o pool) e chama o que existe em `internal/`. Não deve crescer lógica própria — se crescer, é sinal de que devia estar em `internal/`.
- **`internal/`** — todo o código de domínio. O compilador Go impede que qualquer módulo *fora* deste repositório importe pacotes daqui — é a forma da linguagem de dizer "isso é implementação interna, não uma API pública que alguém pode depender".
- **`internal/encode/`** — um pacote por responsabilidade, não um pacote genérico `internal/utils`. `encode` vai concentrar `EncodeJob`, `EncodeResult` e a função `worker` do worker pool (módulo 01 do plano). Conforme o serviço crescer (ex: publicação no outbox), cada responsabilidade nova vira seu próprio pacote em `internal/`, não um arquivo a mais dentro de `encode`.
- **Sem `pkg/`** — esse diretório existe no layout padrão só pra código que *outros módulos externos* vão importar. Como `ingest-service` é um microsserviço isolado, não um SDK, criar `pkg/` agora seria abstração sem uso — vai contra o proverbio Go "a little copying is better than a little dependency".
- **`go.mod`** — módulo já inicializado (`go 1.26`), sem dependências externas ainda; a primeira vai entrar quando `goleak` for necessário (passo 6 abaixo).

Rodar o scaffold atual (só imprime que o worker pool ainda não existe):

```
go run ./cmd/ingest-service
```

## Status atual

Scaffold criado e compilando (`go build ./...` verde). Nenhuma lógica de negócio ainda — `internal/encode/doc.go` só documenta o que vai morar ali.

## Próximos passos

1. Structs `EncodeJob` / `EncodeResult` e a função `worker` em `internal/encode`
2. Pool com N workers fixos + `sync.WaitGroup`
3. Rate limiting com `time.Ticker` (protege o storage compartilhado)
4. Cancelamento gracioso via `context` + `signal.NotifyContext`
5. Validação de zero goroutine leaks com `goleak` + `go test -race`

Ordem e detalhe de cada etapa em `plano-go-v2.html`, aba "Concorrência".
