# Framework / Plugin README

Frameworks and plugins are designed to be extended by others. The README needs to show how to
install, how to use the basic API, and how to build on top of it. The key difference from a
library is that frameworks need extension documentation and an examples showcase.

## Sections (in order)

### Features
More important for frameworks than libraries -- users are choosing an opinionated tool. List
3-5 defining features that differentiate this framework. Focus on what it enables, not
implementation details.

**Pattern:**
```markdown
## Features

- **File-based routing** -- directory structure defines your routes automatically
- **Hot reload** -- changes reflect instantly without restarting the server
- **Built-in auth** -- session management and OAuth out of the box
```

### Installation
Same as a library -- show the canonical install command. If the framework has a scaffolding
CLI, show that instead.

**Pattern:**
```markdown
## Installation

```bash
npx create-my-framework my-app
cd my-app
npm run dev
```
```

If there's no scaffolding CLI, show the manual install:
```markdown
## Installation

```bash
npm install my-framework
```
```

### Quick start
Show the minimal path from install to something running. For frameworks, this is usually
creating one file and starting a dev server, not just a function call.

**Pattern:**
```markdown
## Quick start

1. Create a route:
   ```js
   // src/routes/index.js
   export function GET() {
     return new Response("Hello world");
   }
   ```

2. Start the dev server:
   ```bash
   npm run dev
   ```

   Open http://localhost:3000.
```

### Core concepts
Frameworks are opinionated -- explain 2-3 concepts a newcomer must understand before they can
be productive. Each gets a `###` subsection with a short paragraph and a code example.

**Pattern:**
```markdown
## Core concepts

### Routing

Routes map to files in `src/routes/`. A file named `src/routes/users/[id].js` matches
`/users/:id`:

```js
export function GET({ params }) {
  return Response.json({ userId: params.id });
}
```

### Middleware

Export a default function from `src/middleware.js` to intercept every request:

```js
export function onRequest({ request, next }) {
  console.log(request.url);
  return next();
}
```
```

Keep each concept to one paragraph + one short code example. If you need more, link to docs.

### Plugin API (if applicable)
If the framework supports plugins or extensions, show how to create a minimal one. This is
the section that distinguishes frameworks from libraries.

**Pattern:**
```markdown
## Creating a plugin

```js
export default function myPlugin(options = {}) {
  return {
    name: "my-plugin",
    setup(framework) {
      framework.onBuild(async () => {
        // runs during each build
      });
    },
  };
}
```

Register it in your config:

```js
// my-framework.config.js
import myPlugin from "./plugins/my-plugin";

export default {
  plugins: [myPlugin({ verbose: true })],
};
```
```

Skip this section if the framework has no extension mechanism.

### Examples
Link to the `examples/` directory if one exists, with a brief table of what's there.

**Pattern:**
```markdown
## Examples

See the [`examples/`](examples/) directory:

| Example | Description |
|---------|-------------|
| [`basic`](examples/basic) | Minimal setup |
| [`with-auth`](examples/with-auth) | Authentication integration |
| [`custom-plugin`](examples/custom-plugin) | Writing a plugin |
```

If no examples directory exists, omit this section.

## Things to skip for frameworks
- Full configuration reference (link to docs)
- Migration guides from other frameworks (separate doc)
- Comparison tables with competitors (opinionated and quickly outdated)
- Internal architecture details (contributor doc, not README)
