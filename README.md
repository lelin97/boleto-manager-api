# Boleto Manager — API

> Technical challenge built for a job application process. Shared here as a code sample.

REST API for managing **boletos** (Brazilian bank payment slips): create, list, update and delete them, and mark one or many as paid, with late fees calculated automatically. It's consumed by the front-end in [boleto-manager-web](https://github.com/lelin97/boleto-manager-web).

## The challenge

Build a full CRUD for boletos, with a front-end and a back-end. Each boleto has a description, due date, amount, late fee and interest, and can be marked as paid ("baixa", in Brazilian finance terms).

## What was delivered

- **CRUD endpoints** for boletos, with request validation using **Zod**
- **Bulk payment:** mark several boletos as paid in a single request
- **Automatic late fee on payment:** if a boleto is paid after its due date, the fee is calculated from the interest rate and the number of days overdue, and the total amount is updated
- Duplicate check on creation (by description)
- A consistent response shape for every endpoint and proper `400` / `500` status codes
- Structured logging with **Pino**
- A **TypeORM migration** that seeds the database with sample data

## Tech stack

Node.js 18 · TypeScript · Express · TypeORM · PostgreSQL · Zod · Pino · date-fns

## Getting started

**Requirements:** Node.js 18 (see `.nvmrc`), yarn and Docker.

**1. Start a PostgreSQL container**

```bash
docker run --name ct-db -p 5432:5432 -e POSTGRES_USER=user-ct -e POSTGRES_PASSWORD=12345678 -e POSTGRES_DB=ctech-db -d postgres
```

**2. Configure the environment**

Copy `env.example` to `.env`. The default values already match the container above:

```
DB_HOST=localhost
DB_PORT=5432
DB_USER=user-ct
DB_PASSWORD=12345678
DB_NAME=ctech-db
```

**3. Install and run**

```bash
yarn
yarn start
```

The API runs on `http://localhost:4050`. To seed the database with sample boletos, run `yarn migration:run` and start the API again.

## Endpoints

All routes are under `/boletos`.

| Method | Route | Description |
| ------ | ----- | ----------- |
| `GET` | `/listar-boleto` | List all boletos |
| `POST` | `/criar-boleto` | Create a boleto |
| `PUT` | `/atualizar-boleto/:id_boleto` | Update a boleto |
| `PUT` | `/baixa-boleto/:usuario_cadastro` | Mark boletos as paid (body: array of boleto IDs) |
| `DELETE` | `/deletar-boleto/:id_boleto` | Delete a boleto |

Request body for create and update:

```json
{
  "descricao": "Internet bill",
  "dt_venc": "2024-07-10",
  "valor": 120,
  "valor_multa": 2,
  "juros": 1
}
```

Every response follows the same shape:

```json
{ "sucesso": true, "mensagem": "...", "data": [] }
{ "sucesso": false, "mensagem": "...", "erro": "..." }
```

## Project structure

- **`src/routes`** — Express routers; each route validates input with Zod and delegates to a service
- **`src/services/Boletos`** — One class per use case (create, list, update, delete, mark as paid), each with an `executar` method
- **`src/models`** — TypeORM entity for the `boletos` table
- **`src/zod`** — Validation schemas for request bodies and params
- **`src/migrations`** — Seed data
- **`src/types`** — The `RetornoDB` response contract shared by all services

## Design decisions

- **One service per use case:** routes only handle HTTP concerns (validation, status codes, logging), while business rules live in small, focused service classes.
- **Validation at the edge:** every body and route param is parsed with Zod before reaching a service, so invalid input never touches the database.
- **Late fees calculated at payment time:** the fee depends on how many days have passed since the due date, so it's computed when the boleto is actually paid rather than stored upfront.

## With more time, I would add

- Unit tests for the services and integration tests for the routes
- Await each update inside the bulk-payment transaction (it currently runs them in a `forEach` with async callbacks), so the transaction truly covers all of them
- Authentication, replacing the fixed user ID used when creating boletos
- A `docker-compose.yml` to start the database and the API with a single command
