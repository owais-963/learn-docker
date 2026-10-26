# learn-docker

Engineering notes and reusable tools from my in-depth study of Docker. The focus is on the decisions behind container images and runtime behavior, not just the commands.

## Contents

| Path | What it is |
|---|---|
| [`images/docker-images-and-production-dockerfiles.md`](images/docker-images-and-production-dockerfiles.md) | Docker images and production Dockerfiles: layers, build cache, base images, multi-stage builds, non-root users, secrets, pinning, signals and shutdown, scanning, common mistakes, a worked example and a review checklist |
| [`skills/dockerfile-right-sizer/SKILL.md`](skills/dockerfile-right-sizer/SKILL.md) | A reusable agent skill that writes or reviews Dockerfiles from repository evidence: a baseline applied every time, extras only when evidence justifies them, and a report of every decision, omission and assumption |

More topics (starting with Docker networking) will be added as the series continues.

## Using the skill

`SKILL.md` follows the Agent Skills format (YAML front matter with `name` and `description`, followed by instructions). Add the `dockerfile-right-sizer` folder to the skills location your AI tool supports, then ask it to create, review or harden a Dockerfile.

The skill helps an assistant gather the right context and explain its decisions. It does not scan images, generate SBOMs or guarantee production readiness: build, run, scan and review the result before you rely on it.

## Feedback

Corrections and suggestions are welcome. Open an issue or a pull request.

## License

[MIT](LICENSE)
