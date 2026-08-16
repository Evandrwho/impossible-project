# CLAUDE.md — ingest-service

Contexto pra qualquer sessão do Claude Code trabalhando neste diretório. Leia antes de tocar em código.

## O que é este serviço

`ingest-service` é uma peça do **Mini-Netflix**, o projeto impossível construído em cima do [plano de estudos de Go sênior](../plano-go-v2.html) (`impossible-project/plano-go-v2.html`). O objetivo maior não é "fazer um clone da Netflix" — é fixar conceitos reais de concorrência, performance, gRPC, Kafka, K8s e padrões distribuídos, cada um resolvendo um problema concreto que existe em escala real.

Este serviço especificamente cobre o **Módulo 01 (Concorrência)** do plano: recebe o `master` (arquivo de vídeo original enviado por um estúdio) e transcodifica em ~20 `renditions` (combinações de codec/resolução/bitrate que formam a `bitrate ladder`).

## Fonte de verdade

`plano-go-v2.html` (aba "Concorrência") é a fonte de verdade pra escopo, nomenclatura e critério de conclusão de cada etapa. Existe também `plano-netgo.md` na raiz do repo — é um rascunho anterior com nomes diferentes (`encoding-worker` em vez de `ingest-service`, `billing-service`/`playback-service` em vez de `playback-api`). **Está superado.** Se os dois documentos divergirem, `plano-go-v2.html` vence.

## Decisões já tomadas (não reabrir sem motivo)

- **Worker pool com N fixo**, não goroutine por job. `N` é dimensionado pelos cores disponíveis no nó de encode — nunca hardcoded arbitrário nem ilimitado. Essa é a lição central do módulo: paralelismo controlado pelo que o nó aguenta, não por quantos jobs chegam.
- **Rate limiting** no despacho de jobs (via `time.Ticker`) — protege o storage compartilhado de throughput, não é só "boa prática".
- **Cancelamento gracioso via `context`** — um encode em andamento sempre termina antes do processo encerrar; nunca aborta um vídeo pela metade.
- **`EncodeJob` / `EncodeResult`** são os nomes de struct já definidos no plano — mantenha, não renomeie sem atualizar `plano-go-v2.html` junto.
- **`goleak` + `go test -race` obrigatórios** em qualquer teste que envolva o pool. Zero goroutine leak é critério de conclusão, não sugestão.

## Fora de escopo pra este serviço

- Validação de qualidade pós-encode (QC) — outro serviço.
- Decidir se o título aparece no catálogo — `catalog-service`.
- DRM/licenciamento — `license-service`.
- Publicação real em CDN — fora do escopo deste módulo do plano.

## Vocabulário do domínio

- **master** — arquivo de vídeo original em altíssima qualidade, entregue pelo estúdio, antes de qualquer compressão.
- **rendition** — uma versão comprimida e pronta pra entrega (codec + resolução + bitrate específicos).
- **bitrate ladder** — o conjunto ordenado de renditions de um título, do mais leve ao mais pesado.
- **codec** — algoritmo de compressão (H.264, VP9, AV1).

## Estado atual

Scaffold criado (`go build ./...` verde): `cmd/ingest-service/main.go` (entrypoint vazio) + `internal/encode/doc.go` (pacote vazio, só doc comment). Módulo `github.com/evandrobentoalves/mini-netflix/ingest-service`, `go 1.26`, zero dependências externas até agora. Nenhuma lógica de negócio ainda — seguir a ordem de etapas do README (structs → pool → rate limit → shutdown gracioso → goleak), uma por vez, sem pular. Tudo que for lógica de domínio entra em `internal/encode/`, nunca em `main.go`.

Nota: o repositório em `impossible-project/.git` está com o diretório `.git` incompleto (sem `HEAD`/`config`/`index` — `git status` falha com "not a git repository"). Não foi corrigido ainda; se for mexer em versionamento, checar isso primeiro.
