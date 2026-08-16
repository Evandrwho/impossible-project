# NetGo — Projeto Impossível: Espelho do Netflix em Go

Projeto de portfólio para consolidar o plano de estudos de Go sênior (`plano-go-v2.html`), aplicando cada conceito num cenário concreto de streaming de vídeo, no estilo Netflix.

## Arquitetura geral

Microsserviços:

- `catalog-service` — catálogo de títulos, metadados, busca
- `playback-service` — autoriza e serve streaming, telemetria de reprodução
- `billing-service` — assinaturas, cobrança, planos
- `encoding-worker` — pipeline de transcodificação de vídeo
- `recommendation-service` — histórico e sugestões
- `api-gateway` — entrada única, REST → gRPC, auth, rate limiting

Comunicação interna via gRPC. Eventos assíncronos via Kafka. Deploy em Kubernetes.

## Mapeamento módulo → cenário

### 01 · Concorrência → Encoding Pipeline
Upload de vídeo dispara worker pool: cada arquivo gera N goroutines processando resoluções (1080p, 720p, 480p, mobile). Pool fixo evita explosão de goroutines em pico de upload. Rate limiting controla encodes simultâneos (limite de CPU). Cancelamento gracioso: se admin cancela upload no meio, workers terminam limpo.

### 02 · Performance → Catalog Browsing / Homepage
Endpoint que monta a homepage ("Continue assistindo", "Recomendados", "Em alta") é hot path, chamado toda sessão. Usar pprof para achar alocação excessiva ao montar JSON de milhares de títulos. Reduzir p99 de ~120ms para ~18ms via escape analysis e sync.Pool.

### 03 · gRPC → Comunicação interna entre serviços
`playback-service` chama `billing-service` para checar assinatura ativa antes de liberar stream. Deadline propagation crítico — requisição de play não pode ficar pendurada 8s. Streaming gRPC serve para `ListEpisodes` (server streaming) e telemetria de qualidade de vídeo (client streaming, bitrate reportado a cada segundo).

### 04 · Kafka → Watch Events
Toda interação do player (play, pause, seek, progress a cada 10s, finished) vira evento no topic `watch.events`. Consumers: recommendation-service (atualiza histórico), analytics, "continue assistindo". Outbox Pattern garante que progresso salvo no banco sempre gera evento — sem isso, recomendação fica desatualizada silenciosamente.

### 05 · K8s → Deploy do playback-service
Deploy não pode derrubar stream de quem está assistindo. Graceful shutdown drena conexões WebSocket/gRPC ativas antes de matar o pod. Readiness probe verifica conexão com Redis (cache de sessão) antes de receber tráfego.

### 06 · Fintech Patterns → Billing/Assinatura
Idempotência: clique duplo em "assinar" não pode cobrar duas vezes. Saga: fluxo de upgrade de plano (cobrar cartão → atualizar entitlement → notificar) com compensação se a cobrança falhar. CQRS: write model transacional para billing, read model denormalizado para "meu plano atual". Circuit breaker para o gateway de pagamento externo.

### 07 · Go Idioms → Refatorar recommendation-service
Interface pequena para o motor de recomendação: `type Recommender interface { Suggest(userID string) []Title }`. Troca de implementação (regra simples → ML) sem tocar client. Error wrapping em toda a cadeia catalog → playback → billing.

### 08 · Testes → Fuzz e Integration
`FuzzValidateWatchProgress` testando timestamp negativo, maior que duração total, etc. Integration test com testcontainers subindo Postgres + Kafka reais para validar pipeline de watch-events ponta a ponta.

### 09 · Projeto Final → Fluxo completo
Usuário dá play → gRPC playback-service autentica via billing (idempotente) → evento Kafka `watch.events` → outbox → recommendation atualiza → tudo rodando em K8s com zero-downtime → suite de testes com `-race`. README com flame graph do endpoint de homepage otimizado.

## Próximos passos

- Definir escopo mínimo viável (quantos serviços entram na v1)
- Montar estrutura de monorepo com `go work init`
- Priorizar ordem de implementação seguindo a ordem dos módulos do plano de estudos
