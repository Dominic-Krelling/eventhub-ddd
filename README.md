# EventHub — Proposta de arquitetura (DDD, mensageria e SOLID)

Reorganização arquitetural do **EventHub**, plataforma web de eventos culturais

O código-fonte atual está em: https://github.com/Dominic-Krelling/EventHub-Plataforma-de-Eventos-Culturais


## Conteúdo

| Caminho | O que é |
|---|---|
| [`docs/ARQUITETURA.md`](docs/ARQUITETURA.md) | Questões 1 e 2: contextos DDD, mapeamento do código atual, mensageria e C4 |
| [`docs/ANALISE-SOLID.md`](docs/ANALISE-SOLID.md) | Questão 3: metodologia de inspeção, achados com trecho e linha, refatorações |
| [`docs/diagramas/`](docs/diagramas) | Fontes PlantUML: C4 níveis 1, 2 e 3, fluxo da Outbox e mapa de contextos |
| [`src/`](src) | Estrutura de pastas proposta (backend), apenas diretórios |

## Resumo da proposta

- **Monólito modular** organizado por contexto de negócio (`identity`, `events`, `interactions`, `discovery`, `notifications`), cada um com as camadas `domain`, `application`, `infrastructure` e `presentation`.
- **Mensageria** com RabbitMQ, padrão Outbox e consumidores idempotentes, como evolução futura. Não existe no código atual.
- **SOLID**: seis achados no código atual, mais uma recomendação preventiva sobre OCP.
