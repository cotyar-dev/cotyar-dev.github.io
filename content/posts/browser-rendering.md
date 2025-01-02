+++
date = '2025-01-01T23:04:58-08:00'
title = 'Cloudflare Browser Rendering with Pub-Sub'
featured_image = '/images/gohugo-default-sample-hero-image.jpg'
omit_header_text = true
+++

Cloudflare Browser-Rendering is in beta and currently doesn't charge you much
at all as long as you can work within its limits and have standard Workers billing
set up.

## Problem Statement

Let's imagine we want to build something like a asynchronous system
that consumes messages published on a queue via an API in a fire-and-forget
manner, where consumption entails running some puppeteer scripts
against certain websites with the data present within the messages.

### The Implementation

Keeping in mind, that I didn't have any remote Observability solution integrated into my workers at the time I initially begun this project (for which I highly recommend something like [Better Stack](https://betterstack.com/)), seeing logs appear in Cloudflare Dashboard was my only way of knowing that
messages were correctly flowing between my producer and consume as shown in the diagram below.
While the setup below works great, I would like to point out a few things I learned before I
arrived at this solution.

{{< mermaid >}}
    architecture-beta
    group cf(devicon:cloudflare)[Cloudflare]

    service browser_worker(devicon:browserstack)[Browser Rendering] in cf
    service puppeteer_scripts(devicon:cloudflareworkers)[Puppeteer Scripts] in cf

    service consumer_worker(devicon:cloudflareworkers)[Consumer] in cf

    service producer_worker(devicon:cloudflareworkers)[Producer] in cf

    service complaints_queue(disk)[Queue] in cf

    complaints_queue:R --> L:consumer_worker
    producer_worker:B --> T:complaints_queue
    consumer_worker:R --> L:puppeteer_scripts
    puppeteer_scripts:T --> B:browser_worker
{{< /mermaid >}}

*NOTE: All my below learnings are based on using a push based consumer. Pull based consumer is something I haven't tried yet.*

#### Learning #1: Placing the CF Browser logic in a `consumer` worker doesn't work

I am saying this for those of you who go for the re-use sessions implementation
as mentioned [here](https://developers.cloudflare.com/browser-rendering/get-started/reuse-sessions/). That specific implementation doesn't use a durable object and hence one might be tempted to put that logic directly in a consumer-kind of worker.

If you do that, none of the consumer logs will ever appear since they require the worker
to adequetely acknowledge or error on message events. You will also see the queue
fill-up and not empty at all - I was able to verify this using Queues dashboard to
list the messages.

My theory is that owing to its single threaded nature, which is a feature (allows for being able to reason with ones code), the consumer worker busies itself managing the CF Browser Binding (calling puppeteer API's such as NewPage() etc.) and that this stalls message processing.


#### Learning #2: Corollary of #1, using another Worker or Durable Object where Browser logic live works!

1. **Worker path**: Let's call this other worker *Browser Worker*. Just introducing an RPC/fetch handler in it and situating the browser binding there addresses the issues seen in #1. Consumer call the handler in *Browser Worker* while in its message processing loop. Processing no longer stalls.
2. **Durable Object path**: Same benefit as 1 + you could use something like [blockConcurrencyWhile](https://developers.cloudflare.com/durable-objects/api/state/#blockconcurrencywhile) while hydrating the `browser` using `puppeteer.launch` in the constructor. Potential benefits of this approach are offering Page based concurrency if you want the consumer to run multiple messages on a browser at the same time. In my experience, the [DO example](https://developers.cloudflare.com/browser-rendering/get-started/browser-rendering-with-do/) doesn't offer real advantages when using CF Browser Rendering, since it already offers built in features such as `sessionIDs` that allow re-connecting and there is no need to keep the JavaScript browser object lying around. If however, you are planning on using a different solution as Zenrows or Browserless for a remote browser, the DO example offers clear advantages.

Durable Objects are themselves quite interesting and intricate and I would like to cover them in more detail in a separate dedicated post.

So in closing, I hope you find these tidbits useful as you experiment with these new CF
features and I also hope that more people use them so we know more about how they
work as a community.