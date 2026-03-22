# SKILL 04-D — NestJS: Modules, Providers, Guards, and Interceptors
Category: Backend Development
Applies to: Node.js, NestJS, TypeScript

## What this skill covers
NestJS is a structured Node.js framework built around Angular-style modules, decorators, and dependency injection. This skill covers how to correctly organize a NestJS application: feature modules, provider scoping, request lifecycle, guards for authorization, and interceptors for cross-cutting concerns like logging and response transformation. It is the definitive reference for any developer joining or starting a NestJS project.

## When to activate this skill
- Starting a new NestJS application
- Adding authentication guards or role-based access control
- Adding cross-cutting concerns: logging, caching, request transformation
- Reviewing or refactoring a NestJS codebase for modularity

## Core principles
1. **Feature modules are the unit of organization.** Every domain (users, orders, payments) is a standalone module that imports only what it needs.
2. **Providers are singletons by default.** `@Injectable()` providers are shared across the module graph. Use `REQUEST` scope only when you need per-request state.
3. **Guards run before the route handler.** Use guards for authentication and authorization checks. Guards return `true` to allow or throw `ForbiddenException`/`UnauthorizedException`.
4. **Interceptors wrap the execution chain.** Use interceptors to transform responses, measure timing, or add caching — not for auth logic.
5. **Pipes validate and transform input.** Use `ValidationPipe` globally to validate all DTOs and strip unknown properties.

## Step-by-step guide
1. **Structure by feature module:**
   ```
   src/
     users/
       users.module.ts
       users.controller.ts
       users.service.ts
       dto/
         create-user.dto.ts
       entities/
         user.entity.ts
     auth/
       auth.module.ts
       auth.service.ts
       guards/
         jwt-auth.guard.ts
   app.module.ts
   main.ts
   ```
2. **Register `ValidationPipe` globally in `main.ts`:**
   ```ts
   app.useGlobalPipes(new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true }));
   ```
3. **Create DTOs with `class-validator` decorators** (`@IsEmail()`, `@IsNotEmpty()`, etc.) — these are what `ValidationPipe` validates.
4. **Create a `JwtAuthGuard` extending `AuthGuard('jwt')`.** Apply globally via `APP_GUARD` provider in `AppModule`, then skip specific routes with a `@Public()` decorator and `Reflector`.
5. **Create a `RolesGuard` checking `@Roles()` decorator via `Reflector`.** Apply after `JwtAuthGuard` in the guard order.
6. **Create a logging interceptor** implementing `NestInterceptor` using `Observable` and `tap()` — logs request duration without modifying the response.
7. **Use `ConfigModule.forRoot({ isGlobal: true, validate })` with Joi or Zod for env validation.**

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Auth logic in a controller method | `JwtAuthGuard` applied at controller or route level via decorator |
| Global `try/catch` in every service method | `ExceptionFilter` for global error handling; domain exceptions extend `HttpException` |
| `new UserService()` inside a controller | Inject via constructor: `constructor(private readonly userService: UserService)` |
| `module.exports` CommonJS patterns | TypeScript `@Module()` with `imports`, `controllers`, `providers`, `exports` |
| `REQUEST` scope on every provider | `REQUEST` scope only when strictly needed — it disables singleton caching and creates a new instance per request |

## Stack-specific notes
**NestJS with Prisma:** Create a `PrismaService` extending `PrismaClient` and implementing `OnModuleInit` to call `$connect()`. Export from `DatabaseModule` and import wherever needed.
**NestJS with TypeORM:** Use `TypeOrmModule.forFeature([UserEntity])` in the feature module to register the entity. Inject `@InjectRepository(UserEntity)` in the service.
**Comparison to Express/Fastify:** NestJS uses Express or Fastify as the HTTP adapter underneath. It adds DI, module system, and decorator-based routing but the underlying request/response objects are still Express/Fastify types.

## Common mistakes
1. **Circular module dependencies.** Use `forwardRef(() => ModuleA)` and `@Inject(forwardRef(() => ServiceA))` to break the cycle. But also investigate whether the circular dependency reveals a design problem.
2. **Exporting providers that should be private.** Only export providers that other modules need. The default is private — add to `exports: []` explicitly.
3. **`REQUEST` scope on a service used by many.** `REQUEST`-scoped providers create a new instance per HTTP request — all providers that depend on them also become `REQUEST`-scoped, degrading performance.
4. **Skipping `whitelist: true` in `ValidationPipe`.** Without `whitelist`, unknown properties pass through to the service. Always set both `whitelist: true` and `forbidNonWhitelisted: true`.

## Checklist
- [ ] Every domain has its own feature module
- [ ] `ValidationPipe` with `whitelist: true` registered globally
- [ ] DTOs use `class-validator` decorators for all fields
- [ ] `JwtAuthGuard` applied globally; `@Public()` decorator for public routes
- [ ] `RolesGuard` checks `@Roles()` metadata via `Reflector`
- [ ] Config validated at startup via `ConfigModule` with schema
- [ ] Interceptors used for logging/response transformation only (not auth)
- [ ] Exception filters handle domain errors and return consistent response shapes
- [ ] No circular module dependencies (or resolved explicitly with `forwardRef`)
- [ ] Providers use default singleton scope unless per-request state is needed
