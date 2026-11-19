# Architecture Decision Records

A log of architectural decisions, problems encountered, options considered, and reasoning behind every significant choice made during the build of this project. Written retrospectively after completion.

---

## ADR-001: Docker Engine directly in WSL2 over Docker Desktop

Context:
Two installation options existed for Docker on WSL2 - Docker Desktop for Windows (GUI, integrates with WSL2) or Docker Engine natively inside Ubuntu.

Options considered:

1. Docker Desktop - easier setup, Windows GUI
2. Docker Engine in WSL2 - pure Linux, no GUI.

Decision: Docker Engine natively in WSL2.

Reason: lightweight, practical, faster setup, Docker runs as a Linux service in professional environments, not through a Windows GUI. Learning the native approach builds more transferable skills and better reflects what would be encountered on the job.

---

## ADR-002: Separate composer install from autoloader generation in Dockerfile

Context:
Initial Dockerfile ran `composer install --optimize-autoloader` at the stage where only `composer.json` and `composer.lock` had been copied. This failed because `--optimize-autoloader` generates a class map that requires the full application codebase to exist.

Problem chain:

- `COPY composer.json composer.lock ./`
- `RUN composer install --optimize-autoloader` - fails, no app code to map
- `COPY . .` - source code arrives too late

Decision: Two-step approach:

1. `composer install --no-scripts --no-autoloader` early (pure package download)
2. `composer dump-autoload --optimize` after `COPY . .` (full app exists)

Reason: Docker layer caching makes this efficient - the slow package download step is cached and only reruns if `composer.json` or `composer.lock` change. The autoloader generation runs after all code exists, which is the correct order.

---

## ADR-003: Remove Pail and Breeze from `bootstrap/providers.php`

Context:
During the Docker build, `composer dump-autoload` triggers `php artisan package:discover` which tries to register all service providers. Laravel Pail and Laravel Breeze are both `--dev` dependencies and were not installed (correct for production). Their service providers were registered in `bootstrap/providers.php`, causing the build to fail.

Decision: Remove `Laravel\Pail\PailServiceProvider` and `Laravel\Breeze\BreezeServiceProvider` from `bootstrap/providers.php`.

Reason: Both are development-only tools. Pail is a log tailing utility. Breeze's job was to scaffold auth files - that scaffolding is already baked into the codebase. Neither has any runtime role in production. Removing them is correct, not a workaround.

---

## ADR-004: Service Orchestration & Health Checks

Context:
Nginx frequently crashed on startup with host not found in upstream "app". This was caused by a "Race Condition" where Nginx tried to resolve
the app’s DNS before the PHP-FPM service was fully registered.

Decision:
Implement a Healthcheck-driven Orchestration in docker-compose.yml.

Reason:
Nginx is now delayed until the app container reports a "Healthy" status via netcat. This eliminated the need for manual docker compose down/up restarts.

Additional Comments:
This is an important learning moment, healthcheck and depends on are crucial and extremely useful tool when orchestrating,
learning which service goes first and the timeline on how docker run these things and learning what the app needed to is important yet super fun.

---

## ADR-005: Volume mount is a local development shortcut, not best practice

Context:
Initial `docker-compose.yml` included `- .:/var/www` which bind-mounts the entire project directory from WSL2 into the app container. This was introduced for local development convenience - file changes reflect immediately without rebuilding.

Problems caused by this decision:

1. Symfony tempnam error - Dockerfile `chown -R www-data:www-data /var/www` was overwritten at runtime by the volume mount, which restored WSL2 user ownership. PHP-FPM couldn't write to `storage/` because it didn't own it.
2. CI vendor error - GitHub Actions runner has no `vendor/` folder (it's gitignored). Volume mount overwrote the `vendor/` folder that Composer built into the image, breaking the container.
3. WSL2 file lock - after moving `chown` to the entrypoint script, PHP-FPM took ownership of all of `/var/www` including source files, making them uneditable from WSL2.

What should have been understood upfront:
Volume mounting the entire project directory is a development shortcut with significant tradeoffs - it defeats Docker's immutability, causes permission conflicts, and breaks CI. It should be introduced explicitly as a shortcut with stated tradeoffs, not as the standard approach.

Resolution: See ADR-006 and ADR-007.

---

## ADR-006: Scope `chown` to only `storage/` and `bootstrap/cache/`

Context:
After moving `chown` to the entrypoint script (to run after the volume mount), the initial command was `chown -R www-data:www-data /var/www` - giving PHP-FPM ownership of the entire project. This locked the WSL2 user out of
the source code, preventing access and live editing in VS Code.

Analysis:
PHP-FPM only needs write access to two directories:

- `storage/` - logs, sessions, cache, uploads
- `bootstrap/cache/` - cached config, routes, services

All other project files are read-only at runtime.

Decision:
Scope `chown` to only those two directories.

Reason:
Principle of least privilege. Give PHP-FPM write access only where Laravel actually writes. WSL2 user retains ownership of everything else.

---

## ADR-007: Multi-stage Dockerfile with Double-Bake pattern for Nginx

Context:
Nginx needs access to `public/` to serve static files and with previously stated Bind-mount volumes caused an overlap, where empty host folders hid the pre-installed vendor folder in CI/CD (GitHub Actions)
causing ci test on artisan check to fail, but With separate containers this creates a file sharing problem.

Options considered:

1. Keep bind mount `./public:/var/www/public` - mutability risk, permission issues on CI runner, mostly for local development ease.
2. Named shared volume - both containers mount the same volume, simpler but introduces a single point of failure and complicates scaling.
3. Double-Bake / Multi-stage - both App and Nginx images bake in the files they need independently during build time

Decision: Option 3 - Multi-stage Dockerfile with five stages:

- `php-base` - PHP-FPM with system dependencies and extensions
- `assets-compiler` - Node 24 Alpine, compiles Vite frontend assets
- `vendor-staging` - Composer install, generates vendor directory
- `app` - assembles final PHP-FPM image from above stages
- `web` - Nginx image with config and compiled public assets baked in

Reason:
True immutability. Each image is self-contained. No shared volumes, no bind mounts for application code, no host filesystem dependency. Scales correctly - spinning up multiple app and
Nginx containers works identically because each has everything it needs baked in. Also resolved all volume mount permission problems permanently for CI.

Additional Comments:
So this is a true headache but the good kind because it give me technical depth, insight and cleaner final build, also learn a lot about dockerfiles here, the intricasies and how it works in the larger picture

---

## ADR-008: Secret Governance via Variable Substitution

Context:
Hardcoded database credentials in docker-compose.yml created a security risk and prevented the project from being "GitHub-Safe."

Decision:
Utilize Interpolation ${VAR} in the YAML file to pull values from the .env file.

Reason:
The infrastructure is now "Secret-less." Credentials can be managed via GitHub Secrets or local .env files without modifying the core architecture.

Additional Comments:
in hindsight this was a rookie mistake, but it's a good mistake to learn at the beginning now i know where and how to handle db and secrets so i will
never upload sensitive credential to github, as explained this is a production build, so utilizing github secret to inject .env during ci test
and then using variable substitution to pull the credential data from source file.

---

## ADR-009: Move `storage:link` from entrypoint to Dockerfile build stage

Context:
`php artisan storage:link` was in the entrypoint script. When the container runs as `www-data`, it lacks write permission on `public/` to create the symlink - only `storage/` and `bootstrap/cache/` have `www-data` ownership per ADR-009.

Options considered:

1. Add `public/` to the targeted `chown` in entrypoint
2. Run `storage:link` during build as root, bake the symlink into the image

Decision: Option 2 - run `storage:link` in the Dockerfile as root during build.

Reason: The symlink is static - it never changes between deployments. Creating it at build time means it's baked into the image, always present, no runtime permission complexity. Root is available during build so no permission issues arise.

---

## ADR-010: Init container pattern for database migrations

Context:
`php artisan migrate --force` was in the entrypoint script. At small scale this works. At scale, multiple app containers start simultaneously and all run migrations against the same database - causing race conditions.

This directly contradicted the scalability goal of ADR-010. Building immutable multi-stage images for scale while running migrations in the entrypoint was architecturally inconsistent.

Options considered:

1. Leave migrations in entrypoint - simple but creates race conditions at scale
2. Init container pattern - separate `migrator` service using the same `app` image, runs once, exits, app starts only after successful completion
3. Kubernetes Job - correct at enterprise scale, out of scope for this project

Decision: Option 2 - init container pattern via Docker Compose:

- `migrator` service: `target: app`, `restart: "no"`, runs `php artisan migrate --force`
- `app` service: `depends_on: migrator: condition: service_completed_successfully`

Reason: Migrations run exactly once before any app container starts. No race conditions possible. The `migrator` uses the same image as `app` - no additional build overhead. Consistent with the scalability architecture established in ADR-010.

---

## ADR-011: GitHub Secrets for CI environment injection

Context:
`.env` is gitignored - correctly. The CI pipeline needs real environment values to build and run the Docker stack.

Options considered:

1. Commit real values to `.env.example` - security violation, credentials in public repo
2. GitHub Secrets - encrypted values stored in GitHub, injected at pipeline runtime

Decision: GitHub Secrets with a workflow step that generates `.env` on the runner before `docker compose up`.

Reason: Extends the project's core principle - no hardcoded credentials anywhere - to the CI pipeline itself. Values never appear in code, logs, or pull requests.

---

## ADR-012: Dynamic Backend Resolution for Horizontal App Scaling

Context:
Alpha V1 was designed as a stateless, horizontally-scalable app tier, but two implementation details silently blocked actual scaling from working: (1) the app service had a hardcoded container_name,
which Docker rejects once replica count exceeds 1, and (2) nginx.conf used a static fastcgi_pass app:9000, which resolves the backend hostname once at Nginx startup and never re-checks - so even with multiple replicas running,
all traffic would pin to whichever single IP was resolved first, with new replicas never receiving traffic.

Options Considered:

1. NGINX Plus resolve parameter on an upstream block - provides real dynamic re-resolution with proper failover (fastcgi_next_upstream). Rejected: paid product, out of scope for this project's budget/tier.
2. Docker Swarm with VIP-based service discovery - Swarm's routing mesh handles it natively. Rejected: out of scope for this project.
3. K8s Service - same category of solution as Swarm, at greater infrastructure cost. Rejected: same reasoning, Out of scope for this project.
4. Nginx resolver directive + variable-based fastcgi_pass, relying on Docker's embedded DNS - forces Nginx to re-resolve the backend hostname on a short TTL instead of once at startup; distribution achieved via Docker's own DNS answer rotation across replicas sharing a network alias, not via Nginx load-balancing logic. Selected.

Decision:
Removed container_name from the app service to allow Compose to auto-name replicas uniquely. Adopted the resolver 127.0.0.11 (Docker's embedded DNS) + Nginx variable pattern (set $backend app:9000; fastcgi_pass $backend;)
with an aggressive valid=1s TTL, functioning as a "poor man's load balancer", but with a known structural gap: no upstream block means no fastcgi_next_upstream failover, so a request resolving to a
since-dead replica within the current TTL window fails outright rather than retrying. Manual testing (repeated /health curls, including mid-loop replica kill) confirmed distribution across replicas and fast self-healing (~1s)
with no failed requests observed, though the failure window remains structurally present.

Reason:
Every rejected option solved the problem correctly but at a cost disproportionate to a portfolio project's scope - either financial (NGINX Plus) or architectural (Swarm/K8s migration, both already positioned as separate future learning goals in the project roadmap). The selected approach demonstrates genuine understanding of the underlying DNS/load-balancing mechanics using only free, already-present tooling workaround, while the known limitation is small, explainable, and honestly documented rather than hidden.

---
