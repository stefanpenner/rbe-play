# rbe-play

Tiny end-to-end Bazel remote execution using Bazel's built-in worker. No Docker, no scheduler, no registration.

See [TUTORIAL.md](TUTORIAL.md) for the step-by-step from irreducible core to working worker.

## Diagram

```
+----------------+     gRPC      +------------------+
| Bazel client   | ------------> | remote worker    |
| (this repo)    | <------------ | (Bazel source)   |
| localhost      |   results     | localhost:8080   |
+----------------+               +------------------+
```

## Setup (one-time)

Clone Bazel source and build the worker:

```bash
git clone --depth=1 https://github.com/bazelbuild/bazel.git
cd bazel
bazel build //src/tools/remote:worker
mkdir -p /tmp/rbe/work /tmp/rbe/cas
```

## Run the worker

```bash
# from the bazel checkout
bazel-bin/src/tools/remote/worker \
  --work_path=/tmp/rbe/work \
  --cas_path=/tmp/rbe/cas \
  --listen_port=8080
```

## Build remotely

In this repo:

```bash
bazel build //:hello --config=remote
```

Output should contain `1 remote`.

## What it is

Single process that implements the Remote Execution API. Bazel client sends actions directly to it. No platforms, no pools, no multi-worker scheduling.
