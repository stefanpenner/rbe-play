# rbe-play

Tiny end-to-end Bazel RBE. Local workers register automatically.

## Run

```bash
docker compose up -d
# wait ~10s for worker to register
bazel build //:hello --config=remote
```

You should see `1 remote` in the process summary.

## How it works

- `docker compose` starts Redis + Buildfarm server + worker
- Worker registers with the server on startup (advertises platform properties)
- Bazel client (on your machine) sends actions to `grpc://localhost:8980`
- Server matches actions to the registered worker via `exec_properties`

## Platforms

See `platforms/BUILD`. The Linux platform matches the local worker.

To add more platforms later: start another worker with different properties, add a matching `platform()`.
