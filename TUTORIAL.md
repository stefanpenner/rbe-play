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

---

## Next concrete pieces

### Exact Capabilities response Bazel accepts

```js
async getCapabilities(call) {
  return {
    cacheCapabilities: {
      digestFunctions: ['SHA256'],
      actionCacheUpdateCapabilities: { updateEnabled: true },
      maxBatchTotalSizeBytes: 4 * 1024 * 1024,
      symlinkAbsolutePathStrategy: 'ALLOWED'
    },
    executionCapabilities: {
      digestFunction: 'SHA256',
      execEnabled: true
    },
    lowApiVersion: { major: 2, minor: 0 },
    highApiVersion: { major: 2, minor: 3 }
  };
}
```

### Minimal ByteStream resource-name parsing

```js
function parseBlobResource(name) {
  // download: [instance/]blobs/{hash}/{size}
  // upload:   [instance/]uploads/{uuid}/blobs/{hash}/{size}
  const parts = name.split('/');
  const blobsIdx = parts.lastIndexOf('blobs');
  if (blobsIdx < 0 || blobsIdx + 2 >= parts.length) return null;
  return {
    hash: parts[blobsIdx + 1],
    size: Number(parts[blobsIdx + 2])
  };
}

// Read
async read(call) {
  const { hash } = parseBlobResource(call.request.resourceName) || {};
  const data = cas.get(hash);
  if (!data) { call.emit('error', { code: 5 }); return; }
  call.write({ data });
  call.end();
}

// Write (client streaming)
async write(call) {
  let hash, chunks = [];
  for await (const req of call) {
    if (!hash) ({ hash } = parseBlobResource(req.resourceName) || {});
    chunks.push(req.data);
  }
  if (hash) cas.set(hash, Buffer.concat(chunks));
  call.end();
}
```

### Skeleton Execute that materializes one flat file and runs it

```js
async execute(call) {
  const actionHash = call.request.actionDigest.hash;
  const action = decode(cas.get(actionHash));          // Action proto
  const command = decode(cas.get(action.commandDigest.hash));

  const workDir = fs.mkdtempSync(path.join(this.workPath, 'a-'));

  // simplest case: single file at input root
  const root = decode(cas.get(action.inputRootDigest.hash)); // Directory
  for (const f of root.files || []) {
    const bytes = cas.get(f.digest.hash);
    fs.writeFileSync(path.join(workDir, f.name), bytes);
    if (f.isExecutable) fs.chmodSync(path.join(workDir, f.name), 0o755);
  }

  const { code, stdout, stderr } = await this.run(command, workDir);

  // upload stdout/stderr + declared outputs (omitted)
  const result = { exitCode: code, stdoutRaw: Buffer.from(stdout), stderrRaw: Buffer.from(stderr) };

  call.write({
    name: crypto.randomUUID(),
    done: true,
    response: { typeUrl: 'type.googleapis.com/build.bazel.remote.execution.v2.ExecuteResponse', value: encode({ result }) }
  });
  call.end();
}
```

These three pieces are the next irreducible layer after the service list. The official worker implements exactly this flow (plus tree walking and proper proto encoding).
