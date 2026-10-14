---
name: jrk-coding-stack
description: Designs, migrates, and reviews applications using Laravel, PostgreSQL, Inertia.js, React TypeScript, and styled-components. Use it to build a full-stack application, replace Next.js or Supabase, preserve functional parity, structure a typed Laravel backend, build development and production Docker images, and publish to GHCR.
---

# JRK Coding Stack

## Objective

Build modern monolithic applications with:

- Laravel for the backend, domain logic, authentication, and data access;
- PostgreSQL as the database;
- Inertia.js as the HTTP boundary between Laravel and React;
- React without an additional frontend framework, using strict TypeScript;
- styled-components for all presentation;
- Docker Compose for development and production;
- a multi-stage Docker image published to GHCR by GitHub Actions.

Favor an explicit, typed, and pragmatic architecture. During a migration, preserve behavior and design before improving the architecture.

## Non-negotiable principles

1. Laravel is the only backend.
2. React contains no server-side business rules.
3. Inertia transports serializable props and orchestrates navigation.
4. PostgreSQL is accessed through Laravel. Keep no Supabase client, Supabase RLS, Supabase RPC, or `@supabase/*` dependency.
5. Controllers are thin HTTP adapters.
6. HTTP inputs are validated by `FormRequest` classes.
7. Use cases receive `readonly` DTOs, not `Request`, `UploadedFile`, or implicit arrays.
8. External integrations go through Application ports and Infrastructure adapters.
9. Secrets are read through Laravel configuration, never directly with `env()` outside `config/` files.
10. A stack migration preserves routes, design, copy, HTTP statuses, redirects, cookies, sessions, and features.

## Laravel architecture

Use this structure:

```text
app/
├── Application/
│   └── <Domain>/
│       ├── Actions/
│       ├── Contracts/
│       ├── Data/
│       ├── Presenters/
│       └── Queries/
├── Domain/
│   └── <Domain>/
├── Http/
│   ├── Adapters/
│   ├── Controllers/
│   ├── Middleware/
│   └── Requests/
├── Infrastructure/
│   └── <Domain>/
├── Models/
└── Providers/
```

### Controllers

A controller may:

- authorize the action;
- receive a `FormRequest`;
- call a Query or Action;
- return an Inertia response, JSON response, or redirect.

A controller must not:

- call `DB::`;
- open a transaction;
- contain a business rule;
- normalize a large payload;
- call Slack, n8n, or another API directly;
- build a large props tree.

Move HTTP adaptation to `FormRequest::payload()`. Move projections to a Query or Presenter.

### Actions and Queries

- An Action represents a business mutation.
- A Query represents a read operation.
- A mutation involving multiple writes uses a transaction.
- Concurrent validations are repeated under a lock within the transaction.
- Network calls must not remain under a PostgreSQL lock unless an atomicity requirement is documented and verified.
- External side effects are triggered after commit whenever the business rules allow it.

### DTOs and enums

- Use `final readonly` classes.
- Use backed enums for persisted closed vocabularies.
- Add the corresponding Eloquent casts.
- Precisely document any array shapes that remain necessary with PHPStan.
- Reject impossible states in the DTO constructor or factory.
- Never make Application depend on `Illuminate\Http`.

### Ports and adapters

Create an interface in `Application/<Domain>/Contracts` for:

- external APIs;
- file storage;
- signing or encryption;
- specialized persistence that represents a real boundary;
- infrastructure that may have multiple implementations.

Implement the port in `Infrastructure/<Domain>` and declare the binding in `AppServiceProvider`.

Do not create a generic repository for every model. Eloquent remains acceptable in simple Actions and Queries.

### Eloquent and PostgreSQL

- Laravel migrations are the source of truth for the schema.
- Every schema change uses a new migration.
- Use `timestamptz` for instants.
- Explicitly declare `$fillable` or `$guarded`.
- Use PostgreSQL constraints for integrity, not to hide business rules.
- Avoid N+1 queries through eager loading or grouped reads.
- Paginate in SQL instead of loading an entire dataset.
- Do not serialize sensitive data into Inertia props or audit logs.

## React frontend with Inertia

Use React TypeScript without Next.js, Remix, or another frontend framework.

```text
resources/js/
├── Components/
│   └── MyComponent/
│       ├── MyComponent.tsx
│       └── styled.ts
└── Pages/
    └── MyPage/
        ├── MyPage.tsx
        └── styled.ts
```

### Component conventions

For `MyComponent`:

```tsx
import * as S from './styled';

export function MyComponent() {
    return (
        <S.MyComponentContainer>
            <S.Title>Content</S.Title>
        </S.MyComponentContainer>
    );
}
```

In `styled.ts`:

```tsx
import styled from 'styled-components';

export const MyComponentContainer = styled.section`
    display: grid;
    gap: 1rem;
`;

export const Title = styled.h2`
    margin: 0;
`;
```

Rules:

- always use `import * as S from './styled';`;
- the root node is `S.<ComponentName>Container`;
- use no raw HTML elements in JSX: use a styled component, including for `div`, `span`, `button`, `summary`, and native form fields;
- use no `className` or `style` prop;
- use no `index.ts` file;
- use one folder per component;
- keep each component at no more than 250 lines;
- extract subcomponents into subfolders;
- write every styled component over multiple lines;
- do not condense CSS declarations onto one line;
- keep TypeScript strict and validate unknown network responses before use.

### Inertia

- Pages receive typed props.
- Large props are prepared on the server.
- Use Inertia navigation and forms for application flows.
- Reserve `fetch` for cases that genuinely require JSON responses.
- Never expose an entire Eloquent model when an explicit projection is sufficient.
- Never send secrets or unnecessary personal data to the browser.

## Authentication and authorization

- Laravel owns the application session.
- An SSO integration such as Crossguard uses Socialite or a dedicated provider.
- Read URLs, identifiers, secrets, redirect URIs, scopes, and roles directly through `config/services.php`.
- Do not create environment variables that reference one another in a cascade when explicit values are expected.
- Map SSO roles to application profiles in a dedicated service.
- Repeat authorization checks in Policies, middleware, or sensitive Actions.
- The frontend hides forbidden actions, but the backend remains authoritative.

## Files

- Validate uploads in a `FormRequest`.
- Adapt `UploadedFile` at the HTTP boundary.
- Application receives a content DTO or stream contract.
- For a large file or ZIP archive, use a stream or a path encapsulated by an adapter; do not load and then copy the entire archive in memory.
- Check the extension, MIME type, size, read errors, and ZIP path traversal.
- Close resources in a `finally` block.

## Configuration

- `env()` is allowed only in `config/*.php`.
- Application code uses `config()`.
- `APP_URL` is the canonical origin; do not multiply equivalent variables.
- Use `APP_SCHEME` only when a reverse proxy or runtime requires the scheme to be forced.
- Provide an `.env.example` without real secrets.
- Never commit `.env`.

## Docker

### Development

`docker-compose.yml` must provide:

- the Laravel application;
- PostgreSQL;
- code, `vendor`, and `node_modules` volumes;
- Artisan and Vite accessible from the host;
- a PostgreSQL health check;
- configurable ports without relying on `0.0.0.0` in URLs generated by Vite.

### Production

`docker-compose.prod.yml` must:

- use the immutable GHCR image;
- not mount source code;
- not run Vite;
- configure health checks and dependencies;
- pass only the required configuration;
- run migrations through a controlled step.

### Multi-stage Dockerfile

Provide at least:

1. a PHP base with extensions;
2. a Composer stage without scripts before the complete application is copied when Laravel cannot boot yet;
3. a Node stage that compiles assets;
4. a `dev` target with development tools;
5. a minimal `prod` target with optimized vendor files and compiled assets.

Avoid running `composer install` with Laravel scripts before the application, its providers, and its Artisan file have been copied.

## GitHub Actions and GHCR

Create a workflow that:

1. checks out the repository;
2. enables Buildx;
3. signs in to `ghcr.io` with `GITHUB_TOKEN`;
4. builds the `prod` target;
5. publishes an immutable SHA-based tag;
6. optionally publishes the branch tag or `latest`, according to project policy;
7. uses the GitHub Actions cache;
8. embeds no application secret in the image.

Grant only:

```yaml
permissions:
    contents: read
    packages: write
```

## Migration from Next.js or Supabase

Proceed screen by screen and flow by flow.

1. Inventory routes, props, actions, statuses, cookies, sessions, copy, and edge cases.
2. Capture the functional reference before rewriting anything.
3. Migrate the schema and data to PostgreSQL.
4. Reproduce business rules in Domain/Application.
5. Reproduce pages with Inertia and React without changing the design.
6. Compare payloads and effects before and after.
7. Remove Next and Supabase dependencies only after reaching parity.

Never treat a rendered page as proof of parity. Verify mutations, concurrent states, exports, integrations, permissions, and sensitive data.

## Delivery checks

Adapt commands to the project, then run at least:

```bash
composer validate --strict
composer analyse
vendor/bin/pint --test
php artisan migrate:fresh --force
php artisan route:list
php artisan schedule:list
npm run typecheck
npm run lint
npm run build
docker compose config
docker compose -f docker-compose.prod.yml config
docker build --target prod .
```

Before delivery:

- no Larastan errors at the highest configured level;
- no HTTP dependency in Application;
- no direct external API access from a controller;
- no active legacy Next/Supabase reference;
- no sensitive data in logs, audits, or props;
- no regression in routes, roles, cookies, HTTP statuses, redirects, copy, or design;
- a clean Git tree and a commit limited to the requested scope.

## Anti-patterns

Reject:

- a controller hundreds of lines long;
- an all-knowing service that mixes SQL, business logic, HTTP, and presentation;
- `app()` or `resolve()` used as a service locator;
- an untyped business array crossing multiple layers;
- `Request` in Application;
- `env()` access from a service;
- single-line styled-components;
- raw HTML or inline styles in components;
- loading an entire large ZIP archive into memory;
- a network call under a database lock without justification;
- a stack replacement that silently simplifies or removes a feature.
