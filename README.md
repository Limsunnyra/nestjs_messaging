# Messages API

A small REST API built with [NestJS](https://nestjs.com/) for creating and reading text messages. Messages are stored in a local JSON file (`messages.json`), so you don't need a database.

The project is a simple example of NestJS's layered design: **controller → service → repository**, connected by dependency injection.

## Features

- List all messages
- Get one message by ID
- Create a new message, with request body validation

## Getting started

### Requirements

- Node.js (a version supported by NestJS 12)
- npm

### Install

```bash
npm install
```

### Run

```bash
# development
npm run start

# watch mode (restarts on file changes)
npm run start:dev

# production
npm run build
npm run start:prod
```

The server listens on port `3000` by default. To use a different port, set the `PORT` environment variable:

```bash
PORT=4000 npm run start:dev
```

> **Note:** Start the server from the project root. The repository opens `messages.json` with a relative path, so it reads and writes that file in the directory the process was started from.

## API

Base URL: `http://localhost:3000`

| Method | Path            | Description               |
| ------ | --------------- | ------------------------- |
| GET    | `/messages`     | List all messages         |
| GET    | `/messages/:id` | Get a single message      |
| POST   | `/messages`     | Create a new message      |

### `GET /messages`

Returns every stored message as an object keyed by ID.

```json
{
  "613": { "id": 613, "content": "hello" },
  "830": { "id": 830, "content": "NestJs is cool" }
}
```

### `GET /messages/:id`

Returns one message.

```json
{ "id": 613, "content": "hello" }
```

If no message has that ID, the API responds with `404 Not Found`:

```json
{ "message": "message not found", "error": "Not Found", "statusCode": 404 }
```

### `POST /messages`

Creates a message. The body must be JSON with a string `content` field:

```json
{ "content": "Hello world" }
```

On success it returns `201 Created` with an empty body. If `content` is missing or isn't a string, the request fails validation and returns `400 Bad Request`.

### Trying the API

[request.http](request.http) has ready-made requests for each endpoint. You can run them with the VS Code [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) extension or a JetBrains IDE. You can also use curl:

```bash
curl http://localhost:3000/messages

curl -X POST http://localhost:3000/messages \
  -H "Content-Type: application/json" \
  -d '{"content": "Hello world"}'

curl http://localhost:3000/messages/613
```

## How it works

```
HTTP request
     │
     ▼
MessagesController   — routes requests, validates input, returns 404s
     │
     ▼
MessagesService      — business logic layer (currently passes calls through)
     │
     ▼
MessagesRepository   — reads and writes messages.json
     │
     ▼
messages.json
```

### Project structure

```
src/
├── main.ts                          # App entry point
└── messages/
    ├── messages.module.ts           # Registers the controller and providers
    ├── messages.controller.ts       # HTTP routes
    ├── messages.service.ts          # Service layer
    ├── messages.repository.ts       # File-based data access
    └── dtos/
        └── create-message.dto.ts    # Validation rules for POST bodies
messages.json                        # Data store
request.http                         # Example requests
```

### Startup — [src/main.ts](src/main.ts)

`main.ts` builds the app from `MessagesModule` and registers a global `ValidationPipe`. That pipe checks every incoming request body against its DTO class before the request reaches the controller.

### Module — [src/messages/messages.module.ts](src/messages/messages.module.ts)

`MessagesModule` declares `MessagesController` and registers `MessagesService` and `MessagesRepository` as providers. Nest's dependency injection container creates these and passes them into constructors automatically:

- The controller asks for a `MessagesService` in its constructor.
- The service asks for a `MessagesRepository` in its constructor.

No class creates its own dependencies with `new`. Because of this, you can swap one layer out (for example, replace the file repository with a database repository) or mock it in tests.

### Validation — [src/messages/dtos/create-message.dto.ts](src/messages/dtos/create-message.dto.ts)

`CreateMessageDto` uses the `@IsString()` decorator from `class-validator`. When a `POST /messages` request comes in, the global `ValidationPipe` turns the JSON body into a `CreateMessageDto` and checks it. Invalid bodies are rejected with a `400` before the controller method runs.

### Controller — [src/messages/messages.controller.ts](src/messages/messages.controller.ts)

The controller maps routes under `/messages` to service calls. For `GET /messages/:id`, it throws a `NotFoundException` when the service returns nothing. Nest turns that exception into a `404` response.

### Service — [src/messages/messages.service.ts](src/messages/messages.service.ts)

The service sits between the controller and the repository. Right now it just forwards calls. Business rules belong here as the app grows.

### Repository — [src/messages/messages.repository.ts](src/messages/messages.repository.ts)

The repository is the only part of the app that touches storage. Each method reads `messages.json` and parses it:

- `findAll()` returns the whole object.
- `findOne(id)` returns `messages[id]`, or `undefined` if that ID doesn't exist.
- `create(content)` makes a random ID from 0 to 998, adds `{ id, content }` to the object, and writes the file back.

## Known limitations

This is a learning project, and the storage layer is intentionally simple:

- **ID collisions:** IDs are random numbers below 999. A new message can get an ID that's already in use and overwrite the old message.
- **No concurrency safety:** each write reads the whole file and then writes it back. Two requests at the same moment can overwrite each other's changes.
- **POST returns nothing:** the created message and its ID aren't sent back to the client.
- **No update or delete endpoints.**

## Scripts

| Command              | Description                     |
| -------------------- | ------------------------------- |
| `npm run start`      | Start the app                   |
| `npm run start:dev`  | Start in watch mode             |
| `npm run build`      | Compile to `dist/`              |
| `npm run start:prod` | Run the compiled build          |
| `npm run lint`       | Lint with oxlint                |
| `npm run format`     | Format with Prettier            |
| `npm run test`       | Run unit tests                  |
| `npm run test:e2e`   | Run end-to-end tests            |
| `npm run test:cov`   | Run tests with coverage report  |
