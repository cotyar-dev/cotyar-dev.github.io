+++
date = '2025-01-01T19:34:49-08:00'
title = 'Building on top of Cloudflare Workers'
featured_image = '/images/gohugo-default-sample-hero-image.jpg'
omit_header_text = true
+++

## Introduction

I have recently been building a project using Cloudflare
[Workers](https://developers.cloudflare.com/workers/) as the underlying
infrastructure and platform. I am definitely impressed by a lot
that is offered by Cloudflare Workers in this regard. Here are their offerings
I am utilizing as I build this current project.

1. [Queues](https://developers.cloudflare.com/queues/)
2. [Durable Objects](https://developers.cloudflare.com/durable-objects/)
3. [Wrangler](https://developers.cloudflare.com/workers/wrangler/)

Cloudflare's documentation is extensive but I have noticed that some obvious
thing such as TOML configuration syntax of
[wrangler.toml](https://developers.cloudflare.com/workers/wrangler/configuration/)
is easy to trip on. At other times, some thing that is only discoverable through experience
and not laid out in the docs shows up.

In this blog post, and hopefully some in the future as well, I would like to cover these fine points
so that another passer-by can get a bit of help where the various LLMs may not exhaustively
answer one of those basic question. So without further adieu, let's jump into the first gotcha
that I spent time with today.

### wrangler.toml

Wrangler is the standard CLI provided by Cloudflare for streamlining development and deployment of
your workers. It runs `esbuild` under the hood to bundle your JavaScript/Typescript for
their [runtime](https://github.com/cloudflare/workerd).

Wrangler takes both JSON and TOML formats. I have been using the TOML format and so I was expecting that standard TOML will work under all circumstances, well, kind of. Let's see what I ran into as
I tried to introduce [environments](https://developers.cloudflare.com/workers/wrangler/environments/).

First, let's take a worker, for example, so we can apply the concept of environments to its configuration. Let's name it sample-worker.

#### sample-worker

```shell
/Users/cotyar-dev/cloudflare-post/sample-worker
├── README.md
├── src
|  └── index.ts
├── worker-configuration.d.ts
└── wrangler.toml
```

And if we look at the `wrangler.yaml` for this worker, it is a producer for a queue called
`messages` with a binding name of `MESSAGES`.

```toml
name = "sample-worker"
main = "src/index.ts"
compatibility_date = "2024-11-06"

# the following is the name of queue in production, we would like one for staging
[[queues.producers]]
queue = "messages"
binding = "MESSAGES"
```

#### Attempt #1: at adding a queue for staging env

```toml
... rest of the TOML as shown above ...

[env.staging]

vars = { ENVIRONMENT = "development" }
workers_dev = true

[[queues.producers]]
queue = "messages-dev"
binding = "MESSAGES"
```

This goes on to show that keys that are immediately at the root of `[env.staging]` work fine and our
issue has to do with nested nature of the `producers` key.

```shell
npx wrangler deploy

 ⛅️ wrangler 3.99.0
-------------------

▲ [WARNING] Processing producer/wrangler.toml configuration:
    - "env.staging" environment configuration
      - "queues" exists at the top level, but not on "env.staging".
        This is not what you probably want, since "queues" is not inherited by environments.
        Please add "queues" to "env.staging".
```

Interesting. :thinking:

#### Attempt #2: after learnig some more TOML and running it through the validator

Apparently, TOML has the concept of tables and array of tables. I consulted this [primer](https://learnxinyminutes.com/toml/). This attempt was inspired by seeing the `durable_objects` key
in this [PR](https://github.com/cloudflare/cloudflare-docs/pull/4720).

```toml
... rest of the TOML as shown above ...

[env.staging]
vars = { ENVIRONMENT = "development" }
workers_dev = true

[queues]
producers = [
	{queue = "messages-dev", binding = "MESSAGES"}
]
```

```shell
npx wrangler deploy

⛅️ wrangler 3.99.0
-------------------

✘ [ERROR] Can't redefine existing key

    producer/wrangler.toml:36:0:
      14 │ ]
         ╵
```

Uh oh, :raised_eyebrow: - so Attempt #2 shows us, that Wrangler is making a `queues` key under the hood, which makes sense for many reasons. Under both `queues.producers` and `queues.consumers`, there are various default
settings to be had. Perhaps Wrangler sets those up first and you can't be going on redefining the key.

#### Attempt 3: fully qualify any entry under the environment (always works!)

This was inspired by running across this [post](https://justin.poehnelt.com/posts/cloudflare-workers-wrangler-dev-staging-prod/) by Justin Poehnelt. By targeting the specific
nested property and fully qualifying it, it turns out Wrangler is able to both
set up the parent key and absort what we are trying to put in for the specific `env.staging`
section.

```toml
name = "sample-worker"
main = "src/index.ts"
compatibility_date = "2024-11-06"

# the following is the name of queue in production, we would like one for staging
[[queues.producers]]
queue = "messages"
binding = "MESSAGES"

[env.staging]
vars = { ENVIRONMENT = "development" }
workers_dev = true

[[env.staging.queues.producers]]
queue = "messages-dev"
binding = "MESSAGES"
```

And voila! :heart_eyes:

```shell
npx wrangler deploy

 ⛅️ wrangler 3.99.0
-------------------

Total Upload: 275.51 KiB / gzip: 57.73 KiB
Worker Startup Time: 20 ms
Your worker has access to the following bindings:
- Queues:
  - MESSAGES: messages-dev
- Vars:
  - ENVIRONMENT: "development"
```

Keep a look out for my other learning posts about Cloudflare Workers. Cheers!