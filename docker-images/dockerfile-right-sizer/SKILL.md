---
name: "dockerfile-right-sizer"
description: "Writes, reviews or optimizes Dockerfiles without over-engineering, using repo evidence and minimal user questions, then explains each decision. Use when asked to create, review, fix, modify or harden a Dockerfile, .dockerignore or Containerfile, or to containerize/dockerize an app."
---

# Dockerfile Right-Sizer

Produce the **simplest Dockerfile that is correct, secure and maintainable for this specific project**. Apply a cheap baseline every time. Add any advanced technique **only when evidence shows it is needed**. Every decision must trace back to evidence (a file, a config, a measurement, or a user answer) and be explained in the final report.

## Core principles

1. **Evidence over assumption.** Never invent ports, start commands, runtime versions, architectures or requirements. Find them in the repo. If you apply general knowledge, apply it *to* a piece of evidence (e.g. "`bcrypt` in package.json means a native module, so prefer a glibc base") and say so.
2. **Baseline always, extras on demand.** The baseline costs a few lines and has no real downside. Everything else must answer "yes" to its decision question.
3. **Self-sufficient first, user second.** Investigate before asking. Ask at most **3** questions, batched in one message, and only when the answer is not discoverable **and** would materially change the Dockerfile.
4. **Skipping is a decision.** Record every technique you deliberately did *not* use, with the reason and the trigger that would make it worth adding later.
5. **Measure before optimizing.** Size or speed optimizations beyond the baseline need a measured problem.
6. **Preserve what works.** When improving an existing Dockerfile, keep working behavior, make the smallest change that fixes real problems, and explain each change.

## Workflow

### Phase 1: Gather evidence (do this before deciding anything)

Inspect the repository and any context the user gave. Record each finding as a fact with its source (`path:line` or the command output). Look for:

| Need to know | Where to look |
|---|---|
| Language and package manager | `package.json` + lockfile (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`), `requirements.txt`, `pyproject.toml`, `poetry.lock`, `uv.lock`, `Pipfile.lock`, `go.mod`, `pom.xml`, `build.gradle*`, `Cargo.toml`, `*.csproj`, `Gemfile.lock`, `composer.lock` |
| Runtime version | `.nvmrc`, `.node-version`, `engines` in package.json, `.python-version`, `requires-python`, `go` / `toolchain` directive in go.mod, `java.version` / toolchain in build files, `.tool-versions`, CI setup steps (`actions/setup-node` etc.) |
| Is there a build step? | `scripts.build`, `tsconfig.json`, bundler configs (`vite.config.*`, `webpack.config.*`, `next.config.*`, `nuxt.config.*`), compiled languages, `Makefile` targets |
| Start command | `scripts.start`, `main`, `Procfile`, `[project.scripts]`, `cmd/` directories, framework conventions, existing compose/k8s `command` |
| Listening port and address | Code that calls `listen(...)` / `bind`, `PORT` env usage, framework config, existing compose/k8s ports. Check whether it binds `127.0.0.1` (a container bug: must be `0.0.0.0`). |
| Native/OS dependencies | `bcrypt`, `sharp`, `canvas`, `node-gyp`, `prisma`, `psycopg2` (non-binary), `mysqlclient`, `lxml`, `numpy`/`pandas` wheels, CGO usage, `apt`/`apk` mentions in docs or CI |
| Build-time credentials | `.npmrc` with registry auth, `pip.conf`/`--index-url`, private Git dependencies, `GOPRIVATE`, CI secrets passed to the build |
| Target architecture | k8s `nodeSelector`/affinity on `kubernetes.io/arch`, Terraform/cloud instance types (ARM families such as AWS Graviton `*g` types), `platforms:` in CI or buildx config, fly/render configs, explicit user statements |
| Dependency update automation | `renovate.json`, `.github/renovate.json`, `.github/dependabot.yml` (check for a `docker` ecosystem entry) |
| Existing container setup | `Dockerfile*`, `Containerfile`, `.dockerignore`, `compose*.yaml`, k8s manifests, Helm charts, CI workflows that build images |
| Tests and CI | Test scripts, CI workflows (do tests run inside Docker or natively?) |
| Health endpoint and signals | Routes such as `/health`, `/healthz`; `SIGTERM` handlers |
| Security/compliance requirements | `SECURITY.md`, policy docs, README statements, regulated-domain hints |
| Repo layout | Monorepo (workspaces, `apps/`, `packages/`), which directory is the build context |

If Docker is available in the environment, you may also measure: `docker image ls`, `docker history`, build time of an existing Dockerfile.

### Phase 2: Apply the baseline (always)

- Official or trusted **minimal base** (`-slim` or `-alpine` variant) with a **specific version tag** derived from evidence. Never `latest`.
- `.dockerignore` excluding at least: `.git`, dependency folders (`node_modules`, `.venv`, `target`, `vendor` when re-installed), `.env*`, keys/certs (`*.pem`, `*.key`), local build output, IDE folders, `Dockerfile*`.
- Copy dependency manifests and lockfile **before** source code; install with the lockfile-respecting command (`npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable` or `--frozen-lockfile`, `pip install -r` with pinned requirements, `uv sync --frozen`, `poetry install` without dev groups, `go mod download`).
- **Production dependencies only** in the runtime image.
- Combine related package-manager commands in one `RUN`, and clean caches in the same `RUN` (`apt-get update && apt-get install -y --no-install-recommends ... && rm -rf /var/lib/apt/lists/*`).
- **Non-root `USER`** (use the image's built-in user, e.g. `node`, or create one with a fixed UID). Set file ownership with `COPY --chown`.
- **Exec-form** `CMD` / `ENTRYPOINT`, starting the runtime directly (e.g. `["node", "server.js"]`, not `npm start`).
- **No secrets** in `ARG`, `ENV`, or copied files.
- `EXPOSE` the evidenced port. `ENV` only for genuine runtime defaults (e.g. `NODE_ENV=production`, `PYTHONUNBUFFERED=1`).
- `# syntax=docker/dockerfile:1` as the first line.

### Phase 3: Decide the extras (each needs evidence)

For each row, answer the question from Phase 1 evidence. Use the "Unknown" column only when evidence is absent and asking the user isn't worth it.

| Technique | Decision question | Evidence for YES | Evidence for NO | Unknown → default |
|---|---|---|---|---|
| Multi-stage build | Does the build need tools the runtime doesn't (compiler, TypeScript, bundler, framework build, devDependencies needed to build)? | Build script, tsconfig, compiled language, bundler config | Plain interpreted code run as-is, no build script | Single stage |
| Test stage | Must tests run inside the image (e.g. CI builds `--target test`)? | CI invokes Docker for tests; user says so | CI runs tests natively | Skip |
| Base: alpine vs slim | Are there native modules or glibc-dependent binaries? | Native deps found (see Phase 1) | Pure interpreted deps | `-slim` (safer compatibility) |
| Distroless or scratch | Is the artifact self-contained (static binary or runtime-only bundle) **and** is there a reason (security requirement, compliance) **and** no need for a shell at runtime? | Static Go/Rust binary; security policy | Shell scripts in entrypoint; team debugs via `exec sh` | Not used |
| Digest pinning | Is exact reproducibility required **and** is an update bot configured for Docker? | Renovate/Dependabot docker config present; compliance needs | No update automation | Specific version tag + rebuild with `--pull` (explain that digest pinning without automation stops patches) |
| Build secrets (`--mount=type=secret`) | Does the build itself need credentials? | Private registry auth, private Git deps | All deps public | Skip |
| Cache mounts (`--mount=type=cache`) | Are builds measurably slow due to dependency downloads, and frequent? | Measured slow builds; user complaint | No measurement or fast builds | Skip (layer ordering is enough) |
| Multi-platform | Will the image run on more than one CPU architecture? | ARM instance types, arch selectors, `platforms:` in CI, user statement | Single known target | Build for the current target only |
| Extra size reduction | Is image size causing a real problem (slow deploys, cold starts, registry cost)? | Measured or stated problem | Not stated | Stop at baseline |
| `HEALTHCHECK` instruction | Is the image run with plain Docker/Compose (not an orchestrator with its own probes), with a health endpoint and a tool to call it? | Compose usage, health route, `wget`/`curl` available | Kubernetes probes defined | Skip; mention in report |
| Init process (`tini` / `--init`) | Does the app spawn child processes or run shell wrappers as PID 1? | Process spawning, shell entrypoint scripts | Single-process server with signal handling | Skip; note `docker run --init` / Compose `init: true` as an option |
| OCI labels | Will images be published to a registry? | CI pushes images; registry configs | Local use only | Add `org.opencontainers.image.source` only if the repo URL is known |

### Phase 4: Ask the user (only if needed)

- Ask only about unresolved items that **change the Dockerfile materially** (typically: target architecture, whether the build needs private credentials, the start command when truly ambiguous, a monorepo service choice).
- Max **3 questions**, in **one** batched message. For each, state your recommended default and why.
- If the user can't answer or is unavailable, proceed with the defaults from Phase 3 and label them as **assumptions** in the report.
- Never ask about things the repo already answers.

### Phase 5: Write the files

- Output `Dockerfile` and `.dockerignore` (and nothing else unless asked).
- Keep it as short as the decisions allow. Add a brief comment only where a line is non-obvious (e.g. why a stage exists, why `slim` over `alpine`).
- Use runtime versions, ports, paths and commands **exactly as found in evidence**.

Reference patterns (adapt to evidence, never copy blindly; replace `<version>` with the evidenced version):

**Node.js, no build step**
```dockerfile
# syntax=docker/dockerfile:1
FROM node:<version>-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --chown=node:node . .
USER node
EXPOSE <port>
CMD ["node", "<entry-file>"]
```

**Node.js with a build step (TypeScript/bundler)**: multi-stage is justified
```dockerfile
# syntax=docker/dockerfile:1
FROM node:<version>-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:<version>-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build --chown=node:node /app/<build-output-dir> ./<build-output-dir>
USER node
EXPOSE <port>
CMD ["node", "<build-output-dir>/<entry-file>"]
```

**Python (pip)**
```dockerfile
# syntax=docker/dockerfile:1
FROM python:<version>-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN useradd --create-home --uid 10001 appuser
COPY --chown=appuser:appuser . .
USER appuser
EXPOSE <port>
CMD ["<server>", "<args binding 0.0.0.0:<port>>"]
```

**Go (compiled)**: multi-stage is justified; minimal runtime fits a static binary
```dockerfile
# syntax=docker/dockerfile:1
FROM golang:<version> AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app <main-package-path>

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
EXPOSE <port>
ENTRYPOINT ["/app"]
```
If CGO is required (evidence: `import "C"`, CGO-dependent libraries), use a glibc runtime base instead and explain why.

**Java (Maven/Gradle)**: multi-stage is justified (JDK to build, JRE to run)
```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:<version>-jdk AS build
WORKDIR /src
COPY . .
RUN <evidenced build command, e.g. ./mvnw -B package -DskipTests>

FROM eclipse-temurin:<version>-jre
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
COPY --from=build --chown=appuser:appuser /src/<path-to-jar> app.jar
USER appuser
EXPOSE <port>
CMD ["java", "-jar", "app.jar"]
```

### Phase 6: Verify (when possible)

If Docker is available:
- `docker build -t <name>:check .` succeeds.
- `docker run --rm <name>:check whoami` (or `id`) is not `root` (skip for distroless; confirm via image config `User`).
- The container starts and the port responds (health route if it exists).
- `docker history --no-trunc <name>:check` contains no secrets.
- Record image size and build time.
- Run `hadolint` if available.

If Docker is not available, say exactly which checks the user should run. Never claim something was verified when it wasn't.

### Phase 7: Report (always, at the end)

Use this structure:

```markdown
## Dockerfile summary
One or two sentences: what was produced and the overall shape (e.g. "single-stage, non-root, Node 22 slim").

## Evidence found
| Fact | Source |
|---|---|
| Node 22 | `.nvmrc:1` |
| Listens on 0.0.0.0:8000 | `src/server.js:42` |

## Decisions
| Decision | Why (evidence) |
|---|---|
| `node:22-slim` over alpine | `bcrypt` native module in `package.json:14` |

## Deliberately not used
| Technique | Why not | Add it when |
|---|---|---|
| Multi-stage | No build step | You add TypeScript or a bundler |

## Assumptions (no evidence available)
- Target is amd64 only (no architecture hints found; you didn't specify).

## Questions asked
- (list, or "None: the repo answered everything")

## Verification
- What was run and the results, or the exact commands for the user to run.

## Issues found outside the Dockerfile
- e.g. "`server.js` binds 127.0.0.1, so it will be unreachable in a container; change to 0.0.0.0."
```

## Guardrails against over-engineering

Stop and simplify if any of these is true:
- The Dockerfile is longer than the complexity of the app justifies.
- A technique was added that you can't tie to a specific piece of evidence.
- Size or speed was optimized without a measurement or stated problem.
- Distroless was chosen while entrypoint scripts or debugging needs a shell.
- Digests are pinned with no update automation.
- Multi-platform builds were added with no evidence of a second architecture.
- Stages exist that copy nothing different into the final image.
- You're adding Compose, CI or Kubernetes files the user didn't ask for.

## Evidence rules

- **Code beats docs:** if README says port 3000 but the code listens on 8000, trust the code and mention the mismatch.
- **Lockfile beats manifest** for exact dependency versions.
- **Explicit config beats convention** (e.g. a `start` script beats a framework default).
- **Conflicting evidence:** pick the most authoritative source, and report the conflict.
- **Confidence:** mark any decision based on weak or indirect evidence as such in the report.
- **Unknowns become labeled assumptions**, never silent guesses.