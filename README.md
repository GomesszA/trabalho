# Desenvolvimento de API com Persistência e Arquitetura Distribuída

API RESTful de **Usuários** (CRUD completo), com persistência em **MongoDB**, atrás de um **API Gateway (Nginx)** com rate limiting e load balancing entre duas instâncias, e com um sistema de **mensageria (RabbitMQ)** com duas filas distintas.

## Arquitetura

```
                        ┌───────────────┐
   cliente ───────────▶ │  Gateway      │  (Nginx: rate limit + load balancing)
                        │  :8080        │
                        └───────┬───────┘
                     ┌──────────┴──────────┐
                     ▼                     ▼
              ┌────────────┐        ┌────────────┐
              │   api-1    │        │   api-2    │   (Node.js + Express)
              │   :3000    │        │   :3000    │
              └──────┬─────┘        └──────┬─────┘
                     │                     │
          ┌──────────┴──────────┬──────────┴──────────┐
          ▼                     ▼                      ▼
   ┌────────────┐       ┌───────────────┐     (cada instância também
   │  MongoDB   │       │   RabbitMQ    │      consome as filas abaixo)
   │  :27017    │       │  :5672/:15672 │
   └────────────┘       └───────┬───────┘
                     ┌───────────┴───────────┐
                     ▼                       ▼
           fila.notificacoes         fila.auditoria
        (ex: usuário criado)     (log de create/update/delete)
```

## Como subir o projeto

Pré-requisito: Docker e Docker Compose instalados.

```bash
docker compose up --build
```

Isso sobe: MongoDB, RabbitMQ (com painel de management), as duas instâncias da API (`api-1` e `api-2`) e o Gateway Nginx na frente delas.

- **Gateway (ponto de entrada da API):** http://localhost:8080
- **Painel do RabbitMQ:** http://localhost:15672 (usuário/senha: `guest` / `guest`)
- **MongoDB:** exposto em `localhost:27017` (opcional, útil para inspecionar dados com um client)

Não é necessário nenhum passo manual além do `docker compose up`.

## Endpoints (via Gateway, porta 8080)

| Método | Rota            | Descrição                    |
|--------|-----------------|-------------------------------|
| POST   | `/usuarios`     | Cria um usuário               |
| GET    | `/usuarios`     | Lista todos os usuários       |
| GET    | `/usuarios/:id` | Busca um usuário por id       |
| PUT    | `/usuarios/:id` | Atualiza um usuário           |
| DELETE | `/usuarios/:id` | Remove um usuário             |
| GET    | `/health`       | Healthcheck (mostra qual instância respondeu) |

### Exemplo — criar usuário

```bash
curl -X POST http://localhost:8080/usuarios \
  -H "Content-Type: application/json" \
  -d '{"nome": "Maria Silva", "email": "maria@exemplo.com", "senha": "123456"}'
```

### Exemplo — verificar load balancing

Rode o `/health` várias vezes seguidas e observe o campo `instancia` (ou o header `X-Instance-Id`) alternando entre `api-1` e `api-2`:

```bash
for i in {1..6}; do curl -s http://localhost:8080/health; echo; done
```

### Exemplo — verificar rate limiting

O Nginx está configurado para ~5 requisições/segundo por IP (com burst de 10). Disparando muitas requisições rapidamente, você verá respostas `429 Too Many Requests`:

```bash
for i in {1..30}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health; done
```

## Mensageria (RabbitMQ)

Toda operação de escrita na API dispara eventos assíncronos:

- **`fila.notificacoes`**: publicada quando um usuário é criado. Um consumer simula o envio de uma notificação (ex: e-mail de boas-vindas).
- **`fila.auditoria`**: publicada em toda operação de `CREATE`, `UPDATE` ou `DELETE`. Um consumer registra um log de auditoria no console.

Os consumers rodam dentro de cada instância da API (`api-1` e `api-2`), então o RabbitMQ distribui as mensagens entre elas (competing consumers) — dá pra ver isso nos logs de cada container:

```bash
docker compose logs -f api-1 api-2
```

Você também pode acompanhar as filas e mensagens pelo painel de management do RabbitMQ (http://localhost:15672).

## Estrutura de pastas

```
.
├── docker-compose.yml
├── api/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── app.js               # configuração do Express
│       ├── server.js            # bootstrap: conecta Mongo, RabbitMQ, sobe consumers e o servidor HTTP
│       ├── config/
│       │   ├── db.js            # conexão com MongoDB (com retry)
│       │   └── rabbitmq.js      # conexão com RabbitMQ + declaração das filas
│       ├── models/
│       │   └── User.js          # schema Mongoose
│       ├── controllers/
│       │   └── userController.js
│       ├── routes/
│       │   └── userRoutes.js
│       └── rabbitmq/
│           ├── producer.js
│           └── consumers/
│               ├── notificacoesConsumer.js
│               └── auditoriaConsumer.js
└── gateway/
    ├── Dockerfile
    └── nginx.conf                # rate limit + load balancing (upstream api-1/api-2)
```

## Possíveis extensões futuras

- Adicionar autenticação (JWT) nas rotas.
- Adicionar telemetria/monitoramento com Grafana + Loki + Promtail (terceira opção do trabalho, caso queiram evoluir o projeto).
- Testes automatizados (Jest + Supertest).
