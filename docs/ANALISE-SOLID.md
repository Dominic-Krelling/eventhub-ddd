# Análise dos princípios SOLID no código do EventHub

Código analisado: https://github.com/Dominic-Krelling/EventHub-Plataforma-de-Eventos-Culturais (branch `main`, commit `2f7bd40`). Todos os caminhos são relativos à raiz desse repositório. A análise é estática; nenhuma refatoração foi implementada, como permite o enunciado.

## 1. Metodologia de inspeção

Qualquer pessoa pode repetir a análise seguindo estes passos:

1. **Mapear** os módulos e arquivos do backend (`eventhub-api/modules`, `middlewares`, `config`) e do frontend (`eventhub-frontend/src`).
2. **Selecionar** os arquivos que concentram regras de negócio, acesso a dados, manipulação de arquivos ou comunicação HTTP: services, controllers, componentes com lógica e serviços HTTP.
3. **Listar**, para cada arquivo, as responsabilidades que ele exerce e as dependências que importa. Um comando útil: `grep -n "require(\|^import" <arquivo>`.
4. **Aplicar uma pergunta-guia por princípio**:
   - **SRP**: o arquivo muda por mais de um motivo?
   - **OCP**: acrescentar um comportamento exige editar código já estável?
   - **LSP**: uma implementação substituta quebraria o contrato esperado?
   - **ISP**: algum consumidor depende de mais do que usa?
   - **DIP**: a lógica de alto nível depende de implementações concretas?
5. **Registrar** o trecho de código e as linhas que sustentam cada achado.
6. **Classificar** cada achado como *violação observada* ou *oportunidade de melhoria*, e propor a refatoração.

Critérios adotados: o projeto é organizado em módulos e funções, não só em classes, então as unidades de análise são módulos, funções e componentes. Acessar a infraestrutura não é violação por si só; avalia-se o acúmulo de responsabilidades e o acoplamento que dificulta testar ou alterar o código. LSP e ISP não tiveram ocorrências relevantes, pois o código quase não usa herança nem interfaces formais.

## 2. Resumo

| # | Arquivo | Princípio | Classificação |
|---|---|---|---|
| 1 | `eventhub-api/modules/user/userService.js` | SRP | Violação observada |
| 2 | `eventhub-api/modules/event/eventService.js` | SRP | Violação observada |
| 3 | `eventhub-api/modules/like/likeService.js` | DIP | Oportunidade de melhoria |
| 4 | `eventhub-frontend/src/components/LikeButton.vue` | SRP | Violação observada |
| 5 | `eventhub-frontend/src/services/likeService.js` | DIP | Oportunidade de melhoria |
| 6 | `eventhub-api/modules/like/likeController.js` e `commentController.js` | SRP | Violação observada (duplicação) |
| — | `eventhub-api/modules/search/searchServices.js` | OCP | Recomendação preventiva |

## 3. Achados

### 3.1 SRP — `userService.js` concentra responsabilidades diferentes

Arquivo: `eventhub-api/modules/user/userService.js`, linhas 1–16 e 295–312.

```js
const User = require('./userModel');
const bcrypt = require('bcryptjs');
const fs = require('fs');
const path = require('path');
const Event = require('../event/eventModel')
const { generateToken } = require('../../config/jwt');
// ...
if (fs.existsSync(oldImagePath)) {
    try {
        fs.unlinkSync(oldImagePath);
    } catch (error) { /* ... */ }
}
```

O serviço faz cadastro (`registerUser`), login (`loginUser`), consulta de perfil público e privado (`getPublicProfile`, `getUserProfile`), atualização de perfil (`updateUserProfile`) e remoção da foto antiga no disco. São motivos de mudança distintos: regras de conta, política de senha e token, e armazenamento de imagens.

**Refatoração:** separar os casos de uso de identidade (`RegisterUser`, `LoginUser`), a gestão de perfil (`UpdateProfile`, `GetProfile`) e um `ProfileImageStorage` para o acesso a arquivos. O serviço de aplicação passa a apenas coordenar essas dependências.

### 3.2 SRP — `eventService.js` mistura regras de evento e sistema de arquivos

Arquivo: `eventhub-api/modules/event/eventService.js`, linhas 1–8 e 46–60.

```js
const fs = require('fs/promises')
const path = require('path')
const Event = require('./eventModel')
const User = require('../user/userModel')
const Like = require('../like/likeModel')
const Comment = require('../comment/commentModel')
const sequelize = require('../../config/database')
// ...
async function deleteEventImage(filename) {
  if (!filename) return
  const filePath = path.join(__dirname, '../../public/uploads/events', path.basename(filename))
  try { await fs.unlink(filePath) } catch (error) { /* ... */ }
}
```

O mesmo módulo trata regras de evento, consultas com contadores, transações e a remoção de imagens locais. A política de armazenamento das imagens pode mudar (por exemplo, para um serviço de objetos na nuvem) sem relação com as regras do evento.

**Refatoração:** extrair um `EventImageStorage` e deixar no caso de uso apenas a coordenação entre a persistência do evento e a limpeza da imagem.

### 3.3 DIP — `likeService.js` depende de Sequelize e de modelos concretos

Arquivo: `eventhub-api/modules/like/likeService.js`, linhas 8–11.

```js
const Like = require('./likeModel')
const Event = require('../event/eventModel')
const User = require('../user/userModel')
const sequelize = require('../../config/database')
```

A lógica de curtidas está presa à implementação Sequelize e à conexão configurada, e ainda importa modelos de outros módulos (`Event`, `User`). Isso dificulta testar as regras isoladamente e trocar a persistência.

**Refatoração:** definir contratos (`LikeRepository`, `EventRepository`, `TransactionManager`), com a implementação Sequelize em `infrastructure` e injetada no caso de uso. O ganho é a inversão da dependência; não é preciso criar abstrações para cada consulta simples.

### 3.4 SRP — `LikeButton.vue` combina interação, navegação, estado otimista e chamada HTTP

Arquivo: `eventhub-frontend/src/components/LikeButton.vue`, função `handleClick`, linhas 33–74.

```js
async function handleClick() {
  if (!authStore.isAuthenticated) {
    router.push({ path: '/login', query: { redirect: route.fullPath } })
    return
  }
  if (isBusy.value) return
  // ... guarda o estado anterior e atualiza a tela na hora
  try {
    const response = await toggleLike(props.eventId)
    const data = response.data?.data ?? response.data
    liked.value = data.liked
    count.value = data.likesCount
    emit('change', { liked: liked.value, likesCount: count.value })
  } catch (error) {
    // rollback e mensagem de erro
  }
}
```

O componente decide o redirecionamento ao login, mantém o estado otimista, chama o serviço, interpreta a resposta e faz o rollback. Isso o torna mais difícil de testar e de reaproveitar.

**Refatoração:** mover a operação otimista para um composable (`useLike(eventId)`) e deixar o componente responsável pela apresentação e pela emissão do evento `change`.

### 3.5 DIP — o serviço do frontend expõe a resposta do Axios

Arquivo: `eventhub-frontend/src/services/likeService.js`, linhas 3–9.

```js
export function toggleLike(eventId) {
  return api.post(`/events/${eventId}/toggle-like`)
}
export function getLikeStatus(eventId) {
  return api.get(`/events/${eventId}/like-status`)
}
```

Quem consome recebe a resposta completa do Axios e precisa conhecer sua estrutura, como `response.data?.data ?? response.data` em `LikeButton.vue`, linha 55. Não é uma violação isolada e automática do DIP, e sim uma oportunidade de separar transporte e aplicação.

**Refatoração:** normalizar a resposta no serviço e devolver dados da aplicação (`{ liked, likesCount }`), com um formato de erro consistente que não exponha o Axios ao componente.

### 3.6 SRP — controllers repetem validação de ID e construção de respostas

Arquivos: `eventhub-api/modules/like/likeController.js` (linhas 3–14) e `eventhub-api/modules/comment/commentController.js` (linhas 3–14).

```js
function parseId(value) {
  const id = Number(value)
  return Number.isSafeInteger(id) && id > 0 ? id : null
}

function invalidId(res) {
  return res.status(400).json({
    success: false,
    message: 'ID do evento inválido.',
    errors: []
  })
}
```

As duas funções aparecem idênticas nos dois controllers. A validação de ID e o formato da resposta HTTP são responsabilidades transversais, e o projeto já tem `middlewares/apiResponse.js` com os helpers `success` e `error` que fazem o mesmo trabalho de montar respostas.

**Refatoração:** criar um middleware reutilizável para validar parâmetros de ID e usar `apiResponse` em todos os controllers, deixando neles apenas a orquestração específica de cada rota.

### 3.7 OCP — a busca ainda é um stub (recomendação preventiva)

Arquivo: `eventhub-api/modules/search/searchServices.js`, linhas 1–3.

```js
function globalSearch(query) {
    return { events: [], users:[] };
}
```

A busca global ainda devolve listas vazias, então não há violação de OCP comprovada. Registra-se como prevenção: ao implementá-la, evitar concentrar filtros e estratégias de busca num único bloco condicional.

**Refatoração (quando a busca for implementada):** consultas ou estratégias separadas por tipo de entidade (eventos e usuários), compostas pelo serviço de busca.

## 4. Defeito encontrado fora do escopo SOLID

O arquivo `eventhub-api/modules/like/likeService.js` tem, nas linhas 2–6, código do frontend (`import api from './api'` e a função `getLikedEvents`) colado antes do código do backend. Num módulo CommonJS, a instrução `import` gera erro de sintaxe, e `node --check` falha nesse arquivo. Isso não é uma questão de design: é um defeito a corrigir, apagando essas cinco linhas.

## 5. Plano de refatoração priorizado

| Prioridade | Ação | Resultado esperado |
|---|---|---|
| 1 — Alta | Extrair o armazenamento de imagens de `eventService` e `userService` | Responsabilidades mais claras e testes mais simples |
| 2 — Alta | Normalizar respostas e erros dos serviços do frontend | Componentes menos dependentes do Axios |
| 3 — Alta | Centralizar a validação de IDs e usar `apiResponse` nos controllers | Menos duplicação |
| 4 — Média | Definir repositórios nos casos de uso com regras relevantes | Menor acoplamento com o Sequelize |
| 5 — Média | Organizar os módulos em domínio, aplicação, infraestrutura e apresentação | Limites de contexto explícitos |
| 6 — Futura | Adicionar Outbox, RabbitMQ e consumidores idempotentes | Processamento assíncrono confiável, quando necessário |
