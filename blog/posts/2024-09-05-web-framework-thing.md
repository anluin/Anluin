# Crafting Yet Another Web Framework (2/2)

Welcome back, intrepid developer! In the first part of this series, we embarked on the adventure of crafting a web
framework destined to disrupt, well, absolutely nothing. But our dedication to reinventing the wheel knows no bounds, so
let's dive into the nuances that complete our somewhat futile project.

## The Path Forward

Previously, we got a basic "Hello, world!" being served up, thanks to the magical combination of Deno, esbuild, and
JSDOM. But we stopped short of making our creation fully functional. Today, we're going to distinguish between the
server-side and client-side contexts, ensuring they operate seamlessly.

### Contextual Differences

One of the trickiest hurdles when developing a full-stack web framework is managing context. We need to know whether
code is running on the server or the client. Here's how we handle this thin line:

In the latest commit, we introduced a `comptime` constant to distinguish between client and server execution contexts.
This ensures that our code behaves appropriately depending on where it's being executed.

### Function Separation

First, we refined our `serve` function in `cli/src/main.ts` to build for both server and client contexts using shared
build options. The idea here is to have a common set of build configurations while allowing slight variations for server
and client builds.

#### Shared Build Options

We start by defining the shared build options which include configurations that are common to both server-side and
client-side builds:

```typescript
const sharedBuildOptions = {
    write: false,
    platform: "browser",
    format: "esm",
    bundle: true,
    outdir: "/",
    entryNames: "index",
    minify: true,
    absWorkingDir: projectRootDirectoryPath,
    entryPoints: [frontendEntryPointFilePath],
};
```

#### Parallel Builds

Next, we perform parallel builds for the server-side and client-side contexts. This way, the server and client can be
built simultaneously, speeding up our build process:

```typescript
const [serverSideBuildResult, clientSideBuildResult] = await Promise.all([
    esbuild.build({ ...sharedBuildOptions }),
    esbuild.build({
        ...sharedBuildOptions,
        define: { server: `undefined` },
    })
]);
```

#### Crafting the Server-side Script

For the server-side build, we construct a script to run within a VM context using the results
from `serverSideBuildResult`:

```typescript
const indexJsSource = `async (server) => {${serverSideBuildResult.outputFiles.find(outputFile => outputFile.path === "/index.js")!.text}}`;

const script = new vm.Script(indexJsSource, { filename: "/index.js" });
```

### Handling Client Requests

In our `serve` function, we distinguish between requests for static assets and requests for HTML content:

#### Serving Static Assets

We locate and serve static files like JavaScript and CSS based on the request path. This allows the client to receive
the requisite files for rendering the frontend:

```typescript
const outputFile = clientSideBuildResult.outputFiles.find(outputFile => outputFile.path === url.pathname);

if (outputFile) {
    return new Response(outputFile.contents, {
        status: 200,
        headers: { "Content-Type": "application/javascript; charset=utf-8" },
    });
}
```

#### Handling HTML Requests

For HTML requests, we need JSDOM to create a document and inject the scripts and styles required by the client:

```typescript
if (request.headers.get("accept")?.includes("text/html")) {
    const dom = new jsdom.JSDOM(`<!DOCTYPE html>`, { runScripts: "outside-only", url: request.url });

    try {
        if (clientSideBuildResult.outputFiles.find(outputFile => outputFile.path === "/index.css")) {
            const { document } = dom.window;
            const link = document.createElement("link");
            link.rel = "stylesheet";
            link.href = "/index.css";
            document.head.appendChild(link);
        }

        if (clientSideBuildResult.outputFiles.find(outputFile => outputFile.path === "/index.js")) {
            const { document } = dom.window;
            const scriptElement = document.createElement("script");
            scriptElement.type = "module";
            scriptElement.src = "/index.js";
            document.head.appendChild(scriptElement);
        }

        const response = new Response(undefined, {
            status: 200,
            headers: { "Content-Type": "text/html; charset=utf-8" },
        });

        const pendingPromises: Promise<unknown>[] = [];

        await script.runInContext(dom.getInternalVMContext())({
            response,
            notifyPendingPromise(promise: Promise<unknown>) {
                pendingPromises.push(promise);
            },
        });

        while (pendingPromises.length > 0) {
            await Promise.all(pendingPromises.splice(0));
        }

        return new Response(dom.serialize(), response);
    } finally {
        dom.window.close();
    }
}
```

### Understanding `declare const server`

The `declare const server` declaration in our `main.ts` file plays a pivotal role in discerning whether our code is
running in a server-side rendering context. Let's break it down:

1. **Declaration**: In TypeScript, `declare` is used to tell the compiler about an entity's existence without defining
   it. Here, we're informing TypeScript that a `const` named `server` might exist globally.

    ```typescript
    // main.ts
    declare const server: {
        readonly response: Response;

        // Before the DOM is serialized and sent to the client,
        // all notified promises are waited for.
        notifyPendingPromise(promise: Promise<unknown>): void;
    } | undefined;
    ```

2. **Context Identification**: The `server` constant is only defined when the code is being executed server-side. It
   holds a Response object and a method to notify of pending promises that need resolution before sending the response.
   If `server` is `undefined`, we know the code is running client-side:

    ```typescript
    if (server) {
        document.title = "Hello, world!";
        document.body.appendChild(document.createTextNode("Hello, world!"));
    }
    ```

3. **Server-side Responsibilities**: When running server-side, `server` allows us to:
    - Manipulate the `Response` object.
    - Register promises that need to resolve before the final HTML is serialized and sent to the client.

This setup ensures that server-specific logic runs only when it should, providing a clear separation between server-side
rendering and client-side execution. It avoids any unpredictable behavior of running server-specific code in the client
context or vice versa.

## The Full Circle

We've successfully differentiated between the client and server! This means we can now build advanced full-stack
applications without worrying about where our code is executing. Not bad for a slightly useless reinvention, right?

Stay tuned for more incremental improvements, where we might even add features useful enough to make this framework more
than just a personal indulgence.

Until next time, happy coding!