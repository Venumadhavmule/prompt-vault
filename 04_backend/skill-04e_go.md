# SKILL 04-E — Go Backend: Project Layout and Error Handling
Category: Backend Development
Applies to: Go, net/http, chi, Echo, Gin

## What this skill covers
Go's standard library is powerful enough to build production APIs, but conventions matter — Go has no opinionated framework forcing structure. This skill covers the standard Go project layout (`cmd/`, `internal/`, `pkg/`), idiomatic error handling with typed errors (not error codes), dependency injection without a framework, and the HTTP patterns needed for a clean, maintainable Go service.

## When to activate this skill
- Starting a new Go backend service
- Choosing a Go HTTP framework/router
- Implementing error handling that avoids code duplication
- Reviewing Go code for idiomatic patterns

## Core principles
1. **Errors are values, not exceptions.** Every function that can fail returns `(result, error)`. Check every error at the call site — never ignore with `_`.
2. **Wrap errors with context.** Use `fmt.Errorf("doing X: %w", err)` to add context while preserving the sentinel error for `errors.Is()` / `errors.As()` checks.
3. **`internal/` enforces encapsulation.** Code in `internal/` can only be imported by packages rooted at the same parent. Use it for all domain code.
4. **Prefer `net/http` or a thin router (`chi`) over full frameworks.** Go's standard library handles most needs. `chi` adds routing with middleware chaining. Gin/Echo add more but have more magic.
5. **Inject dependencies via struct fields.** No global state, no `init()` side effects. Construct all dependencies in `main()` and pass them down.

## Step-by-step guide
1. **Standard project layout:**
   ```
   cmd/
     api/
       main.go           ← entry point: wire dependencies, start server
   internal/
     handler/            ← HTTP handlers (parse request, call service, write response)
     service/            ← business logic
     repository/         ← database queries
     domain/             ← types, interfaces, domain errors
     middleware/         ← auth, logging, recovery
   pkg/                  ← reusable, importable by external packages (if needed)
   ```
2. **Define domain errors in `internal/domain/errors.go`:**
   ```go
   type NotFoundError struct { Resource string }
   func (e *NotFoundError) Error() string { return e.Resource + " not found" }
   ```
3. **Map domain errors to HTTP codes in the handler layer only:**
   ```go
   var notFound *domain.NotFoundError
   if errors.As(err, &notFound) { w.WriteHeader(http.StatusNotFound); ... }
   ```
4. **Wire dependencies in `main.go`:**
   ```go
   db := postgres.Connect(cfg.DatabaseURL)
   userRepo := repository.NewUserRepo(db)
   userService := service.NewUserService(userRepo)
   userHandler := handler.NewUserHandler(userService)
   ```
5. **Use `chi` for routing with middleware:** `r := chi.NewRouter()`, `r.Use(middleware.Logger)`, `r.Mount("/users", userHandler.Routes())`.
6. **Graceful shutdown:** `http.Server` with `Shutdown(ctx)` on `SIGTERM`.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `if err != nil { log.Fatal(err) }` in a service | Return `fmt.Errorf("creating user: %w", err)` and let the caller decide |
| Global `var db *sql.DB` | Inject `*sql.DB` via struct constructor |
| `errors.New("not found")` string comparison | Typed sentinel error / typed struct error for `errors.Is` / `errors.As` |
| All code in `main.go` | `cmd/api/main.go` is wiring only; domain code in `internal/` |
| Ignoring the second return value: `result, _ := service.Call()` | Always check and handle the error |

## Stack-specific notes
**net/http:** sufficient for simple services. Manual mux with `http.HandleFunc`. Add middleware as handler wrappers.
**chi:** idiomatic middleware chaining, sub-routers, context-based parameter reading. Recommended for most new services.
**Gin/Echo:** more opinionated, framework-specific patterns. Use if the team is already familiar; otherwise chi + stdlib keeps code dependency-light.
**Deployment:** Go compiles to a single static binary. Docker images can `FROM scratch` or `FROM gcr.io/distroless/static`. No runtime dependencies.

## Common mistakes
1. **Swallowing errors.** `result, _ := fn()` hides bugs. Even if you can't handle an error, log it and propagate it up.
2. **Calling `log.Fatal()` inside a library or service.** `log.Fatal` calls `os.Exit(1)` which bypasses deferred cleanup. Only call in `main()` during init. Services should return errors.
3. **Not setting read/write timeouts on `http.Server`.** Default Go `http.Server` has no timeouts — a slow client can hold a connection open indefinitely. Always set `ReadTimeout`, `WriteTimeout`, `IdleTimeout`.
4. **Storing request-scoped data in a package-level variable.** Use `context.WithValue(ctx, key, val)` for per-request storage, passed through the call chain.

## Checklist
- [ ] All code organized under `cmd/`, `internal/`, `pkg/` layout
- [ ] Domain errors defined as typed structs in `internal/domain/`
- [ ] HTTP-to-domain error mapping only in handler layer
- [ ] All dependencies injected via constructors — no package-level global state
- [ ] `http.Server` configured with `ReadTimeout`, `WriteTimeout`, `IdleTimeout`
- [ ] Graceful shutdown on `SIGTERM` using `server.Shutdown(ctx)`
- [ ] Every error returned with `%w` wrapping for context
- [ ] No `log.Fatal()` outside of `main()`
- [ ] `errors.Is()` / `errors.As()` used for error type checking (not string comparison)
- [ ] All handler parameters validated before passing to service layer
