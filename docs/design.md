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

% Globals are set up at startup, before annotations run, and are frozen afterwards. %
global app: App = NewApp(title: "Users API", version: "1.0.0");
global users: Group = NewGroup(app, "/users", tag: "users");

% A handler gets the request and returns a response or an error.
  Path parameters, query strings, and JSON bodies are read through helpers.
  The annotation registers it on the `users` group. %
@route.get(users, "/{id}", returns: schema(User))
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

@route.post(users, "/", body: schema(NewUser), returns: schema(User))
fnc CreateUser(req: Request): Response, error ->
    set body: NewUser, err: error = from_json(req.body);    % 400 on bad JSON %
    if err != none ->
        return none, err;
    <-;
    return Json(201, User(id: 2, name: body.name, email: body.email)), none;
<-;

@hook.startup(app)
fnc Startup(keep a: App): error ->
    show("connecting to the database");
    return none;
<-;

@hook.shutdown(app)
fnc Shutdown(keep a: App) ->
    show("bye");
<-;

set err: error = Serve(app, port: 8080);    % docs at /docs, spec at /openapi.json %
if err != none ->
    show(err);
    return;
<-;
```

## Features

| Feature | How |
|---|---|
| Routing | annotations `@route.get` / `post` / `put` / `patch` / `delete(target, path, body:, returns:)` above a handler; path parameters `{name}`; method + path matching with a small trie |
| Route groups | `NewGroup(app, prefix, tag:)`, nested groups, per-group hooks (middleware) |
| Hooks | `@hook.startup(app)` (an error stops startup), `@hook.shutdown(app)` (runs on SIGTERM / SIGINT after in-flight requests finish), `@hook.before(target)` / `@hook.after(target)` per request |
| Errors to HTTP | the error kind picks the status: `invalid_input` 400, `unauthorized` 401, `not_found` 404, `conflict` 409, `timeout` 504, anything else 500; the JSON body is `{"error": kind, "message": message}` (internal details are not sent for 500) |
| OpenAPI | `/openapi.json` built from the routes, their `body:` / `returns:` schemas, path parameters, and tags |
| Swagger UI | `/docs` serves a page that loads Swagger UI (from a CDN) pointed at `/openapi.json` |
| Concurrency | every request runs in its own task (plan 3); a runtime error fails that request with 500 |

## Compiler features it needs

1. **Function references** (`fnc(Request): Response, error` as a parameter type, so
   handlers and hooks can be registered). Plain function pointers: no closures.
   **Done in the compiler (plan 4).**
2. **Dotted function names and annotations.** A package may declare `fnc route.get(...)`
   (a dotted name, callable as `route.get(...)`). An annotation `@route.get(users, "/{id}")`
   on the line above `fnc GetUser` means: call `route.get(users, "/{id}", GetUser)` (the
   annotated function is passed as the last argument) during the program's startup phase,
   after the file's globals are set and before its `init`. So `users` must be a `global`.
   **Done in the compiler (plan 4).**
3. **JSON built-ins driven by types**: `to_json(value): str` for any value, and
   `from_json(text)`, which returns `T, error` where `T` comes from the variable it is
   stored in (like `channel(n)`). Both use the type descriptors that already exist.
4. **`schema(T)`**: the JSON Schema of a type as a `str`, for OpenAPI.
5. **Networking in the standard library**: TCP listen / accept / read / write on the
   task scheduler (plan 3) with epoll / kqueue, plus signals for graceful shutdown.
6. **HTTP/1.1 parsing**: written in Meng inside meng-api on top of (5), or provided by
   the standard library (decision when milestone 4 starts).
7. **`init` functions and globals** (requested separately): a file may declare `fnc init() -> ... <-;`
   and `global` variables; they run in the startup phase, dependencies first. Globals are
   private to their file and frozen after startup (safe to read from every request task).
   No task can be spawned during startup, so request workers start in `Serve`.
   **Done in the compiler (plan 4).**

## Order of work

1. Compiler: finish plan 3 (tasks, channels, threads).
2. Compiler: `init` functions, function references, dotted names, and annotations (one plan).
3. Compiler: JSON built-ins and `schema(T)`.
4. Compiler: standard library networking and signals.
5. meng-api 0.1.0: routing, groups, hooks, error mapping, OpenAPI, Swagger UI, tests;
   tag `v0.1.0`; it is already listed in the registry (`meng-registry/index.meow`).

## Decisions

- Routes and hooks are declared with annotations above the function
  (`@route.get(users, "/{id}", returns: schema(User))`), decided by the user.
- Function names: Meng functions use `PascalCase`; namespaced package functions use
  lowercase dotted names (`route.get`, `hook.startup`).
