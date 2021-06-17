## Overview

`afr`: **A**lways **Fr**esh. Tiny library for [Deno](https://deno.land) that:

* Reloads pages on changes.
* Reinjects CSS without reloading.
* Optionally serves files.
  * Optionally just like GitHub Pages.

Two components:

* Server component:
  * Used inside your server, or via optional CLI.
  * Notifies clients.
  * Notification can be triggered by HTTP request from another process.
    * Allows page reload _immediately_ after server restart. See [`examples`](tree/master/examples).
* Client component:
  * Tiny [script](blob/master/client.mjs).
  * Listens for server notifications.
  * Reinjects CSS without reloading. Reloads on other changes.

Other features:

* Tiny and dependency-free. Being small is a big feature!
* Doesn't force a separate server. Runs from _within_ your Deno server, without complicating your environment. Optionally, run separately via CLI.
* Can signal page reload _after server restart_. Extremely useful when developing a server-rendered app.
  * Implemented by running in a separate process, sending notifications from your main server process.
  * Accepts signals over HTTP, which can be sent from any process, from any language.
* Serves static files with a flexible directory configuration. Allows multiple paths with filters.

Super-lightweight alternative to Browsersync, Livereload, and file-serving libraries.

**This readme is for Deno.** For Node support, see [`afr@0.3.2`](https://github.com/mitranim/afr/blob/ef96d7daa0e6d1540e54e43c5e295521e95ab020/readme.md).

## TOC

* [Why](#why)
* [Usage](#usage)
  * [As Library](#as-library)
  * [As CLI](#as-cli)
* [Examples](#examples)
* [API](#api)
  * [`class Broad`](#class-broadopts)
  * [`class Dir`](#class-dirpath-filter)
  * [`function send`](#function-sendmsg-opts)
  * [`function maybeSend`](#function-maybesendmsg-opts)
  * [`function watch`](#function-watchpath-dirs-opts)
  * [`function resFile`](#function-resfilereq-dirs-opts)
  * [`function resSite`](#function-ressitereq-dirs-opts)
  * [`function resSiteWithNotFound`](#function-ressitewithnotfoundreq-dirs-opts)
  * [`function resExactFile`](#function-resexactfilepath-opts)
  * [Undocumented](#undocumented)
* [Known Limitations](#known-limitations)
* [Misc](#misc)

## Why

This library is born from frustrations with Browsersync and other related tools. Advantages:

* Very small, simple, fast.
* No dependencies, rather than customary **tens of megabytes**.
* Doesn't require its own server; plugs into yours.
  * Doesn't infect your stack with junk.
* Silent, doesn't spam your terminal with crap.
* No forced delays.
* Compatible with plain Deno servers. **No** framework junk.
  * Uses the built-in HTTP API, which is currently considered unstable.
* Injected CSS doesn't have long-ass names.
* Failing webpage requests don't get stuck forever.
* Reliable: if the server is running, the client is connected.
* Can reload pages immediately after server restart.
* Built-in file server.
  * Optionally compatible with GitHub Pages rules.
* ... probably more that I'm forgetting to mention.

## Usage

### As Library

```js
import * as a from 'https://deno.land/x/afr@0.4.0/afr.mjs'
```

### As CLI

Put this in a makefile, and run concurrently with your server. See [examples](#examples).

```sh
deno run --allow-net --allow-read --unstable https://deno.land/x/afr@0.4.0/afr.mjs --port 23456 --verbose true
```

## Examples

Runnable example: clone this repo, `cd` to [`examples`](tree/master/examples), and run `make`.

## API

### `class Broad(opts)`

```ts
interface BroadOpts {
  // URL pathname prefix for all Afr endpoints, including the client script.
  namespace?: string = '/afr/'
}
```

Short for "broadcaster". Handles Afr clients:

  * Serves `client.mjs`.
  * Maintains persistent connections from clients waiting for notifications.
  * Broadcasts notifications to those clients.

```ts
const bro = new a.Broad()

// Broadcasts a reload signal to all clients.
await bro.send({type: 'change'})

function serveFetchEvent(event) {
  return event.respondWith(response(event.request))
}

function response(req) {
  return bro.res(req) || new Response('fallback')
}

// Broadcasts a reload signal to all clients.
async function change() {
  await bro.send({type: 'change'})
}
```

Running Afr [as a CLI](#as-cli) starts an HTTP server that handles all requests using a [`Broad`](#class-broadopts) instance and responds with 404 to everything unknown.

### `class Dir(path, filter)`

`ƒ(string|URL, RegExp|(string)=>bool)`

Fundamental tool for serving files and handling FS events. Takes an FS path and an optional filter. For example:

```js
const dir = a.dir('target', /[.]html|css|mjs$/)
```

Many Afr functions require an array of dirs:

```js
const dirs = [
  a.dir('target'),
  a.dir('.', /[.]html|css|mjs$/),
]
```

The filter may be either a regexp or a function. Afr applies it to a path that is Posix-style (`/`-separated), relative to the dir, and _not_ URL-encoded. Dirs without a filter are permissive and "allow" any sub-path when asked.

```js
const dirs = [
  a.dir('target'),
  a.dir('.', /^static|images|scripts[/]/),
]
```

### `function send(msg, opts)`

`ƒ(any, SendOpts) -> Promise`

```ts
interface SendOpts {
  url?: URL
  port?: number
  hostname?: string
  namespace?: string
}
```

Broadcasts `msg` to Afr clients. Assumes that on `opts.url` or `opts.hostname + opts.port` there is a reachable server that handles requests using [`Broad`](#class-broadopts) instance, and makes an HTTP request that causes that broadcaster to relay `msg`, as JSON, to every connected client.

This is useful when running Afr and your own server in separate processes. This allows clients to stay connected when your server restarts, and immediately reload when it's ready.

See the [`examples`](tree/master/examples) folder for a runnable example using this pattern.

```js
const afrOpts = {port: 23456}
const dirs = [a.dir('target')]

// Call this when your server starts.
async function watch() {
  // May cause connected clients to immediately reload.
  a.maybeSend(a.change, afrOpts)

  // Watch files and notify clients about changes that don't involve restarting
  // the server, for example in CSS files.
  for await (const msg of a.watch('target', dirs, {recursive: true})) {
    await a.maybeSend(msg, afrOpts)
  }
}
```

### `function maybeSend(msg, opts)`

`ƒ(any, SendOpts) -> Promise`

Same as [`send`](#function-sendmsg-opts), but ignores any connection errors.

### `function watch(path, dirs, opts)`

`ƒ(string|URL, []Dir, WatchOpts) -> AsyncIterator<FsEvent>`

```ts
interface WatchOpts {
  recursive?: bool
  signal?: AbortSignal
}

interface FsEvent {
  type: string
  path: string
}
```

Wraps `Deno.watchFs` (Deno), converting FS events into messages understood by `client.mjs`.

`path` and `opts` are passed directly to the underlying FS watch API. `dirs` must be an array of [`Dir`](#class-dirpath-filter); they're used to convert absolute FS paths to relative URL paths, and to filter events via `dir.allow`.

To ignore certain paths, use dir filters; see [`Dir`](#class-dirpath-filter).

The resulting messages can be broadcast to connected clients via `bro.send` (when using a [broadcaster](#class-broadopts) in the same process) or [`send`](#function-sendmsg-opts) (when using an external process).

For cancelation, just break out of the loop or call `.return()` on the iterator. You can also pass `opts.signal`, which must be an `AbortSignal`, and later abort it.

Example:

```js
const dirs = [a.dir('target'), a.dir('.', /[.]mjs$/)]

for await (const msg of a.watch('.', dirs, {recursive: true})) {
  await a.maybeSend(msg, afrOpts)
}
```

### `function resFile(req, dirs, opts)`

`ƒ(Request, []Dir, ResponseInit) -> Promise<Response | undefined>`

Tries to serve a file specified by `req.url` from `dirs`.

`dirs` must be an array of [`Dir`](#class-dirpath-filter). They're used as mount points _and_ filters. For each dir, `req.url` is resolved relative to that directory, and only the paths "allowed" by its filter may be served. Unlike most file-serving libraries, this allows you to easily and _safely_ serve files out of `.`. In addition, this will automatically reject paths containing `..`.

Has limited `content-type` detection; see [`resExactFile`](#function-resexactfilepath-opts) for details.

File closing should be automatic; see [`resExactFile`](#function-resexactfilepath-opts) for details.

```js
const dirs = [a.dir('target'), a.dir('.', /[.]html$/)]

async function response(req) {
  return (
    (await a.resFile(req, dirs)) ||
    new Response('not found', {status: 404})
  )
}
```

### `function resSite(req, dirs, opts)`

`ƒ(Request, []Dir, ResponseInit) -> Promise<Response | undefined>`

Same as [`resSiteWithNotFound`](#function-ressitewithnotfoundreq-dirs-opts), but without the `404.html` fallback.

### `function resSiteWithNotFound(req, dirs, opts)`

`ƒ(Request, []Dir, ResponseInit) -> Promise<Response | undefined>`

Variant of [`resFile`](#function-resfilereq-dirs-opts) that mimics GitHub Pages, Netlify, and other static-site hosting providers, by trying additional fallbacks when no exact match is found:

  * Try appending `.html`, unless the URL already looks like a file request or ends with `/`.
  * Try appending `/index.html`, unless the URL already looks like a file request.
  * Try serving `404.html` with status code 404.

Extremely handy for developing a static site to be served by providers such as GitHub. Check [`examples`](tree/master/examples) for runnable examples.

```js
const dirs = [a.dir('target'), a.dir('.', /[.]html$/)]

async function response(req) {
  return (
    (await a.resSiteWithNotFound(req, dirs)) ||
    new Response('not found', {status: 404})
  )
}
```

### `function resExactFile(path, opts)`

`ƒ(string|URL, ResponseInit) -> Promise<Response>`

Lower-level tool used by other file-serving functions. Serves a specific file, which _must_ exist in the FS. `path` is anything accepted by `Deno.open`; it may be a relative FS path, absolute FS path, or file URL.

Has limited `content-type` detection. If `opts.headers` doesn't already include `content-type`, tries to guess it by file extension. Known content types are stored in the `contentTypes` dictionary (exported but undocumented), which you can import and mutate.

**Warning**: this keeps the file open until the stream is fully read, or until `res.body.cancel()`. Both are handled automatically by Deno when serving the response, but it's _your_ responsibility to immediately start serving this response. Otherwise the file descriptor may leak.

**Warning**: this may blindly serve **any** file from the filesystem. _Never_ pass externally-provided paths such as `req.url` to this function. This must be used _only_ for paths that are safe to publicly expose. For serving arbitrary files from a folder, use [`resFile`](#function-resfilereq-dirs-opts) or [`resSite`](#function-ressitereq-dirs-opts).

```js
async function response() {
  return (
    (await a.resExactFile('index.html')) ||
    (await a.resExactFile('404.html', {status: 404})) ||
    new Response('not found', {status: 404})
  )
}
```

### Undocumented

Some APIs are exported but undocumented to avoid bloating the docs. Check the source files and look for `export`.

## Known Limitations

Supports only the built-in Deno HTTP server. For stdlib support, use [`afr@0.3.2`](https://github.com/mitranim/afr/blob/ef96d7daa0e6d1540e54e43c5e295521e95ab020/readme.md).

Afr's file-serving features are probably not production-grade. It _does_ take measures to prevent unauthorized access, and _does_ stream instead of buffering, but does _not_ support automatic cache headers and etags. However, Afr _does_ help you implement smart, fine-grained caching headers yourself. You can combine `resolveFile` (undocumented) and [`resExactFile`](#function-resexactfilepath-opts), adding caching headers based on each file's location and FS info. Different apps might have different caching strategies for different assets. A one-size-fits-all solution provided by most file-serving libraries is usually not the best strategy.

No TypeScript yet. May convert in later versions.

## Changelog

### `0.4.0`

* Support only Deno.
* Support only built-in HTTP server.
* Revamped many signatures to `ƒ(Request) -> Response`.

### `0.3.2`

Improved the timing of the first response over a new HTTP connection when running via CLI in Node on Windows.

### `0.3.1`

In Deno, when loading/running Afr by URL, `Broad` should now be able to serve the client script.

### `0.3.0`

* Support both Node and Deno.
* Removed daemon features. Run Afr in foreground, in parallel with your server. Use Make to orchestrate build tasks and sub-processes.
* Removed `Watcher` class; use `watch` to iterate over FS messages.
* Removed `Aio`.
* Removed `Dirs`.
* Moved IO methods from `Dirs` and `Dir` into plain functions, with some minor renaming.

### `0.2.3`

File server corrections for Windows compatibility (for real this time).

### `0.2.2`

File server corrections for Windows compatibility.

### `0.2.1`

Corrected minor race condition in CSS replacement.

### `0.2.0`

Now an extra-powerful all-in-one.

## License

https://unlicense.org

## Misc

I'm receptive to suggestions. If this library _almost_ satisfies you but needs changes, open an issue or chat me up. Contacts: https://mitranim.com/#contacts
