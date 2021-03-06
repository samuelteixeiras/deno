## Workers

Deno supports
[`Web Worker API`](https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker).

Workers can be used to run code on multiple threads. Each instance of `Worker`
is run on a separate thread, dedicated only to that worker.

Currently Deno supports only `module` type workers; thus it's essential to pass
`type: "module"` option when creating a new worker:

```ts
// Good
new Worker("./worker.js", { type: "module" });

// Bad
new Worker("./worker.js");
new Worker("./worker.js", { type: "classic" });
```

### Permissions

Creating a new `Worker` instance is similar to a dynamic import; therefore Deno
requires appropriate permission for this action.

For workers using local modules; `--allow-read` permission is required:

**main.ts**

```ts
new Worker("./worker.ts", { type: "module" });
```

**worker.ts**

```ts
await console.log("hello world");
self.close();
```

```shell
$ deno run main.ts
error: Uncaught PermissionDenied: read access to "./worker.ts", run again with the --allow-read flag

$ deno run --allow-read main.ts
hello world
```

For workers using remote modules; `--allow-net` permission is required:

**main.ts**

```ts
new Worker("https://example.com/worker.ts", { type: "module" });
```

**worker.ts** (at https[]()://example.com/worker.ts)

```ts
console.log("hello world");
self.close();
```

```shell
$ deno run main.ts
error: Uncaught PermissionDenied: net access to "https://example.com/worker.ts", run again with the --allow-net flag

$ deno run --allow-net main.ts
hello world
```

### Using Deno in worker

> This is an unstable Deno feature. Learn more about
> [unstable features](./stability.md).

By default the `Deno` namespace is not available in worker scope.

To add the `Deno` namespace pass `deno: true` option when creating new worker:

**main.js**

```ts
const worker = new Worker("./worker.js", { type: "module", deno: true });
worker.postMessage({ filename: "./log.txt" });
```

**worker.js**

```ts
self.onmessage = async (e) => {
  const { filename } = e.data;
  const text = await Deno.readTextFile(filename);
  console.log(text);
  self.close();
};
```

**log.txt**

```
hello world
```

```shell
$ deno run --allow-read --unstable main.js
hello world
```

When the `Deno` namespace is available in worker scope, the worker inherits its
parent process' permissions (the ones specified using `--allow-*` flags).

We intend to make permissions configurable for workers.
