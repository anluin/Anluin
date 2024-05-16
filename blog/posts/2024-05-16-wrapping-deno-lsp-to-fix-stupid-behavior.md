# Wrapping `deno lsp` to fix annoying behavior

## Introduction

Deno is a modern runtime for JavaScript and TypeScript that simplifies the process of building applications. Currently,
I’m working on a tool for building full-stack applications with Deno. Applications built with this tool have two entry
points: `src/backend/main.ts` and `src/frontend/main.ts`. The backend is a standard Deno app, while the frontend needs
to go through esbuild first to generate two `bundle.js` files – one for server-side rendering and one for client-side
rendering.

Although one bundle could suffice, the additional magic happening in the process simplifies API implementations, making
my life easier. But that’s not the focus for now!

## The Problem

In the frontend, it should be possible to import CSS files using "import," which then end up in the `bundle.css`. While
earlier versions of Deno made this challenging, importing CSS files is now relatively straightforward. **Unfortunately,
** `deno check` still gets upset if you try to import classNames from `.css` files, as Deno doesn’t know what to do with
this.

### Issue with `deno check`

Here’s why `deno check` is problematic:

1. **CSS Imports**: While importing CSS files is supported, importing specific classNames from them is not.
2. **Interference**: It interferes with the seamless development experience, showing errors that disrupt productivity.

## Solution

To tackle this, I created a tool that searches for all `.css` files in the frontend and generates `.d.ts` files for
them. Then it uses an importMap to overwrite importing the `.css` with the newly created `.d.ts` files – thus appeasing
Deno.

### How It Works

This method works great! However, `deno lsp` now suggests that all `.css` file imports should be adjusted as found in
the importMap, which is quite frustrating since it's not what I want.

As ignoring "import-map-remap" via a comment is impossible, I decided to write a wrapper for `deno lsp`.

#### Wrapper Explanation

This wrapper is quite simple:

- **Initialization**: When started, it simply starts the Deno LSP and pipes stdin, stdout, and stderr back and forth.
- **Manipulation**: It manipulates stdout by checking if a diagnostic of "import-map-remap" is encountered. If it is a
  relative import, or if the current import is shorter than the suggestion from Deno, the diagnostic is filtered out.

### Code Explanation

We'll break down the crucial parts of the code for clarity.

#### Setting Up the Process

Here's how we initialize and spawn the Deno LSP process:

```typescript
import { dirname, join } from "https://deno.land/std@0.224.0/path/mod.ts";

const handleLspCommand = async () => {
    const process = (
        new Deno.Command(join(dirname(Deno.execPath()), "./deno"), {
            env: Deno.env.toObject(),
            cwd: Deno.cwd(),
            args: Deno.args,
            stdin: "piped",
            stdout: "piped",
            stderr: "piped",
        }).spawn()
    );

    // ... Other initialization code
};
```

#### Transform Stream: Buffer Expansion and Content Length Extraction

This part of the `TransformStream` handles expanding the buffer if needed and extracting the content length from
headers:

```typescript
const textDecoder = new TextDecoder();
const textEncoder = new TextEncoder();
let contentLength;
let buffer = new Uint8Array(1024);
let bufferCursor = 0;

const handleStream = new TransformStream({
    transform(chunk, controller) {
        if (buffer.length < bufferCursor + chunk.length) {
            const newBuffer = new Uint8Array(buffer.length * 2);
            newBuffer.set(buffer, 0);
            buffer = newBuffer;
        }
        buffer.set(chunk, bufferCursor);
        bufferCursor += chunk.length;

        for (; ;) {
            // Process the buffer and parse headers to find content length
            if (contentLength === undefined) {
                const index = buffer.findIndex((_, index, buffer) => {
                    return (
                        buffer[index + 0] === 13 && // '\r'
                        buffer[index + 1] === 10 && // '\n'
                        buffer[index + 2] === 13 && // '\r'
                        buffer[index + 3] === 10    // '\n'
                    );
                });

                if (index !== -1) {
                    const headersText = textDecoder.decode(buffer.slice(0, index));
                    buffer.set(buffer.slice(index + 4), 0);
                    bufferCursor -= index + 4;
                    buffer.fill(0, bufferCursor);

                    const headers = new Headers(headersText.split("\r\n").map(header => {
                        const splitIndex = header.indexOf(":");

                        return [
                            header.slice(0, splitIndex).trim(),
                            header.slice(splitIndex + 1).trim(),
                        ];
                    }));

                    const contentLengthText = headers.get("content-length");

                    if (!contentLengthText) {
                        Deno.exit(1); // Exit if content-length header is missing
                    }

                    contentLength = +contentLengthText; // Parse content length

                    continue; // Continue to next iteration
                }
            }

            // ... Additional processing

            break;
        }
    },
    // ... Other required handlers
});
```

#### Content Processing and Diagnostic Filtering

Once the content length is known and the content is ready, we process and filter diagnostics:

```typescript
for (; ;) {
    // ... Additional processing

    if (contentLength <= bufferCursor) {
        const contentText = textDecoder.decode(buffer.slice(0, contentLength));
        buffer.set(buffer.slice(contentLength), 0); // Remove processed data from buffer
        bufferCursor -= contentLength; // Update cursor position
        buffer.fill(0, bufferCursor, buffer.length); // Clear the remaining part of buffer
        contentLength = undefined; // Reset content length

        const contentJson = JSON.parse(contentText);

        // Filter out unnecessary diagnostics if method is 'textDocument/publishDiagnostics'
        if (contentJson.method === "textDocument/publishDiagnostics") {
            contentJson.params.diagnostics = contentJson.params.diagnostics.filter(diagnostic => !(
                diagnostic.code === "import-map-remap" &&
                diagnostic.data.from.startsWith("./") &&
                diagnostic.data.from.length < diagnostic.data.to.length
            ));
        }

        const contentBytes = textEncoder.encode(JSON.stringify(contentJson));
        controller.enqueue(textEncoder.encode(`Content-Length: ${contentBytes.length}\r\n\r\n`));
        controller.enqueue(contentBytes);

        continue; // Continue to next iteration
    }

    break;
}
```

#### Handling the Types Command

We also need to handle the `types` command to ensure everything works smoothly with WebStorm:

```typescript
const handleTypesCommand = async () => Deno.exit(await (
    new Deno.Command(join(dirname(Deno.execPath()), "./deno"), {
        env: Deno.env.toObject(),
        cwd: Deno.cwd(),
        args: Deno.args,
        stdin: "inherit",
        stdout: "inherit",
        stderr: "inherit",
    }).output().then(output => output.code)
));

switch (Deno.args[0]) {
    case "lsp":
        await handleLspCommand();
        break;
    case "types":
        await handleTypesCommand();
        break;
}

Deno.exit(1);
```

You can find the complete source code on [GitHub Gist](https://gist.github.com/anluin/e78105c1ec3d955df82efdc31b1b3c71).

## Challenges and Workarounds

After modifying the path to `deno (lsp)` in my IDE, things worked with the exception that WebStorm could no longer load
type definitions. It was quite annoying to figure out why, but the solution required not just wrapping the `lsp`
sub-command of Deno but also the `types` sub-command – since WebStorm’s Deno plugin uses both commands.

Once I wrapped both sub-commands, everything worked perfectly, and I could patch type definitions and diagnostics
from `deno lsp`.

## Conclusion and Takeaways

### Benefits of the Solution

- **Seamless Development**: By filtering out unnecessary diagnostics, the development experience becomes smoother.
- **Maintainability**: Avoided the need to fork and patch Deno itself, which would have been more cumbersome.

## Personal Insights

Working on this solution provided valuable insights into the inner workings of Deno and its LSP. While not an optimal
solution, it showcases the flexibility and power of creating custom wrappers to meet specific needs. My advice to others
facing similar challenges is to explore creative solutions and always test thoroughly in different environments.

Feel free to incorporate these insights and improvements into your workflow for a more productive and less frustrating
development experience!