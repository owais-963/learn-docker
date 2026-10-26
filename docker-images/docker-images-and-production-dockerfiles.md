# Docker Images and Production Dockerfiles

> Engineering notes on how images work and how to make deliberate Dockerfile decisions: build efficiency, dependency control, security, reproducibility and runtime behavior.
>
> **Running example:** a small Node.js (Express) API from my learning project. The principles apply to other runtimes; the commands and details are Node-specific where noted.
>
> **Author:** Owais Arshad · **Status:** learning notes, reviewed against the official documentation listed in [References](#22-references). Corrections are welcome via issues or pull requests.

**How to read the labels used below**

| Label | Meaning |
|---|---|
| **[Universal]** | Applies to almost every production image. Skipping it needs a specific reason. |
| **[Context]** | The right answer depends on the application, base image, runtime or deployment platform. |
| **[Optional]** | Worth doing when you have evidence of a problem it solves (measured, stated or required by policy). |

---

## Contents

1. [What an image is and why its design matters](#1-what-an-image-is-and-why-its-design-matters)
2. [Image fundamentals and layers](#2-image-fundamentals-and-layers)
3. [How builds work and how the cache behaves](#3-how-builds-work-and-how-the-cache-behaves)
4. [Build context and `.dockerignore`](#4-build-context-and-dockerignore)
5. [Choosing a base image](#5-choosing-a-base-image)
6. [Writing maintainable Dockerfiles](#6-writing-maintainable-dockerfiles)
7. [Installing dependencies](#7-installing-dependencies)
8. [Multi-stage builds](#8-multi-stage-builds)
9. [Image size versus security, compatibility and operability](#9-image-size-versus-security-compatibility-and-operability)
10. [Running as a non-root user](#10-running-as-a-non-root-user)
11. [File ownership, permissions and the filesystem](#11-file-ownership-permissions-and-the-filesystem)
12. [Secrets and credentials](#12-secrets-and-credentials)
13. [`ARG`, `ENV` and runtime configuration](#13-arg-env-and-runtime-configuration)
14. [Pinning base images and dependencies](#14-pinning-base-images-and-dependencies)
15. [Tagging, reproducibility and the supply chain](#15-tagging-reproducibility-and-the-supply-chain)
16. [Runtime behavior: `CMD`, `ENTRYPOINT`, signals, shutdown and health](#16-runtime-behavior-cmd-entrypoint-signals-shutdown-and-health)
17. [BuildKit features worth knowing](#17-buildkit-features-worth-knowing)
18. [Scanning, SBOMs, provenance and verification](#18-scanning-sboms-provenance-and-verification)
19. [Common Dockerfile mistakes and their consequences](#19-common-dockerfile-mistakes-and-their-consequences)
20. [A worked example with trade-offs](#20-a-worked-example-with-trade-offs)
21. [Using AI to generate and review Dockerfiles](#21-using-ai-to-generate-and-review-dockerfiles)
22. [References](#22-references)
- [Appendix A: production Dockerfile review checklist](#appendix-a-production-dockerfile-review-checklist)

---

## 1. What an image is and why its design matters

An **image** is a read-only, layered filesystem plus a **configuration** (default command, environment, user, working directory, exposed ports, labels). A **container** is a process started from that image, with a thin writable layer on top.

The image is the **unit of deployment**. Whatever is inside it ships to every environment that runs it: the dependencies, the files you forgot to exclude, the user it runs as and the way it handles a stop request. That is why Dockerfile decisions show up later as operational problems:

| Dockerfile decision | Where it shows up later |
|---|---|
| What goes into the image | Image size, pull time, number of vulnerability findings, leaked files |
| How layers are ordered | Build time and CI cost |
| How versions are referenced | Whether a build or rollback can be reproduced |
| Which user runs the process | The impact of an application compromise |
| How the process is started | Whether deployments shut down cleanly or drop requests |

A successful `docker build` only proves that the instructions executed. It says nothing about whether the result is small enough, safe enough, reproducible, or well behaved when an orchestrator stops it. Most of these notes are about closing that gap.

---

## 2. Image fundamentals and layers

### Layers and configuration

- Instructions that change the filesystem (`RUN`, `COPY`, `ADD`) produce **layers**: archives of the changes they made.
- Instructions such as `ENV`, `CMD`, `ENTRYPOINT`, `USER`, `EXPOSE`, `LABEL` and `WORKDIR` mainly change the **image configuration**. (`WORKDIR` also creates the directory if it doesn't exist.)
- A storage driver (for example overlayfs) stacks the layers into one view. Upper layers hide files in lower layers.

```
 ┌──────────────────────────────┐
 │ container writable layer     │  created at `docker run`, removed with the container
 ├──────────────────────────────┤
 │ COPY server.js               │  ┐
 │ RUN npm ci --omit=dev        │  │ image layers: read-only, content-addressed,
 │ COPY package*.json           │  │ shared between images that use the same ones
 │ base image layers            │  ┘
 └──────────────────────────────┘
```

### Deleting a file later does not remove it **[Universal]**

A layer records changes. If one instruction adds a file and a later instruction deletes it, the later layer only records a "whiteout" that hides the file. The bytes are still in the earlier layer, still downloaded with the image and still recoverable by anyone who can pull it.

```dockerfile
# Careless: the archive (and any secret in it) stays in the first layer
RUN curl -fsSLo /tmp/tool.tgz https://example.com/tool.tgz
RUN tar -xzf /tmp/tool.tgz -C /opt && rm /tmp/tool.tgz
```

```dockerfile
# Better: download, use and delete inside one RUN, so the archive never lands in a layer
RUN curl -fsSLo /tmp/tool.tgz https://example.com/tool.tgz \
 && tar -xzf /tmp/tool.tgz -C /opt \
 && rm /tmp/tool.tgz
```

For anything sensitive, "delete it in the same `RUN`" is still not enough: see [section 12](#12-secrets-and-credentials). A multi-stage build ([section 8](#8-multi-stage-builds)) is often the cleaner fix, because nothing from the build stage reaches the final image unless you copy it.

### Copy-on-write and sharing

When a container modifies a file that came from the image, the file is first copied into the container's writable layer. Identical layers are stored once and shared between images, which is why a common base image is cheap to reuse across services.

### Identity: tags, digests and indexes

| Term | Meaning |
|---|---|
| **Tag** (`node:22-slim`) | A human-friendly, **mutable** pointer. The publisher can move it to a different image. |
| **Digest** (`node@sha256:…`) | A content hash. The same digest always refers to the same bytes. |
| **Image index** (manifest list) | One reference that points to several platform-specific images (`linux/amd64`, `linux/arm64`, …). The client pulls the matching one. |
| **Image ID** | The local identity of an image's configuration. |

### Inspecting what you built

```bash
docker history node:22-slim                          # layers and the instruction that created each
docker image inspect node:22-slim -f '{{json .Config}}'   # Cmd, Env, User, WorkingDir…
docker buildx imagetools inspect node:22-slim        # digests and platforms in the index
docker diff <container>                              # changes in a container's writable layer
```

[`dive`](https://github.com/wagoodman/dive) is a useful third-party tool for exploring layer contents. If you run it with the Docker socket mounted, remember that socket access is equivalent to root on the host, so only do that with tools you trust.

---

## 3. How builds work and how the cache behaves

Since Docker Engine 23.0, `docker build` uses **BuildKit**. BuildKit builds a dependency graph from the Dockerfile, runs independent stages in parallel, skips stages the requested target doesn't need, and reuses cached results when an instruction and its inputs are unchanged.

### Cache rules that matter in practice

| Instruction | What invalidates its cache |
|---|---|
| `RUN` | A change to the command string or to any earlier step. **Not** a change in the outside world: `RUN apt-get update` stays cached even after the upstream package index changes. |
| `COPY` / `ADD` | A change in the content (and relevant metadata) of the copied files. |
| `ARG` / `ENV` | A changed value, for the steps that use it. |

Once one step is rebuilt, every later step in that stage is rebuilt too.

### Instruction order **[Universal]**

**Decision:** put steps that change rarely and cost the most (dependency installation) before steps that change often (application source).

**Careless version:**

```dockerfile
COPY . .
# Any source change invalidates this step and reinstalls every dependency
RUN npm ci
```

**Consequence:** every code change triggers a full dependency install. On a project with many dependencies, that is minutes per build, multiplied by every CI run.

**Better:**

```dockerfile
COPY package.json package-lock.json ./
# Cached until the manifest or lockfile changes
RUN npm ci
# Only this layer and later ones rebuild on code changes
COPY . .
```

**When it doesn't matter:** a tiny service with few dependencies and rare builds won't notice the difference. It costs nothing to do it anyway.

### Getting fresh packages on purpose

Images are snapshots. They don't receive security patches unless you rebuild them. The official guidance is to rebuild regularly:

- `docker build --pull …` checks for a newer version of the base image.
- `docker build --no-cache …` re-runs every step, but **does not** fetch a newer base image on its own.
- Use both together for a fully fresh build, typically on a schedule in CI.

---

## 4. Build context and `.dockerignore`

The **build context** is the set of files sent to the builder (the `.` in `docker build .`). Anything in the context can be copied into the image, and a large context slows every build.

### Why `.dockerignore` is a security control as well as a performance one **[Universal]**

**Careless version:** no `.dockerignore`, combined with `COPY . .`.

**Consequences:**

- `.env` files, private keys and local credentials end up inside the image.
- `.git` ends up inside the image, including the history of every secret that was ever committed and later "removed".
- Local `node_modules` (built for your laptop's OS and architecture) can overwrite the dependencies installed in the image.
- The context upload is larger, and unrelated file changes invalidate the `COPY . .` layer.

**Better:** exclude everything the image doesn't need.

```gitignore
# .dockerignore
.git
.gitignore
node_modules
npm-debug.log*
.env
.env.*
*.pem
*.key
coverage
test-results
.vscode
.idea
Dockerfile*
.dockerignore
```

An even stricter pattern is an **allowlist**: ignore everything, then re-include what the build needs.

```gitignore
*
!package.json
!package-lock.json
!server.js
!test/
```

The allowlist is safer for security-sensitive repositories, but needs maintenance whenever the build needs a new file. Pick the style your team will actually keep correct.

**Related details:**

- A `.dockerignore` is not a substitute for copying selectively. In the final stage, `COPY` only the files the runtime needs ([section 20](#20-a-worked-example-with-trade-offs)).
- If you keep several Dockerfiles in one directory, BuildKit also supports a Dockerfile-specific ignore file named `<Dockerfile-name>.dockerignore` (for example `Dockerfile.dev.dockerignore`).
- Docker's `CopyIgnoredFile` build check warns when a `COPY` refers to a file the ignore rules exclude.

---

## 5. Choosing a base image

The official best-practices guide gives two criteria: a **trusted source** (Docker Official Images, Verified Publisher, or Docker-Sponsored Open Source) and **small**. Everything else depends on the application.

| Base type | Example | Strengths | Costs and caveats |
|---|---|---|---|
| Full Debian-based | `node:22` | Many tools, easy debugging, broad compatibility | Large; more packages means more scanner findings and more to patch |
| Slim | `node:22-slim` | glibc-based, much smaller, still has a shell and package manager | Fewer tools; you install what you need |
| Alpine | `node:22-alpine` | Very small | Uses **musl** libc instead of glibc. Node.js lists musl builds as an **"Experimental"** support tier, and some native modules or prebuilt binaries expect glibc |
| Distroless | Google distroless images | No shell or package manager; small attack surface | Harder to debug; entrypoint scripts that need a shell won't work |
| Hardened images | Docker Hardened Images and similar vendor offerings | Minimal and maintained with a vulnerability focus | Availability, pricing and update policy depend on the provider |
| `scratch` | Static Go or Rust binaries | Nothing except your binary | Only for fully static artifacts; you must add CA certificates, time-zone data or a user yourself if you need them |

### How to decide **[Context]**

1. **Start from what the application needs at runtime**, not from the smallest available image.
2. **Check for native dependencies.** Packages such as `bcrypt`, `sharp`, `canvas`, `prisma` or `psycopg2` (non-binary) are evidence for a glibc base such as `-slim`. An application with only pure-JavaScript dependencies (like the Express API in these notes) can run on either.
3. **Consider how the team debugs.** Removing the shell is a real security benefit, but only if the team has another way to investigate running containers (logs, metrics, `docker debug`, Kubernetes ephemeral debug containers, or a sidecar such as `nicolaka/netshoot`).
4. **Measure.** If size matters to you (slow deploys, cold starts, registry cost), compare actual images rather than relying on rules of thumb.

**When the smallest image is the wrong choice:** when it breaks compatibility (musl issues), when it removes the tools your incident response depends on, or when nobody on the team can patch or rebuild it with confidence.

---

## 6. Writing maintainable Dockerfiles

### Instructions that matter most

| Instruction | Use it for | Notes |
|---|---|---|
| `# syntax=docker/dockerfile:1` | First line: use the latest stable 1.x Dockerfile frontend | Makes newer features (for example secret mounts with `env=`) available regardless of the builder's bundled version |
| `FROM image AS name` | Start a stage | Name stages so `COPY --from` and `--target` stay readable |
| `WORKDIR /app` | Set the working directory | Use absolute paths (the `WorkdirRelativePath` build check warns otherwise) |
| `COPY` | Copy files from the context or another stage | Prefer it over `ADD` |
| `ADD` | Remote URLs, Git sources, auto-extracting local archives | Use only when you need those features; pin remote downloads with `--checksum` |
| `RUN` | Execute commands at build time | Group commands that belong together |
| `USER` | Set the runtime user | Set it after steps that need root |
| `EXPOSE` | **Document** the listening port | It doesn't publish the port; `-p` or `ports:` does that |
| `CMD` / `ENTRYPOINT` | Define what runs at start | Use exec (JSON) form ([section 16](#16-runtime-behavior-cmd-entrypoint-signals-shutdown-and-health)) |
| `LABEL` | Metadata | Use standard OCI keys such as `org.opencontainers.image.source` |

### OS packages in one `RUN` **[Universal when you install OS packages]**

```dockerfile
# Careless: the update layer can be cached for months, so the install uses a stale index
RUN apt-get update
RUN apt-get install -y curl
```

```dockerfile
# Better: update and install together, skip recommended extras, clean up in the same layer
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl \
 && rm -rf /var/lib/apt/lists/*
```

If a `RUN` uses a pipe (`curl … | tar …`), add `set -o pipefail` (with a shell that supports it, such as bash) so that a failure on the left side of the pipe fails the build instead of being hidden.

### Readability

- Keep the Dockerfile as short as the decisions allow. Every extra stage or flag should have a reason someone can state.
- Comment only non-obvious lines: why a stage exists, why `slim` instead of `alpine`, why a package is installed.
- Run `docker build --check .` to apply Docker's built-in build checks without producing an image. [hadolint](https://github.com/hadolint/hadolint) is a widely used third-party linter for the same purpose.

---

## 7. Installing dependencies

### Lockfiles and lockfile-respecting installs **[Universal]**

**Decision:** install exactly what the lockfile specifies.

**Careless version:** `npm install` without a committed lockfile.

**Consequence:** two builds of the same commit can resolve different dependency versions. A transitive dependency can change between your test run and your release build.

**Better:** commit the lockfile and use the strict install command for your package manager:

| Ecosystem | Strict install |
|---|---|
| npm | `npm ci` (fails if `package.json` and `package-lock.json` disagree) |
| pnpm | `pnpm install --frozen-lockfile` |
| Yarn (Berry) | `yarn install --immutable` |
| Python (uv) | `uv sync --frozen` |
| Python (pip) | `pip install -r requirements.txt` with fully pinned versions, ideally with `--require-hashes` |
| Go | `go mod download` (versions come from `go.mod` / `go.sum`) |

### Production dependencies only **[Universal]**

Linters, test frameworks and type checkers are build-time tools. In the runtime image they add size and scanner findings without adding function.

```dockerfile
RUN npm ci --omit=dev
```

(`npm` also omits dev dependencies when `NODE_ENV=production` is set, but stating `--omit=dev` explicitly makes the intent visible.)

### Native dependencies **[Context]**

If a dependency compiles native code at install time, the build stage needs compilers and headers (`python3`, `make`, `g++` or `build-base`). Those tools should stay in the build stage. That is one of the clearest reasons to use a multi-stage build.

---

## 8. Multi-stage builds

A multi-stage build uses several `FROM` stages in one Dockerfile. Build stages contain the tooling; the final stage receives only the artifacts it needs through `COPY --from=<stage>`.

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
# e.g. TypeScript → dist/
RUN npm run build

FROM node:22-slim AS runtime
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER 1000:1000
CMD ["node", "dist/server.js"]
```

**What it buys you:** compilers, dev dependencies and source files stay out of the runtime image, which reduces size and the number of packages that scanners report and attackers could use. BuildKit can also build independent stages in parallel.

### When it is justified **[Context]**

| Evidence | Multi-stage? |
|---|---|
| A compile or bundle step (TypeScript, Vite/webpack, Go, Java, Rust) | Yes |
| Native modules that need compilers only at install time | Yes |
| Tests should run inside the image build in CI | A `test` stage is useful |
| Plain interpreted code with no build step and no native compilation | A single stage is often enough |

A stage that copies nothing different into the final image is just noise. Don't add stages to look "production-grade".

### Useful behaviors

- `docker build .` builds the **last** stage. `docker build --target test .` stops at the `test` stage.
- BuildKit **skips stages the target doesn't depend on**. If the runtime stage doesn't `COPY --from=test`, a normal `docker build .` does **not** run your tests. CI has to build the test target explicitly.

---

## 9. Image size versus security, compatibility and operability

Smaller images are usually better: faster pulls, faster scaling, fewer packages to patch. But size is a proxy, not the goal.

| Claim | Reality |
|---|---|
| "Smaller is more secure." | Often, because fewer packages means less attack surface and fewer findings. But a small image built on an unmaintained base, running as root with a leaked token in its history, is not secure. The security properties come from what is in the image and how it runs, not from the byte count. |
| "Distroless is always best." | It removes the shell and package manager, which is valuable. It also breaks shell entrypoint scripts and removes tools for live debugging. That's a fine trade when the team has another debugging path. |
| "Alpine is the obvious default." | Alpine is small, but musl can break native modules or prebuilt binaries that expect glibc. Test before choosing it. |
| "Fewer layers is always better." | Combining related steps matters (cleanup in the same `RUN`). Squeezing unrelated steps into one huge `RUN` hurts readability and cache reuse. |

**Practical rule:** apply the cheap baseline (slim or other minimal trusted base, `.dockerignore`, production dependencies only, multi-stage when there is a build step). Optimize beyond that only when you have measured a problem or a policy requires it.

---

## 10. Running as a non-root user

### Why it matters **[Universal]**

By default, a container process runs as root (UID 0) inside the container. Container isolation still applies, but if the application is compromised, a root process has far more room to act: it can modify files in the image, install tools, and make a container-escape vulnerability or a dangerous misconfiguration (for example a mounted Docker socket or host path) much more damaging.

**Careless version:** no `USER` instruction. `docker run --rm my-image whoami` prints `root`.

**Better:** use a dedicated user. Official Node.js images ship a `node` user (UID 1000). Other images may need one created:

```dockerfile
# Debian/Ubuntu-based images
RUN useradd --create-home --uid 10001 appuser
USER appuser
```

### Use a numeric UID when Kubernetes will enforce it **[Context]**

Kubernetes' `runAsNonRoot: true` checks the image's user before starting the container. If the image specifies a **name** (`USER node`), the kubelet can't verify that the name doesn't map to UID 0 and refuses to start the container unless the pod also sets `runAsUser`. Using the numeric form avoids that:

```dockerfile
# The "node" user and group in official Node.js images
USER 1000:1000
```

### When root may be justified

- A process that genuinely needs to bind ports below 1024 inside the container. Usually you can avoid it by listening on a high port and mapping it (`-p 80:8000`).
- An entrypoint that must adjust permissions on a mounted volume before dropping privileges (with a tool such as `gosu` or `su-exec`). Treat this as a deliberate pattern, not a default.

Runtime controls such as rootless Docker, user namespaces, dropped capabilities and `no-new-privileges` add further protection, but they are configured at runtime, not in the Dockerfile.

---

## 11. File ownership, permissions and the filesystem

### Who should own the application code? **[Context]**

A common pattern, used in my original notes and in many examples, is:

```dockerfile
COPY --chown=node:node . .
USER node
```

That makes the application files **writable by the user the application runs as**. If the application is compromised, the attacker can modify its own code and dependencies inside that container.

If the application doesn't need to write to its own files, leave them owned by root (readable by everyone, writable by nobody at runtime) and grant write access only where it is needed:

```dockerfile
# Owned by root: readable by the app user, not writable
COPY package.json server.js ./
# The only location the app user can write to
RUN mkdir -p /app/tmp && chown 1000:1000 /app/tmp
USER 1000:1000
```

`COPY --chmod` can set permissions in the same step (`COPY --chmod=0444 …`), without a separate `RUN chmod` layer.

**When `--chown` on the code is reasonable:** frameworks that write caches or compiled assets next to the code at runtime, or a development image where convenience matters more than hardening.

### Read-only root filesystem **[Optional, runtime]**

Running with `docker run --read-only` (or `readOnlyRootFilesystem: true` in Kubernetes) blocks writes to the image filesystem entirely. Applications that need scratch space then get a `tmpfs` or a volume for specific paths. It's a strong control, configured at runtime, and the Dockerfile should make it easy by keeping writable paths few and well known.

### Volumes and ownership

With Docker, an empty named volume mounted onto a path that exists in the image is initialized with that path's contents and ownership. A bind mount keeps the host's ownership and numeric IDs. Kubernetes volumes behave differently again (see `fsGroup` in the pod security context). Permission errors after switching to a non-root user usually come from here, not from the Dockerfile itself.

---

## 12. Secrets and credentials

### What goes wrong **[Universal]**

| Careless pattern | Consequence |
|---|---|
| `ARG NPM_TOKEN` used in a `RUN` | The value can appear in `docker history` and image metadata |
| `ENV API_KEY=…` | Stored in the image configuration; visible to anyone who can pull or inspect the image |
| `COPY .env .` or `COPY . .` without `.dockerignore` | The file is inside a layer |
| `RUN rm .npmrc` in a later step | The file is still in the earlier layer |

The official secrets documentation is explicit: build arguments and environment variables are inappropriate for passing secrets to a build, because they persist in the final image. Docker's `SecretsUsedInArgOrEnv` build check warns when an `ARG` or `ENV` name looks like a secret. Read build warnings.

### Demonstration (do not use for real secrets)

```dockerfile
# Dockerfile.leaky — for learning only
FROM node:22-slim
ARG NPM_TOKEN
WORKDIR /app
COPY package.json package-lock.json ./
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc \
 && npm ci --omit=dev \
 && rm .npmrc
```

```bash
docker build -f Dockerfile.leaky --build-arg NPM_TOKEN=dummy-123 -t api:leaky .
docker history --no-trunc api:leaky | grep dummy-123     # the token appears
```

Deleting `.npmrc` in the same `RUN` keeps the *file* out of the layer, but the `ARG` value is recorded with the build step.

### Better: secret mounts for build-time credentials

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm_token,env=NPM_TOKEN \
    echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > /tmp/.npmrc \
 && npm ci --omit=dev --userconfig /tmp/.npmrc \
 && rm -f /tmp/.npmrc
```

```bash
export NPM_TOKEN=…                                   # from your secret store, not your shell history
docker build --secret id=npm_token,env=NPM_TOKEN -t api:1.0.1 .
```

- By default a secret is mounted as a file at `/run/secrets/<id>`, only for that `RUN` step. `env=` exposes it as an environment variable for that step instead.
- `RUN --mount=type=ssh` with `docker build --ssh default` forwards an SSH agent for private Git dependencies.
- BuildKit has two predefined secrets, `GIT_AUTH_TOKEN` and `GIT_AUTH_HEADER`, for building from private Git repositories.
- Note that the command above writes the token to a temporary file *during* the step and deletes it before the step ends, so it never reaches a layer.

### Runtime secrets are a different problem

Database passwords and API keys the application needs **at runtime** should not be in the image at all. Inject them when the container starts: Docker/Compose secrets, Kubernetes Secrets (ideally backed by an external secret manager), or the cloud provider's secret service. An image should be safe to share with anyone who has registry access.

---

## 13. `ARG`, `ENV` and runtime configuration

| | `ARG` | `ENV` |
|---|---|---|
| Available during the build | Yes | Yes |
| Available in the running container | No | Yes |
| Visible in the image | Values used by `RUN` steps can appear in history | Yes, in the image configuration |
| Good for | Versions, build flags, selecting a base image | Genuine runtime defaults (`NODE_ENV=production`, `PYTHONUNBUFFERED=1`) |
| Never for | Secrets | Secrets |

### Build once, configure at deploy time **[Universal]**

**Careless version:** building separate images for staging and production with different URLs or feature flags baked in via `ARG`/`ENV`.

**Consequence:** the image you tested is not the image you deploy. Promoting a release means rebuilding, which reintroduces every source of drift.

**Better:** build one image per commit and supply environment-specific configuration at runtime (`-e`, `env_file`, Compose `environment`, Kubernetes ConfigMaps and Secrets). Keep `ENV` for defaults that are true everywhere.

**Scope detail:** an `ARG` declared before the first `FROM` is only usable in `FROM` lines. To use it inside a stage, declare it again (without a value) after that stage's `FROM`.

---

## 14. Pinning base images and dependencies

### What each reference actually guarantees

| Reference | Example | Same bytes every time? | Picks up patches on rebuild? |
|---|---|---|---|
| Floating major tag | `node:22-slim` | No | Yes, with `--pull` |
| Fully specified tag | `node:22.x.y-slim` (use a real version) | No. Even specific tags can be rebuilt and re-pushed (Docker Official Images are rebuilt when their base OS is updated), and stop receiving updates once newer releases exist | Only until that tag stops being maintained |
| Digest | `node:22-slim@sha256:<digest>` | Yes | No, until you change the digest |

Only a digest identifies exact content. The trade-off is that pinning by digest also **freezes security patches**.

### The decision **[Context]**

- **Digest pinning with update automation** (Renovate, or Dependabot with a `docker` ecosystem entry): the bot opens a pull request whenever the digest changes; CI rebuilds, tests and scans; you merge. You get reproducibility *and* patches. This is the strongest option when automation is in place.
- **Digest pinning without automation:** reproducible, but the image quietly falls behind on patches. This is a common way well-intended pinning makes images less safe over time.
- **Specific version tag plus scheduled `--pull` rebuilds:** a reasonable middle ground when there is no update bot yet. Builds are not byte-for-byte reproducible, but you choose the runtime version deliberately and keep receiving OS patches.

Keeping the tag next to the digest (`node:22-slim@sha256:…`) keeps the reference readable; the digest is what the builder actually uses.

### Application and OS dependencies

- Application dependencies: commit the lockfile and use the strict install command ([section 7](#7-installing-dependencies)).
- OS packages: pinning exact `apt` versions improves reproducibility but tends to break builds when the distribution removes old versions. Many teams pin the base image and accept the distribution's current patch versions instead. Choose deliberately.

---

## 15. Tagging, reproducibility and the supply chain

### Tags you deploy **[Universal]**

**Careless version:** build and deploy `my-api:latest`.

**Consequences:**

- You can't tell which code is running from the tag.
- Rolling back to "the previous `latest`" is guesswork, because the tag has already moved.
- Nodes that cached an old `latest` and nodes that pulled a new one can run different code under the same name.

**Better:**

- Tag images with something traceable: a release version and/or the Git commit SHA (`my-api:1.4.2`, `my-api:sha-3f9c2ab`).
- Treat release tags as immutable (many registries can enforce this).
- For the strongest guarantee, **deploy by digest**, which is what the orchestrator will actually pull.
- `latest` can still exist as a convenience pointer for humans. It just shouldn't be what production references.

### Labels

OCI labels make an image traceable back to its source:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/<owner>/<repo>" \
      org.opencontainers.image.revision="<git-sha>"
```

In CI, tools such as `docker/metadata-action` generate these labels and tags automatically, which avoids hard-coding them.

### Reproducibility is a spectrum

Pinning the base image by digest, committing lockfiles and building in CI from a clean checkout gets most teams most of the value. Fully bit-for-bit reproducible images need more (for example controlled timestamps via `SOURCE_DATE_EPOCH`) and are rarely the first priority.

---

## 16. Runtime behavior: `CMD`, `ENTRYPOINT`, signals, shutdown and health

A Dockerfile defines how the process starts, and therefore how it stops. This is where many images that "work" fail during deployments.

### Exec form versus shell form **[Universal]**

```dockerfile
# Shell form: runs via /bin/sh -c "node server.js"
CMD node server.js

# Exec form: runs node directly
CMD ["node", "server.js"]
```

When `docker stop` (or Kubernetes, during a rollout) stops a container, it sends `SIGTERM` to the container's main process (PID 1), waits for a grace period (10 seconds by default in Docker, 30 seconds by default in Kubernetes), then sends `SIGKILL`.

With shell form, the main process is a shell, and Docker's documentation notes that a shell parent doesn't pass signals to the application. The result is an application that never learns it should stop: in-flight requests are cut off by `SIGKILL`, and every stop takes the full grace period. Some shells replace themselves with the command when the string is a single simple command, so the exact outcome varies by base image and shell. That variability is the reason to use exec form instead of relying on it. Docker's `JSONArgsRecommended` build check flags shell-form `CMD`/`ENTRYPOINT`.

**Wrappers have the same problem.** `CMD ["npm", "start"]` puts npm between Docker and Node. Run the runtime directly (`["node", "server.js"]`). If you need an entrypoint script, end it with `exec "$@"` (or `exec node server.js`) so the application replaces the shell.

### `ENTRYPOINT` and `CMD` together

```dockerfile
ENTRYPOINT ["node"]
CMD ["server.js"]
# docker run img            → node server.js
# docker run img --version  → node --version
```

`ENTRYPOINT` is the fixed executable; `CMD` supplies default arguments that are easy to override. For most application images, `CMD` alone is enough.

### Exec form is necessary but not sufficient: PID 1 **[Context]**

Linux treats PID 1 specially: a signal that the process hasn't installed a handler for doesn't trigger the default action. So a process running as PID 1 **without** a `SIGTERM` handler ignores `SIGTERM` and is killed only after the grace period. The Node.js Docker image documentation states that Node.js was not designed to run as PID 1.

Two valid approaches:

1. **Handle the signal in the application** and shut down gracefully:

   ```js
   process.on("SIGTERM", () => {
     server.close(() => process.exit(0));         // stop accepting, finish in-flight requests
     setTimeout(() => process.exit(1), 8000).unref(); // give up before the grace period ends
   });
   ```

2. **Add a minimal init process** that forwards signals and reaps zombie processes: `docker run --init`, `init: true` in Compose, or `tini` as the `ENTRYPOINT`. This matters most when the application spawns child processes.

Also check that the application stops on the signal the platform sends. If an application only reacts to `SIGINT` or `SIGQUIT`, set `STOPSIGNAL` in the Dockerfile accordingly.

### Health checks **[Context]**

`HEALTHCHECK` tells Docker how to test whether the container is healthy. Docker records the status; Compose can use it for `depends_on: condition: service_healthy`.

Important limits:

- **Kubernetes ignores the Dockerfile `HEALTHCHECK`.** It uses its own liveness, readiness and startup probes defined in the pod spec.
- Plain Docker marks a container `unhealthy` but does not restart it because of that; restart policies react to the process exiting. (Docker Swarm services do replace unhealthy tasks.) Recovery is the job of the orchestrator or platform.
- A health check needs a tool inside the image to run it. Installing `curl` just for a health check adds a package; on a Node image you can use Node itself:

  ```dockerfile
  HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD ["node", "-e", "fetch('http://127.0.0.1:8000/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"]
  ```

- A health endpoint should check what the process can actually fix by restarting. A liveness check that fails whenever the database is down can turn a database outage into a restart storm.

### Bind to all interfaces

An application that listens on `127.0.0.1` inside a container is unreachable from outside the container, even with `-p`. Listen on `0.0.0.0` (or `::`) and let the network configuration decide exposure.

---

## 17. BuildKit features worth knowing

| Feature | What it does | When to use it |
|---|---|---|
| `RUN --mount=type=secret` | Makes a secret available to one step without storing it in a layer | **[Universal]** whenever the build needs credentials |
| `RUN --mount=type=ssh` | Forwards an SSH agent for one step | Private Git dependencies |
| `RUN --mount=type=cache,target=…` | Persists a package manager cache **between builds**; not stored in the image | **[Optional]** when dependency downloads are measurably slow and builds are frequent |
| `RUN --mount=type=bind,target=…` | Makes context files available to one step without copying them into a layer | Compiling from source when only the output should be kept |
| `--target <stage>` | Builds up to a named stage | Test and debug stages |
| `--platform` / `buildx build --platform a,b` | Builds for other CPU architectures, producing one index | **[Context]** when the image runs on more than one architecture |
| `--cache-to` / `--cache-from` | Exports and imports build cache (registry, GitHub Actions cache, …) | CI runners without a persistent local cache |
| `docker build --check` | Runs build checks without building | Every change, ideally in CI |

### Cache mounts

```dockerfile
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev
```

The cache path must match where the package manager stores its cache for the user running the step (here root, because no `USER` has been set yet). On CI runners that start from scratch, a cache mount helps only if the builder or its cache is persisted.

### Multi-platform images

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t <registry>/<repo>:1.0.0 --push .
docker buildx imagetools inspect <registry>/<repo>:1.0.0
```

- An image built only for `amd64` fails on an `arm64` host (for example Apple Silicon or ARM cloud instances) with `exec format error`.
- Non-native platforms build under QEMU emulation, which can be slow. Native builders or cross-compilation (common in Go) are faster.
- Storing multi-platform images locally needs the containerd image store. It is the default for new installations of Docker Engine 29 and recent Docker Desktop versions; existing installations may need to opt in. Otherwise, push directly to a registry.

---

## 18. Scanning, SBOMs, provenance and verification

A Dockerfile controls what goes into an image. It doesn't tell you whether those contents have known vulnerabilities, and it doesn't prove where the image came from.

### Vulnerability scanning **[Universal]**

Scan images in CI and on a schedule, because new vulnerabilities are published against packages you already ship. Common tools include Docker Scout, Trivy and Grype.

How to treat the results:

- A scan lists **known** vulnerabilities in **detected** packages. A clean scan is not proof that an image is secure.
- Not every finding is exploitable in your context. Triage by severity, whether a fix exists, and whether the vulnerable component is reachable, and record decisions instead of silently ignoring findings.
- Rebuilding on a patched base image fixes many findings without any code change, which is another reason to rebuild regularly.

### SBOMs and provenance **[Context]**

- An **SBOM** (software bill of materials) lists the packages in an image. With BuildKit you can attach one as an attestation: `docker buildx build --sbom=true …`.
- **Provenance** records how the image was built. BuildKit adds a minimal provenance attestation (`mode=min`) by default when the build driver and image store support attestations; `--provenance=mode=max` records more detail.
- **Signing** (for example with Sigstore `cosign`) and admission policies let a cluster refuse images that were not built and signed by your pipeline.

These matter most when you publish images, operate in regulated environments, or need to answer "exactly what is running, and who built it?" during an incident.

### Verifying a built image

```bash
docker build --check .                                        # build checks only
docker build -t api:check .
docker run --rm --entrypoint id api:check                     # expect a non-zero UID
docker history --no-trunc api:check | grep -iE 'token|secret|password'   # expect no output
docker image ls api:check                                     # record the size
docker run -d --name api-check -p 127.0.0.1:8000:8000 api:check
curl -fsS http://127.0.0.1:8000/api/health
time docker stop api-check                                    # should be well under the grace period
docker rm api-check
```

The `grep` over history is a quick sanity check for obvious mistakes, not a secret scanner. Dedicated tools exist for that.

---

## 19. Common Dockerfile mistakes and their consequences

| Mistake | What actually happens | Scope |
|---|---|---|
| Large base image chosen by default | Slower pulls and scaling, more packages to patch and more scanner noise | **[Context]** (sometimes the full image is justified for compatibility or debugging) |
| Build tools and dev dependencies in the runtime image | More size and attack surface; compilers available to an attacker | **[Universal]** when there is a build step |
| `COPY . .` without `.dockerignore` | `.env`, keys and `.git` history inside the image; host `node_modules` overwrites installed ones | **[Universal]** |
| Source copied before dependency installation | Every code change reinstalls all dependencies | **[Universal]** |
| No lockfile or a non-strict install | Two builds of one commit can contain different dependency versions | **[Universal]** |
| Process runs as root without a reason | A compromise gets root inside the container; escapes and misconfigurations become more damaging | **[Universal]** |
| Secrets in `ARG`, `ENV` or copied files | Credentials recoverable from history, metadata or layers by anyone with pull access | **[Universal]** |
| Deleting files in a later `RUN` | Image doesn't shrink; deleted secrets remain recoverable | **[Universal]** |
| Build-time and runtime configuration confused | Different images per environment, or settings that silently don't apply at runtime | **[Universal]** |
| Deploying `latest` | No traceability; unreliable rollbacks; nodes running different code under one name | **[Universal]** |
| Digest pinning without update automation | Reproducible builds that stop receiving security patches | **[Context]** |
| Assuming smaller means secure | Attention goes to size while root users, leaked secrets or unpatched bases remain | **[Universal]** |
| Shell-form `CMD` or `npm start` as the main process | `SIGTERM` may never reach the app; slow stops and dropped requests on every deploy | **[Universal]** |
| No signal handling or init when running as PID 1 | The process ignores `SIGTERM` and is killed after the grace period | **[Context]** |
| Using `HEALTHCHECK` as if it restarted containers, or relying on it in Kubernetes | False confidence; Kubernetes ignores it and plain Docker doesn't restart unhealthy containers | **[Context]** |
| Application binds to `127.0.0.1` | Container runs but is unreachable | **[Universal]** |
| Image built for one architecture, deployed on another | `exec format error` at start-up | **[Context]** |
| Over-optimizing size (distroless, `scratch`, aggressive stripping) without a reason | Broken entrypoints, missing CA certificates or time zones, no way to debug during an incident | **[Context]** |
| Treating a successful build as production readiness | Problems surface during deployment or incidents instead of review | **[Universal]** |
| Assuming the Dockerfile provides application security | Injection flaws, authentication bugs, network exposure and runtime privileges are outside the Dockerfile's control | **[Universal]** |

---

## 20. A worked example with trade-offs

### The application

A small Express API with one production dependency (`express`), one dev dependency (`eslint`), a test script (`node --test test/`), a health route at `/api/health`, and a `SIGTERM` handler. It listens on `0.0.0.0` and reads `PORT` (default `8000`).

```
api/
├── package.json
├── package-lock.json
├── server.js
└── test/
    └── smoke.test.js
```

### Where it started

```dockerfile
FROM node:22
WORKDIR /app
COPY . .
RUN npm install
EXPOSE 8000
CMD npm start
```

It builds and it runs. It also has seven problems: a full base image, source copied before dependencies, no `.dockerignore`, dev dependencies installed, shell form plus npm in front of Node, a root user, and a floating tag with no stated update strategy.

### Where it ended

```dockerfile
# syntax=docker/dockerfile:1

# One place to pin the runtime. Use the exact version you test on, and add
# @sha256:<digest> once an update bot (Renovate/Dependabot) keeps it current.
FROM node:22-slim AS base
WORKDIR /app

# Build/test dependencies, including devDependencies
FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci

# Tests: CI runs `docker build --target test .`. A plain `docker build .` skips this stage.
FROM deps AS test
COPY . .
RUN npm test

# Production dependencies only
FROM base AS prod-deps
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# Runtime image
FROM base AS runtime
ENV NODE_ENV=production
COPY --from=prod-deps /app/node_modules ./node_modules
COPY package.json server.js ./
# Numeric form of the image's built-in "node" user, so Kubernetes runAsNonRoot can verify it.
# Files above stay root-owned: the app can read its code but not modify it.
USER 1000:1000
EXPOSE 8000
CMD ["node", "server.js"]
```

```gitignore
# .dockerignore
.git
.gitignore
node_modules
npm-debug.log*
.env
.env.*
*.pem
*.key
coverage
test-results
.vscode
.idea
Dockerfile*
.dockerignore
```

### Decisions and trade-offs

| Decision | Why | Trade-off or alternative |
|---|---|---|
| `node:22-slim` | Trusted official image; glibc avoids musl issues if a native dependency is added later | `node:22-alpine` is smaller and works for this pure-JavaScript app; Node.js lists musl builds as "Experimental", so test before switching. Choose the Node.js release line your application supports, and plan upgrades before it reaches end of life. |
| One `base` stage | The runtime version is pinned in one line | None significant |
| Separate `deps`, `test` and `prod-deps` stages | Tests run with dev dependencies; the runtime receives production dependencies only | Three stages for a tiny app is borderline. Without the CI `--target test` requirement, `deps` and `test` could be dropped. |
| Copy only `package.json`, `server.js` and `node_modules` into the runtime | Tests, docs and local files never reach production | New runtime files must be added to the `COPY` line explicitly. That's intentional. |
| Root-owned code, `USER 1000:1000` | Non-root process that can't modify its own code | If the app later needs to write files, create a specific writable directory ([section 11](#11-file-ownership-permissions-and-the-filesystem)) |
| `CMD ["node", "server.js"]` | Node receives `SIGTERM` directly | Safe here because `server.js` handles `SIGTERM`. Without a handler, add `--init` / `init: true` ([section 16](#16-runtime-behavior-cmd-entrypoint-signals-shutdown-and-health)). |
| `ENV NODE_ENV=production` only | A genuine runtime default | `PORT` is read with a default in code and set at deploy time if needed |

### Deliberately not used (and when to add them)

| Technique | Why not here | Add it when |
|---|---|---|
| Digest pinning | No update automation configured yet | Renovate or Dependabot is set up for the Dockerfile |
| Secret mount | All dependencies come from the public registry | A private registry or private Git dependency is added ([section 12](#12-secrets-and-credentials)) |
| Cache mount | No measured build-time problem | Dependency downloads dominate build time in CI |
| `HEALTHCHECK` | The target platform isn't decided; Kubernetes would ignore it | The image runs under Docker Compose and something depends on its health |
| Distroless runtime | The team's debugging workflow still relies on a shell | Debugging moves to logs, metrics and ephemeral debug containers, or policy requires it |
| Multi-platform build | Only one target architecture | The image must run on both `amd64` and `arm64` |
| Removing npm from the runtime image | npm isn't used at runtime but ships with the base image and can appear in scan results | Findings in npm's bundled packages become a recurring triage cost |

### Verification status

The commands in [section 18](#18-scanning-sboms-provenance-and-verification) are the checks this image should pass. Record the results (image size, build time, stop time, scan summary) next to the Dockerfile when you run them. Numbers you measured yourself are worth more than any rule of thumb in these notes.

---

## 21. Using AI to generate and review Dockerfiles

### Why "Write a Dockerfile for my application" is not enough

With a request that short, an assistant has to guess most of what determines whether the Dockerfile is right. If it can't inspect the repository, it may assume:

- the language, framework and runtime version;
- the package manager, and whether a lockfile exists;
- the build command, and whether native dependencies need compilers;
- the start command, the port and the bind address;
- which files the application needs at runtime;
- whether a multi-stage build is needed;
- the runtime user and which paths must be writable;
- how configuration and secrets are supplied, at build time and at runtime;
- how the process handles signals and health checks;
- the target platform (Docker Compose, Kubernetes, a managed service) and CPU architecture;
- the team's requirements for size, security and reproducibility.

The result usually *looks* reasonable and often builds. Wrong guesses show up later: a Node version the application doesn't support, `npm start` as PID 1, a missing native dependency, an `amd64`-only image on ARM hosts, or a token passed as a build argument.

This is not an argument that AI can't write good Dockerfiles. Output quality depends on the context, requirements and constraints it receives, and on what is verified afterwards.

### Three different levels of confidence

| Level | What it means | Evidence required |
|---|---|---|
| **1. It builds** | The instructions execute | A successful build log |
| **2. It runs** | The container starts, serves requests, runs as non-root and stops cleanly | Run, request, `id`, and stop-time checks |
| **3. It meets the production requirements** | Evaluated against the target platform, security policy and operational needs | Scan results, secrets check, review of decisions, behavior in a staging environment |

Each level needs its own evidence. A response that only demonstrates level 1 shouldn't be described as "production-ready".

### What a useful request includes

| Category | Examples of what to provide |
|---|---|
| Application context | Language, framework, project layout, monorepo service path, application type (API, worker, static site) |
| Dependency management | Package manager, lockfile, private registries |
| Build process | Build command, output directory, native modules, required build tools |
| Runtime requirements | Start command, port, bind address, runtime files, OS packages |
| Image constraints | Preferred or mandated base images, target architectures, size constraints |
| Security | Non-root requirement, build-time and runtime secret handling, writable paths |
| Reproducibility | Version pinning policy, whether update automation exists |
| Operational behavior | Signal handling, shutdown time, health endpoint, logging to stdout/stderr |
| Deployment context | Docker Compose, Kubernetes, ECS, Cloud Run, …, and the constraints they impose |
| Verification | Which checks to run: build, tests, scan, run-time checks |

A compact version of such a request:

> Create a Dockerfile and `.dockerignore` for the Node.js API in `api/`. Use the Node version in `.nvmrc` and install with `npm ci` from the committed lockfile. There is no build step; the entry point is `server.js`, listening on `0.0.0.0:8000`. It deploys to Kubernetes on arm64 nodes with `runAsNonRoot: true`, so use a numeric non-root UID. No private registries. Keep dev dependencies out of the runtime image. Explain each decision, list what you chose not to add and why, and give me the commands to verify the result.

Good prompting defines requirements. It doesn't replace validation.

### A responsible AI-assisted workflow

1. **Inspect the project first:** manifests, lockfiles, runtime version files, start scripts, existing container and CI configuration.
2. **Generate** the Dockerfile and `.dockerignore` from that evidence.
3. **Explain** each decision and each deliberate omission, and label assumptions.
4. **Verify:** build, run, check the user, stop time and history, scan.
5. **Review** like any other code change, by someone who understands the deployment environment.

### How I packaged this workflow as a reusable skill

Writing the full set of requirements in every prompt is repetitive, so I packaged my Dockerfile standards into a reusable agent skill, `dockerfile-right-sizer` (see [`skills/dockerfile-right-sizer/SKILL.md`](../skills/dockerfile-right-sizer/SKILL.md)). What it instructs the assistant to do:

- **Gather evidence before deciding:** inspect lockfiles, runtime version files, build and start scripts, the listening port and bind address, native dependencies, build-time credentials, architecture hints, update automation, existing container files and health endpoints.
- **Apply a baseline every time:** minimal trusted base with a specific version tag, `.dockerignore`, manifests copied before source, lockfile-respecting installs, production dependencies only, non-root user, exec-form `CMD`, no secrets in `ARG`/`ENV`.
- **Add extras only with evidence:** multi-stage builds, test stages, Alpine vs slim, distroless, digest pinning, secret and cache mounts, multi-platform builds, `HEALTHCHECK`, an init process and OCI labels each have a decision question and a default for when evidence is missing.
- **Ask few questions:** at most three, batched, and only when the answer isn't in the repository and would change the Dockerfile.
- **Verify when possible:** build, confirm a non-root user, check that the port responds, inspect history for secrets, record size and build time, and run `hadolint` if available. If Docker isn't available, list the commands instead of claiming verification.
- **Report the reasoning:** evidence found, decisions, techniques deliberately not used (and when to add them), assumptions, and issues found outside the Dockerfile.

What it doesn't do: it doesn't scan images for vulnerabilities, generate SBOMs or sign images; it produces only the Dockerfile and `.dockerignore` (no Compose, CI or Kubernetes files unless asked); its reference patterns cover Node.js, Python, Go and Java; and its output still needs the verification and review described above.

---

## 22. References

**Official Docker documentation**

- [Building best practices](https://docs.docker.com/build/building/best-practices/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Build cache: optimize cache usage](https://docs.docker.com/build/cache/optimize/)
- [Build context and `.dockerignore`](https://docs.docker.com/build/building/context/)
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Build secrets](https://docs.docker.com/build/building/secrets/)
- [Build checks](https://docs.docker.com/reference/build-checks/) · [JSONArgsRecommended](https://docs.docker.com/reference/build-checks/json-args-recommended/) · [SecretsUsedInArgOrEnv](https://docs.docker.com/reference/build-checks/secrets-used-in-arg-or-env/)
- [Multi-platform builds](https://docs.docker.com/build/building/multi-platform/)
- [Build attestations (SBOM and provenance)](https://docs.docker.com/build/metadata/attestations/)
- [Cache storage backends](https://docs.docker.com/build/cache/backends/)
- [Docker Engine v29 release announcement (containerd image store default)](https://www.docker.com/blog/docker-engine-version-29/)

**Runtime and image documentation**

- [Node.js Docker image README](https://github.com/nodejs/docker-node) (image variants, Alpine/musl support tier) and [Docker and Node.js best practices](https://github.com/nodejs/docker-node/blob/main/docs/BestPractices.md) (PID 1, `--init`, non-root user)
- [Google distroless images](https://github.com/GoogleContainerTools/distroless)
- [Kubernetes: configure a security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes: liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

**Further reading and tools**

- [Hynek Schlawack: Why your Dockerized application isn't receiving signals](https://hynek.me/articles/docker-signals/)
- [tini](https://github.com/krallin/tini) · [hadolint](https://github.com/hadolint/hadolint) · [dive](https://github.com/wagoodman/dive)
- [Trivy](https://github.com/aquasecurity/trivy) · [Grype](https://github.com/anchore/grype) · [Docker Scout](https://docs.docker.com/scout/) · [Sigstore cosign](https://github.com/sigstore/cosign)

> **Sourcing note:** cache behavior, secret mounts, build checks, attestations, `--pull`/`--no-cache` and mutable tags come from the official Docker documentation above. Alpine/musl support status and the PID 1 guidance for Node.js come from the Node.js Docker image documentation. Recommendations labeled **[Context]** are engineering judgment, not universal rules.

---

## Appendix A: production Dockerfile review checklist

**Contents of the image**

- [ ] Base image comes from a trusted source and fits the runtime's needs (glibc vs musl, shell needed or not).
- [ ] `.dockerignore` excludes `.git`, dependency folders, `.env*`, keys and local build output.
- [ ] The final stage copies only what the runtime needs.
- [ ] Build tools and dev dependencies are not in the runtime image.
- [ ] Dependencies are installed from a committed lockfile with a strict install command.

**Build efficiency**

- [ ] Dependency manifests are copied and installed before application source.
- [ ] Related `RUN` commands are combined, with caches cleaned in the same step.
- [ ] Multi-stage, cache mounts and other extras are each justified by evidence.

**Security**

- [ ] The process runs as a non-root user (numeric UID if Kubernetes enforces `runAsNonRoot`).
- [ ] Application code is not writable by the runtime user unless there is a reason.
- [ ] No secrets in `ARG`, `ENV`, copied files or image history; build-time credentials use secret or SSH mounts.
- [ ] Runtime secrets are injected at deploy time, not baked into the image.
- [ ] The image is scanned in CI and on a schedule, and findings are triaged.

**Reproducibility and supply chain**

- [ ] The base image version is deliberate, with a stated update strategy (digest + bot, or tag + scheduled `--pull` rebuilds).
- [ ] Deployed images use traceable, immutable tags or digests, not `latest`.
- [ ] OCI labels (source, revision) are set, ideally by CI.
- [ ] SBOM, provenance and signing requirements are known and met where they apply.

**Runtime behavior**

- [ ] `CMD`/`ENTRYPOINT` use exec form and start the runtime directly.
- [ ] The application handles `SIGTERM` (or runs under an init process) and shuts down within the grace period.
- [ ] The application listens on `0.0.0.0` on the documented port.
- [ ] Health checks are defined where the platform actually uses them (Kubernetes probes vs Docker `HEALTHCHECK`).
- [ ] The image is built for every CPU architecture it will run on.

**Verification evidence**

- [ ] Build checks pass (`docker build --check .`).
- [ ] The container starts, responds, runs as non-root and stops cleanly; results are recorded.
- [ ] Someone who understands the deployment environment has reviewed the decisions.
