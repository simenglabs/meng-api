# meng-api design (draft for review)

A web framework for Meng in the spirit of FastAPI: typed handlers, routing, route
groups, startup/shutdown hooks, and generated OpenAPI docs with a Swagger UI page.
Installed with `meng add meng-api:<version>` and imported with `use "meng-api";`.

Status: **design only**. It needs compiler features that do not exist yet (listed
below); the package is written once they land.

## What using it looks like

```meng
use "meng-api";

struct NewUser ->
    name: str;
    email: str;
<-;

struct User ->
    id: int;
    name: str;
    email: str;
<-;

fnc Startup(app: App): error ->
    show("connecting to the database");
    return none;
<-;

fnc Shutdown(app: App) ->
    show("bye");
<-;

% A handler gets the request and returns a response or an error.
  Path parameters, query strings, and JSON bodies are read through helpers. %
fnc GetUser(req: Request): Response, error ->
    set id: int, err: error = PathInt(req, "id");
    if err != none ->
        return none, err;                    % becomes 400 invalid_input %
    <-;
    if id != 1 ->
        return none, error(not_found, "user " + str(id) + " not found");   % 404 %
    <-;
    return Json(200, User(id: 1, name: "Dara", email: "dara@example.com")), none;
<-;

fnc CreateUser(req: Request): Response, error ->
    set body: NewUser, err: error = from_json(req.body);    % 400 on bad JSON %
    if err != none ->
        return none, err;
    <-;
    return Json(201, User(id: 2, name: body.name, email: body.email)), none;
<-;

set app: App = NewApp(title: "Users API", version: "1.0.0");
OnStartup(app, Startup);
OnShutdown(app, Shutdown);

set users: Group = NewGroup(app, "/users", tag: "users");
Get(users, "/{id}", GetUser, returns: schema(User));
Post(users, "/", CreateUser, body: schema(NewUser), returns: schema(User));

set err: error = Serve(app, port: 8080);    % docs at /docs, spec at /openapi.json %
if err != none ->
    show(err);
    return;
<-;
```

## Features

| Feature | How |
|---|---|
| Routing | `Get` / `Post` / `Put` / `Patch` / `Delete(target, path, handler, ...)`; path parameters `{name}`; method + path matching with a small trie |
| Route groups | `NewGroup(app, prefix, tag:)`, nested groups, per-group hooks (middleware) |
| Hooks | `OnStartup(app, F)` (an error stops startup), `OnShutdown(app, F)` (runs on SIGTERM / SIGINT after in-flight requests finish), `Before(target, F)` / `After(target, F)` per request |
| Errors to HTTP | the error kind picks the status: `invalid_input` 400, `unauthorized` 401, `not_found` 404, `conflict` 409, `timeout` 504, anything else 500; the JSON body is `{"error": kind, "message": message}` (internal details are not sent for 500) |
| OpenAPI | `/openapi.json` built from the routes, their `body:` / `returns:` schemas, path parameters, and tags |
| Swagger UI | `/docs` serves a page that loads Swagger UI (from a CDN) pointed at `/openapi.json` |
| Concurrency | every request runs in its own task (plan 3); a runtime error fails that request with 500 |

## Compiler features it needs

1. **Function references** (`fnc(Request): Response, error` as a parameter type, so
   handlers and hooks can be registered). Plain function pointers: no closures.
2. **JSON built-ins driven by types**: `to_json(value): str` for any value, and
   `from_json(text)`, which returns `T, error` where `T` comes from the variable it is
   stored in (like `channel(n)`). Both use the type descriptors that already exist.
3. **`schema(T)`**: the JSON Schema of a type as a `str`, for OpenAPI.
4. **Networking in the standard library**: TCP listen / accept / read / write on the
   task scheduler (plan 3) with epoll / kqueue, plus signals for graceful shutdown.
5. **HTTP/1.1 parsing**: written in Meng inside meng-api on top of (4), or provided by
   the standard library (decision when milestone 4 starts).
6. **`init` functions** (requested separately): a file may declare `fnc init() -> ... <-;`;
   it runs before the program's top-level statements, dependencies first.

## Order of work

1. Compiler: finish plan 3 (tasks, channels, threads).
2. Compiler: `init` functions and function references (one small plan).
3. Compiler: JSON built-ins and `schema(T)`.
4. Compiler: standard library networking and signals.
5. meng-api 0.1.0: routing, groups, hooks, error mapping, OpenAPI, Swagger UI, tests;
   tag `v0.1.0`; it is already listed in the registry (`meng-registry/index.meow`).

## Open questions for review

- Is the registration style (`Get(users, "/{id}", GetUser, returns: schema(User))`) what
  you want, or should routes be declared with an annotation above the function, for
  example `@get("/users/{id}")`? Annotations need more compiler work but read closer to
  FastAPI.
- Function names: Meng functions use `PascalCase` today; the package follows that.
