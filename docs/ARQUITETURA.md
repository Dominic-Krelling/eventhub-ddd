# Arquitetura proposta para o EventHub

## 1. Domínio

O EventHub trata da publicação, da descoberta e da interação com eventos culturais. Uma pessoa cria conta, publica eventos, consulta o feed, visita perfis públicos, curte e comenta.

### Linguagem ubíqua

Código, documentação e conversas da equipe usam os mesmos termos: **Usuário, Evento, Curtida, Comentário, Feed, Perfil Público** e **Evento Publicado**. "Curtida" tem um único significado: a relação de um usuário com um evento, que pode ser criada ou removida.

## 2. Situação atual

O backend (`eventhub-api`) é um monólito modular em Node.js, Express e Sequelize. Cada módulo em `modules/` reúne controller, service, model, routes e, quando há, validator. O frontend (`eventhub-frontend`) é uma aplicação Vue 3 com views, componentes, composables, stores e serviços HTTP.

Os limites entre os módulos ainda são pouco explícitos. Por exemplo, `eventService.js` importa os modelos de evento, usuário, curtida e comentário e a conexão Sequelize, e `userService.js` reúne autenticação, perfil e remoção de arquivos.

## 3. Bounded Contexts (Questão 1)

| Contexto | Responsabilidade | Elementos centrais |
|---|---|---|
| Identidade e Perfil | Cadastro, login, autenticação e dados de perfil | Usuário, Perfil, Autenticação |
| Gestão de Eventos | Criação, edição, publicação, consulta e remoção | Evento, EventRepository, casos de uso de evento |
| Interações | Curtidas e comentários em eventos | Curtida, Comentário |
| Descoberta | Feed, paginação e pesquisa de eventos e usuários | FeedQuery, SearchService |
| Notificações (futuro) | Reação a fatos relevantes, como novo evento ou comentário | Notification, NotificationHandler |

Mapa de contextos: [`diagramas/contextos-ddd.puml`](diagramas/contextos-ddd.puml).

### Camadas de cada contexto

- **domain**: conceitos e regras de negócio, sem dependência de framework ou banco.
- **application**: casos de uso, que coordenam o domínio e a infraestrutura.
- **infrastructure**: Sequelize, upload de arquivos, JWT, bcrypt, mensageria.
- **presentation**: rotas, controllers e validadores HTTP.

A regra de dependência é: `presentation` e `infrastructure` apontam para `application` e `domain`, nunca o contrário.

### Estrutura-alvo

```text
src/
  shared/
    domain/            erros e tipos comuns
    infrastructure/    banco, logger, mensageria, configuração
    presentation/      middlewares transversais, formato das respostas
  identity/
    domain/  application/  infrastructure/  presentation/
  events/
    domain/  application/  infrastructure/  presentation/
  interactions/
    likes/       domain/  application/  infrastructure/  presentation/
    comments/    domain/  application/  infrastructure/  presentation/
  discovery/
    feed/        application/  infrastructure/  presentation/
    search/      application/  infrastructure/  presentation/
  notifications/       (futuro) application/  infrastructure/
  app.js
```

### Mapeamento do código atual para a estrutura-alvo

Caminhos relativos a `eventhub-api/` no repositório atual.

| Arquivo atual | Destino proposto |
|---|---|
| `modules/user/userService.js` | `identity/application/` (RegisterUser, LoginUser, UpdateProfile, GetProfile) |
| `modules/user/userModel.js` | `identity/infrastructure/` (modelo Sequelize) |
| `modules/user/userController.js`, `userRoutes.js`, `userValidator.js` | `identity/presentation/` |
| `config/jwt.js` | `identity/infrastructure/` |
| `middlewares/auth.js`, `optionalAuth.js` | `identity/presentation/` |
| `middlewares/profileMulter.js` | `identity/infrastructure/` |
| `modules/event/eventService.js` | `events/application/` (CreateEvent, UpdateEvent, DeleteEvent, GetEvent) |
| `modules/event/eventModel.js` | `events/infrastructure/` |
| `modules/event/eventController.js`, `eventRoutes.js`, `eventValidator.js` | `events/presentation/` |
| `middlewares/eventMulter.js` | `events/infrastructure/` |
| `modules/event/feedRoutes.js` | `discovery/feed/presentation/` |
| `modules/like/likeService.js` | `interactions/likes/application/` |
| `modules/like/likeModel.js` | `interactions/likes/infrastructure/` |
| `modules/like/likeController.js`, `likeRoutes.js` | `interactions/likes/presentation/` |
| `modules/comment/commentService.js` | `interactions/comments/application/` |
| `modules/comment/commentModel.js` | `interactions/comments/infrastructure/` |
| `modules/comment/commentController.js`, `commentRoutes.js`, `commentValidator.js` | `interactions/comments/presentation/` |
| `modules/search/*` | `discovery/search/` (application e presentation) |
| `config/database.js`, `config/associations.js`, `config/constants.js` | `shared/infrastructure/` |
| `middlewares/apiResponse.js`, `asyncHandler.js`, `errorHandler.js` | `shared/presentation/` |
| `routes/index.js`, `app.js`, `bin/www` | `src/app.js` e ponto de entrada (composição das rotas) |

No frontend, a recomendação é manter a estrutura por tipo (components, views, services) e aplicar as refatorações da [análise SOLID](ANALISE-SOLID.md).

### Por que monólito modular

Cada contexto poderia virar um microsserviço, mas isso traria custos de implantação, rede e consistência entre bancos que o projeto ainda não precisa assumir. O monólito modular dá limites claros entre as áreas de negócio e permite separar um contexto no futuro, se a necessidade aparecer.

## 4. Mensageria e modelo C4 (Questão 2)

### Necessidade de negócio

Ao publicar um evento, a plataforma tende a precisar de outras ações: avisar pessoas interessadas, atualizar métricas, alimentar um índice de busca, registrar auditoria. Se tudo isso acontecer dentro da mesma requisição HTTP, a resposta fica mais lenta, e a falha numa tarefa secundária pode comprometer a publicação. Com mensageria, a API grava o evento e publica uma mensagem; os consumidores reagem de forma assíncrona e novos consumidores podem ser acrescentados sem alterar quem publica.

### Escolha técnica

- **RabbitMQ** como broker: filas, confirmação de recebimento (ack) e reentrega, adequado para poucos produtores e consumidores com tarefas independentes. Um log distribuído como o Kafka seria mais pesado do que o problema exige agora.
- **Outbox**: a mensagem é gravada na mesma transação do evento, e um publicador a envia ao broker depois. Evita evento salvo sem mensagem registrada.
- **Consumidores idempotentes**: a mesma mensagem pode chegar mais de uma vez sem causar efeito duplicado.
- **Reentrega com espera progressiva e Dead Letter Queue** para mensagens que não puderem ser processadas.
- **Contrato versionado**, sem senhas, tokens ou dados pessoais desnecessários.

Adotar um broker acrescenta um componente a operar. A recomendação é introduzi-lo apenas quando notificações ou outro processamento assíncrono forem requisitos de fato.

### Eventos de domínio

| Evento | Quando | Uso possível |
|---|---|---|
| `EventCreated` | Evento persistido | Notificações, indexação |
| `EventUpdated` | Título, descrição ou imagem alterados | Atualizar índices |
| `EventDeleted` | Evento removido | Limpar dados derivados |
| `CommentCreated` | Comentário criado | Notificar o criador do evento |
| `LikeAdded` / `LikeRemoved` | Curtida criada ou removida | Métricas, notificações |

### Diagramas

| Diagrama | Arquivo |
|---|---|
| C4 nível 1 — Contexto | [`diagramas/c4-nivel1-contexto.puml`](diagramas/c4-nivel1-contexto.puml) |
| C4 nível 2 — Contêineres | [`diagramas/c4-nivel2-conteineres.puml`](diagramas/c4-nivel2-conteineres.puml) |
| C4 nível 3 — Componentes da API | [`diagramas/c4-nivel3-componentes-api.puml`](diagramas/c4-nivel3-componentes-api.puml) |
| Fluxo com Outbox e RabbitMQ | [`diagramas/fluxo-outbox.puml`](diagramas/fluxo-outbox.puml) |

Os níveis do C4 são três perspectivas do mesmo sistema. O nível 1 mostra o EventHub como um bloco e com quem ele se relaciona. O nível 2 mostra as partes executáveis e os armazenamentos, e onde o broker entra. O nível 3 abre a API e mostra a divisão interna de responsabilidades.

### O que existe hoje e o que é proposta

O código atual **não** tem RabbitMQ, Outbox nem consumidores. Nos diagramas, os elementos marcados "(futuro)" são propostos. Os demais (Frontend Vue, API Express, MySQL/MariaDB e uploads locais) existem hoje.
