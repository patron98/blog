---
title: "Our CI Failed in 2 Seconds With No Error — So We Moved It to a Raspberry Pi"
date: 2026-08-03
tags: ["ci", "github-actions", "raspberry-pi", "self-hosted", "arm64"]
summary: "GitHub Actions burned our entire monthly budget emulating ARM builds, then started refusing jobs in a way that looked exactly like broken code. The fix: three runner containers on the same Raspberry Pi the images deploy to. Builds went from an hour to minutes, and the bill went to zero."
---

Our trading lab deploys everything to a Raspberry Pi 5, which means every
Docker image we ship is **arm64**. Our CI ran on GitHub Actions' hosted
runners, which are **amd64**. If you've never thought about what that
combination costs, this post is for you.

## The mystery: a 2-second failure with no failing step

One evening, every build across three repos started failing. The runs
completed in about **two seconds**, with the `test` job marked failed —
but no step inside it had failed. No logs, no red step, nothing to click.

If you've trained yourself to read CI failures, this pattern is deeply
disorienting. A compile error takes minutes and names a file. A flaky test
names a test. This named *nothing* and took *seconds*. We spent a while
staring at the diff of the last commit (which was innocent) before pulling
the job metadata from the API:

> "The job was not started because recent account payments have failed or
> your spending limit needs to be increased."

Not a code problem. A billing problem. We had burned through the entire
monthly included-usage budget for private repos — and GitHub's UI presents
that as a *failed workflow run*, visually identical to broken code.

**Lesson one: a CI run that fails in seconds with no failing step is an
account problem, not a code problem.** Check billing before you check the
diff. We've since written that into our runbook, because the failure mode
*will* fool the next person too.

## Where the money actually went

The budget didn't vanish because we build a lot. It vanished because of
**QEMU emulation**. Building arm64 images on amd64 hosted runners means
every instruction of every compile runs through a CPU emulator. Our
Python gateway image — which compiles numpy and pandas from source during
the build — took roughly **an hour** per build under emulation. The Java
bot image wasn't far behind.

Emulated builds are a silent cost multiplier: you pay for every emulated
minute at the normal per-minute rate, and arm64-under-QEMU routinely runs
5–10× slower than native. We were effectively paying a 10× tax to build
images for a computer that was sitting in the living room, idle, already
running the target architecture natively.

Once you phrase it like that, the fix names itself.

## Three runner containers on the deploy target

We run three self-hosted GitHub Actions runners — one per repo, because
personal accounts don't support account-level runners — as Docker
containers on the Pi itself:

```yaml
x-runner: &runner
  image: myoung34/github-runner:ubuntu-noble
  restart: unless-stopped
  mem_limit: 512m
  environment: &runner-env
    ACCESS_TOKEN: ${GH_RUNNER_PAT}
    RUNNER_SCOPE: repo
    LABELS: self-hosted,linux,ARM64,pi
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```

The workflow change is one line per repo:

```yaml
  build-and-push:
    needs: test
    runs-on: [self-hosted, linux, ARM64]   # was: ubuntu-latest
```

The runner container itself is tiny; the heavy lifting (the actual
`docker build`) happens in the host's Docker daemon via the mounted
socket, natively on arm64.

Results, same day:

| | Before (hosted + QEMU) | After (Pi, native) |
|---|---|---|
| Gateway image build | ~60 min | **~4 min** |
| Java bot image build | ~15 min | **~6 min** |
| Monthly Actions cost | entire budget | **$0** |

The first native build finished before we managed to open the Actions tab
to watch it.

## What we deliberately did *not* move

Two decisions mattered as much as the migration itself:

**Test jobs stay on GitHub-hosted runners.** Our Java test suite spins up
Postgres via Testcontainers and eats several GB of RAM. The Pi also runs
the production trading bot. Letting CI tests fight production for memory
on the same box is how you turn a green build into a trading incident.
Tests are amd64-agnostic and cheap in hosted minutes — the expensive part
was only ever the emulated *builds*. Splitting the workflow this way keeps
both halves in their cheapest, safest place.

**Public repos never touch these runners.** A docker-socket mount is
root-equivalent on the host. For private repos with no fork PRs, that's an
acceptable, well-understood risk. For a public repo — where any drive-by
pull request could run workflow code — it would be handing strangers root
on the machine that runs our trading bot. This blog, ironically, is built
from a public repo: it deploys via GitHub's hosted runners, which are free
for public repos anyway.

**Lesson two: self-hosted runners are a security decision first and a
cost decision second.** The threat model changes completely with repo
visibility. Decide per repo, not globally.

## The part nobody tells you

Registering the runners took ten minutes. The things that actually cost
time:

- **The PAT dance.** A fine-grained token needs the *Administration:
  read & write* repository permission to register runners — not an
  obvious name for "may add CI machines."
- **Knowing about the billing failure mode** (see lesson one). If we'd
  known, we'd have saved an hour of debugging a phantom code problem.
- **Accepting that "enterprise CI" can be a €80 single-board computer.**
  The Pi builds our images faster than GitHub's datacenter did, because
  native beats emulated by more than a datacenter beats a living room.

*The Pi in question also runs the trading bot, a Postgres instance, a
Prometheus stack, and a media server. It is, by a comfortable margin, the
hardest-working €80 in the house.*
