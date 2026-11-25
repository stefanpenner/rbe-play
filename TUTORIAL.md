# Step-by-step: from irreducible core to working Bazel worker

Goal: understand the absolute minimum a worker must do, then end with a worker Bazel can use.

## Irreducible core

Bazel (client) and worker speak the Remote Execution API over gRPC.

Minimal services a worker must implement for Bazel to execute a simple action:

1. **Capabilities.GetCapabilities**  
   Bazel queries this first. Return at least:
   - `cache_capabilities` with SHA-256 digest function
   - `execution_capabilities` with `exec_enabled: true`

2. **ContentAddressableStorage** (CAS)
   - `FindMissingBlobs`
   - `BatchReadBlobs` / `BatchUpdateBlobs`
   - Plus **ByteStream** `Read` / `Write` for blobs larger than the batch limit  
     Resource names:
     - upload: `{instance}/uploads/{uuid}/blobs/{hash}/{size}`
     - download: `{instance}/blobs/{hash}/{size}`

3. **ActionCache**
   - `GetActionResult` (cache lookup)
   - `UpdateActionResult` (store result)

4. **Execution.Execute**
   - Receives `ExecuteRequest` containing `action_digest`
   - Fetches `Action` and `Command` protos from CAS
   - Materializes the input root (directory tree) from CAS into a temp dir
   - Runs the command
   - Uploads outputs + stdout/stderr to CAS
   - Returns a long-running `Operation` whose `response` is an `ExecuteResponse` containing `ActionResult`

Everything else (sandboxing, platforms, priority, WaitExecution, GetTree, etc.) is optional for the tiniest working case.

## Step 1: Confirm the protocol with the official worker

Clone Bazel and run its built-in worker (this is the reference implementation of the core):

```bash
git clone --depth=1 https://github.com/bazelbuild/bazel.git
cd bazel
bazel build //src/tools/remote:worker
mkdir -p /tmp/rbe/{work,cas}
bazel-bin/src/tools/remote/worker \
  --work_path=/tmp/rbe/work \
  --cas_path=/tmp/rbe/cas \
  --listen_port=8080
```

In this repo (another terminal):

```bash
bazel build //:hello --config=remote
```

You should see `1 remote`. That proves the client ↔ worker path works.

## Step 2: What the official worker actually does

Inside `src/tools/remote/src/main/java/com/google/devtools/build/remote/worker/`:

- `RemoteWorker.java` wires the gRPC services (Capabilities, CAS, ByteStream, ActionCache, Execution).
- `CasServer` / `ByteStreamServer` implement blob storage keyed by digest.
- `ExecutionServer`:
  1. Looks up Action + Command from CAS
  2. Builds a temp directory under `work_path`
  3. Materializes the input tree
  4. Spawns the process
  5. Collects outputs, uploads them, builds `ActionResult`
  6. Completes the long-running Operation

That is the irreducible loop.

## Step 3: Skeleton of a custom worker (JS outline)

The classes below show the control flow. A real implementation needs the generated protobuf stubs from `remote_execution.proto` + `bytestream.proto` + long-running Operations.

```js
class ContentAddressableStorage {
  constructor() { this.blobs = new Map(); }
  // FindMissingBlobs, BatchUpdateBlobs, BatchReadBlobs, + ByteStream Read/Write
}

class ActionCache {
  constructor() { this.results = new Map(); }
  // GetActionResult, UpdateActionResult
}

class ExecutionService {
  constructor({ cas, ac, workPath }) {
    this.cas = cas;
    this.ac = ac;
    this.workPath = workPath;
  }

  async execute(call) {
    const actionDigest = call.request.actionDigest;
    // 1. try ActionCache
    // 2. fetch Action + Command from CAS
    // 3. materialize input root into temp dir under workPath
    // 4. spawn command
    // 5. upload outputs to CAS
    // 6. UpdateActionResult
    // 7. stream done Operation with ExecuteResponse
  }
}
```

## Step 4: Make it real

To turn the skeleton into a working worker you must:

1. Generate JS stubs from the official protos:
   - https://github.com/bazelbuild/remote-apis/blob/main/build/bazel/remote/execution/v2/remote_execution.proto
   - https://github.com/googleapis/googleapis/blob/master/google/bytestream/bytestream.proto
   - google/longrunning/operations.proto

2. Implement:
   - Capabilities (return SHA-256 + exec_enabled)
   - CAS + ByteStream (digest → bytes)
   - ActionCache (digest → ActionResult)
   - Execution.Execute that:
     - walks the input Directory tree (GetTree or recursive BatchRead)
     - writes files into a temp dir
     - runs the argv
     - collects declared output_paths
     - stores them back in CAS
     - returns a completed Operation

3. Point Bazel at it:

```bash
bazel build //:hello \
  --remote_executor=grpc://localhost:8080 \
  --remote_cache=grpc://localhost:8080
```

## Working end state

The official Bazel worker already is that working worker.  
The tutorial above is the map from the irreducible protocol surface to that implementation.  
Once you have the official worker running against this repo’s `//:hello`, you have the complete end-to-end system.
