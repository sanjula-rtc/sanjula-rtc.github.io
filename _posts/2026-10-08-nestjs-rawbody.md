---
layout: post
title: "NestJS rawBody: What It Is, When You Need It, and Why"
date: 2026-10-08
categories: nestjs
tags: [nestjs, rawbody, webhooks, hmac, security]
---

Most of the time you never think about the request body. NestJS reads the JSON for you and hands you a nice object. But there is one situation where that helpful parsing **breaks things**: verifying webhook signatures. That's what `rawBody` is for.

## What is rawBody?

When a request arrives, the body is just **bytes**. NestJS (through Express or Fastify) normally does this for you:

```
 bytes on the wire                   what your controller gets
 ─────────────────                   ─────────────────────────
 {"id": 1,  "paid": true}   ──▶  parse  ──▶   { id: 1, paid: true }   (@Body())
```

The parsed object is convenient, but it **throws away the original bytes**: the spaces, the key order, the exact formatting. `rawBody` keeps a copy of those original bytes as a `Buffer`:

```
 {"id": 1,  "paid": true}   ──▶  keep a copy  ──▶  <Buffer 7b 22 69 64 ...>   (req.rawBody)
```

## Why would you need the original bytes?

Because of **signatures**. Services like Stripe, GitHub and Shopify prove a webhook really came from them by signing the body:

```
 Sender                                   Your server

 body = '{"id": 1,  "paid": true}'
 sig  = HMAC(secret, body)   ──────────▶  receives body + sig
 sends body + sig in a header             computes HMAC(secret, body) too
                                          same?  ✔ real     different?  ✘ reject
```

The key point: an HMAC depends on **every single byte**. Change one space and the signature is completely different.

Now see what goes wrong if you use the parsed object:

```
 Original body:     {"id": 1,  "paid": true}      ← two spaces
 After parse + JSON.stringify:
                    {"id":1,"paid":true}          ← different bytes!

 HMAC(original) ≠ HMAC(re-stringified)   →   valid webhook rejected ❌
```

Re-stringifying gives you the *same data* but *different bytes*, so the signature check fails. You must hash **exactly what was sent**.

## How to turn it on

Add one option when creating the app:

```ts
// main.ts
const app = await NestFactory.create(AppModule, {
  rawBody: true,
});
```

Then read it in a controller. Use the `RawBodyRequest` type so TypeScript knows about `rawBody`:

```ts
import { Controller, Post, Req, Headers, BadRequestException } from '@nestjs/common';
import type { RawBodyRequest } from '@nestjs/common';
import type { Request } from 'express';

@Controller('webhooks')
export class WebhookController {
  @Post('github')
  handle(
    @Req() req: RawBodyRequest<Request>,
    @Headers('x-hub-signature-256') signature: string,
  ) {
    if (!req.rawBody) throw new BadRequestException('No raw body');
    // verify signature using req.rawBody (see below)
    return { ok: true };
  }
}
```

`req.rawBody` is a `Buffer`. The normal `@Body()` still works too, so you get both.

## Verifying a signature correctly

Here is the GitHub style (`sha256=<hex>`) as an example:

```ts
import { createHmac, timingSafeEqual } from 'node:crypto';

function isValid(rawBody: Buffer, signatureHeader: string, secret: string) {
  const expected =
    'sha256=' + createHmac('sha256', secret).update(rawBody).digest('hex');

  const a = Buffer.from(expected);
  const b = Buffer.from(signatureHeader ?? '');

  // timingSafeEqual throws if lengths differ, so check first
  return a.length === b.length && timingSafeEqual(a, b);
}
```

Two things to note:

- **Hash the `Buffer` directly.** Don't convert to a string and back, since encoding can change bytes.
- **Use `timingSafeEqual`, not `===`.** A normal string comparison stops at the first different character, and the tiny timing difference can leak information about the correct signature.

Some SDKs do this for you. For example, Stripe's `stripe.webhooks.constructEvent(rawBody, signature, secret)` takes the raw body and the signature header and throws if they don't match.

## When to use it (and when not)

| Situation | Use rawBody? |
|---|---|
| Webhooks signed with HMAC (Stripe, GitHub, Shopify, Slack…) | **Yes** |
| Any API where a signature covers the exact request body | **Yes** |
| Storing or forwarding the exact original payload (audit logs, replays) | Yes |
| Normal JSON API for your own frontend | No, use `@Body()` |
| File uploads | No, handle with a streaming approach (like multer) |

## Gotchas

1. **It costs memory.** NestJS keeps a copy of each request body next to the parsed one. For normal JSON that's nothing, but for large payloads it adds up. If only one route needs it, consider handling that route separately.
2. **Only for bodies NestJS parses.** The raw copy is captured by the built-in body parser. If you set `bodyParser: false` and wire up your own parser, you have to capture the raw bytes yourself.
3. **Don't put middleware that changes the body before it.** Anything that rewrites or re-encodes the body ahead of the parser defeats the purpose.
4. **Behind proxies, check what's forwarded.** The signature is over the bytes the **sender** produced. A proxy that re-encodes or decompresses the body in unexpected ways can break it.
5. **Fastify works too**, but you type the app as `NestFastifyApplication` and the request is the Fastify request, not Express.
6. **Check the version.** The `rawBody: true` option exists in recent NestJS versions (v10 and later, as far as I know). In older versions you had to set up a custom `verify` function on the body parser.
7. **Verify before you trust.** Check the signature first, and only then use the parsed `@Body()`. Also check the webhook's timestamp if the provider sends one, to block replay attacks.

## The rule of thumb

> **`@Body()` is for using the data. `rawBody` is for proving the data wasn't changed.**
> If a signature covers the body, hash the raw bytes. Never re-stringify the parsed object.
