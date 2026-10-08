---
layout: post
title: "Node fetch Buffers Large Uploads: Use http.request Instead"
date: 2026-10-08
categories: nodejs
tags: [nodejs, fetch, http, streams, memory, uploads]
---

Uploading a big file from Node with `fetch` looks like it should use almost no memory, because you pass it a stream. In a test, it didn't.

## The measurement

Uploading a **500 MB** file:

| Method | Memory held during the upload |
|---|---|
| `fetch` with a streamed body | **~500 MB** |
| `http.request` / `https.request` | **~18 MB** |

The whole file ended up in memory with `fetch`. With `http.request`, memory stayed flat no matter how big the file was.

> **Rule:** for large uploads, use Node's `http` / `https` and pipe a stream into the request.

## Picture: a pipe vs a bucket

```
 http.request  (a pipe)

   file ──▶ [64 KB chunk] ──▶ socket ──▶ server
            only a small piece is in memory at any moment
            memory:  ▂▂▂▂▂▂▂▂▂▂▂▂▂▂▂   (flat, ~18 MB)


 fetch  (a bucket, in this test)

   file ──▶ [ ████████████████████ ] ──▶ socket ──▶ server
            the whole body is collected first, then sent
            memory:  ▂▃▄▅▆▇██████████   (grows with file size, ~500 MB)
```

A 5 GB file would need 5 GB of RAM in the second case, and a container with a memory limit would be killed (OOM) long before the upload finished.

## The code

The memory-friendly version, using streams with `pipeline` so backpressure is handled for you:

```js
import { request } from 'node:https';
import { createReadStream } from 'node:fs';
import { stat } from 'node:fs/promises';
import { pipeline } from 'node:stream/promises';

async function upload(path, url) {
  const { size } = await stat(path);

  const req = request(url, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/octet-stream',
      'Content-Length': size,      // lets the server know the size up front
    },
  });

  // Resolve when the server answers
  const response = new Promise((resolve, reject) => {
    req.on('response', resolve);
    req.on('error', reject);
  });

  // Streams the file in small chunks, pausing when the socket is full
  await pipeline(createReadStream(path), req);

  const res = await response;
  res.resume();                     // drain the response so the socket is freed
  return res.statusCode;
}
```

What keeps memory low:

- **`createReadStream`** reads the file in small chunks (64 KB by default), not all at once.
- **`pipeline`** pauses the file when the socket can't take more (*backpressure*) and resumes when it can. This is the key idea.
- **`Content-Length`** is set, so no chunked-encoding guessing is needed.

## What it looks like with fetch

The tempting version:

```js
const res = await fetch(url, {
  method: 'PUT',
  body: Readable.toWeb(createReadStream(path)),
  duplex: 'half',                 // required by Node when the body is a stream
});
```

On paper this streams. In our test it still held roughly the whole body in memory. Other `fetch` bodies, like `Blob`, `Buffer`, and `FormData`, are fully in memory by definition, since you've already built them before `fetch` even starts.

I haven't pinned down the exact internal reason for the stream case, and it may differ between Node versions. So **don't take the numbers on trust. Measure your own setup.**

## How to measure it yourself

Log memory while the upload runs:

```js
const timer = setInterval(() => {
  const { rss } = process.memoryUsage();
  console.log(`rss: ${(rss / 1024 / 1024).toFixed(0)} MB`);
}, 500);

await upload('./big-file.bin', url);   // or the fetch version
clearInterval(timer);
```

Use a file well above your expected memory baseline, such as 500 MB, and compare the two versions. Watch **`rss`**, not just `heapUsed`. Buffers live outside the JavaScript heap, so `heapUsed` can look healthy while the process is still growing. (That is also why `--max-old-space-size` won't save you here.)

## When fetch is still fine

- Small bodies: JSON payloads, small forms, anything that comfortably fits in memory.
- Downloads: reading a response body as a stream is a different path from sending one. Test it before assuming it behaves the same.
- Code that must run in browsers too, where `fetch` is the standard.

## The rule of thumb

> **Big upload from Node? Pipe a file stream into `http.request`.**
> A stream passed to `fetch` isn't guaranteed to stay a stream. Measure `rss` with a large file before you trust it.
