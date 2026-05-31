---
title: Run AI Locally in Chrome for Free — A Deep Dive into the Prompt API (with Playground)
published: false
description: Chrome ships Gemini Nano right inside the browser. No API keys, no cloud calls, zero cost. Here's everything you can do with it and a playground to try it all.
tags: javascript, chrome, ai, webdev
cover_image: https://hackmd.io/_uploads/SyFkroulzl.png
---

You might not have noticed, but Chrome quietly started shipping a local AI model called **Gemini Nano** — bundled right into the browser. No API keys. No cloud round-trips. No per-token cost. It just runs on your machine.

The interface to talk to it is called the **Prompt API**, and it landed in Chrome 138. I spent some time going through the full API surface and built a [playground](https://github.com/lovestaco/gemini-brow) that lets you experiment with every feature — session management, streaming, structured output, multimodal input, response prefixing, and more — in one page.

This post walks you through all of it.

---

## Why does this matter?

On-device AI flips the usual tradeoffs:

- **Free at runtime** — the model runs on the user's hardware, not your servers
- **Private by default** — no data leaves the device once the model is downloaded
- **Works offline** — after the initial download, no network required
- **Low latency** — no round-trip to a data centre

The catch is that Gemini Nano is a small model. It's great for classification, summarization, Q&A on focused content, and structured extraction. It won't replace GPT-4 for complex reasoning. Think of it as a smart, free, always-available layer you can add on top of your existing product.

---

## Hardware requirements

Before you get excited, check if your machine qualifies:

- **OS**: Windows 10/11, macOS 13+, Linux, or ChromeOS (Chromebook Plus only)
- **Storage**: 22 GB free on the volume holding your Chrome profile
- **GPU**: >4 GB VRAM (required for image/audio input)
- **CPU fallback**: 16 GB RAM + 4 CPU cores (text-only)

No mobile support yet. Chrome for Android and iOS are out for now.

---

## Enabling the API

The Prompt API isn't on by default in all Chrome builds. Enable two flags:

**Step 1** — Go to `chrome://flags/#optimization-guide-on-device-model` and set it to **Enabled BypassPerfRequirement**.

![enable optimization guide on device](https://hackmd.io/_uploads/r1f2xougMg.png)

**Step 2** — Go to `chrome://flags/#prompt-api-for-gemini-nano` and enable both the base API and the multimodal option.

![prompt api for gemini nano, prompt api for gemini nano with multimodel input](https://hackmd.io/_uploads/BySqgougMx.png)

Relaunch Chrome. Then visit `chrome://on-device-internals` to check the model download status. First use will trigger a download — Gemini Nano is a few gigabytes.

---

## The Playground

I put together a single-file HTML playground that covers the entire API surface. Clone it and open `playground.html` directly in Chrome — no build step, no server.

```
git clone https://github.com/lovestaco/gemini-brow
```

Then open `playground.html` in Chrome 138+.

---

## Session Setup and Context Window

Everything starts with a **session**. You create one with `LanguageModel.create()`, optionally passing a system prompt and expected input/output modalities.

```js
const session = await LanguageModel.create({
  initialPrompts: [
    { role: 'system', content: 'You are a helpful and friendly assistant.' }
  ],
  expectedInputs:  [{ type: 'text', languages: ['en'] }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});
```

Always call `LanguageModel.availability()` with the **same options** you'll pass to `create()` before creating a session — the model may not support certain modalities on every device.

```js
const avail = await LanguageModel.availability({
  expectedInputs:  [{ type: 'text', languages: ['en'] }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});
// 'available' | 'downloadable' | 'downloading' | 'unavailable'
```

Each session has a **context window** — a token budget that tracks everything in the conversation. When it fills up, the oldest prompt/response pairs are dropped (but the system prompt is never dropped). You can monitor it:

```js
console.log(`${session.contextUsage} / ${session.contextWindow} tokens used`);

session.addEventListener('contextoverflow', () => {
  console.log('Oldest turns are being dropped to make room');
});
```

The playground shows a live progress bar for context usage, and a warning badge on overflow.

![Chrome Prompt API Playground: Session Setup, Context Window](https://hackmd.io/_uploads/SyFkroulzl.png)

You can also **clone** a session to fork the conversation at a point in time — the clone is fully independent and won't see future messages sent to the original.

```js
const forkedSession = await session.clone();
```

And **destroy** it when you're done to free resources:

```js
session.destroy();
```

---

## Prompting the Model

### Request-based (wait for the full response)

Use `prompt()` when you want the complete output before rendering:

```js
const result = await session.prompt('Write me a short haiku about coffee.');
console.log(result);
```

Pass an `AbortController` signal to add a stop button:

```js
const controller = new AbortController();
stopBtn.onclick = () => controller.abort();

const result = await session.prompt('Write me a poem!', {
  signal: controller.signal,
});
```

![Prompt — request-based](https://hackmd.io/_uploads/SkBa4j_xfg.png)

### Streaming (show output as it generates)

Use `promptStreaming()` for longer responses. It returns a `ReadableStream` where each chunk is a **delta** — the new tokens only. Accumulate them yourself:

```js
const stream = session.promptStreaming('Explain how a browser renders a web page.');

let fullText = '';
for await (const chunk of stream) {
  fullText += chunk;
  outputEl.textContent = fullText;
}
```

This is the right pattern — don't replace the display with each raw chunk or you'll get flickering (each chunk is only a word or two).

---

## Append Messages

`session.append()` lets you pre-load context into the session without triggering a response. This is useful when you want the model to process heavy inputs (like images) while the user is still typing their question.

```js
// Pre-load context
await session.append([{
  role: 'user',
  content: 'Here is the document you will answer questions about: ...'
}]);

// Later, ask the question
const answer = await session.prompt('What are the key takeaways?');
```

The promise from `append()` resolves once the input has been processed and is ready in the session's context.

![Append Messages](https://hackmd.io/_uploads/B1574i_gzl.png)

---

## Image Input (Multimodal)

If your device has a GPU with more than 4 GB VRAM, the model can process images. **Image input needs its own session** — you must declare `{ type: 'image' }` in `expectedInputs` at creation time, and check availability separately since not all devices support it.

```js
const imageAvail = await LanguageModel.availability({
  expectedInputs:  [{ type: 'text', languages: ['en'] }, { type: 'image' }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});

if (imageAvail === 'unavailable') {
  // GPU requirement not met
  return;
}

const imageSession = await LanguageModel.create({
  expectedInputs:  [{ type: 'text', languages: ['en'] }, { type: 'image' }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});

const imageBlob = await fetch('photo.jpg').then(r => r.blob());

const result = await imageSession.prompt([{
  role: 'user',
  content: [
    { type: 'text',  value: 'What is in this image?' },
    { type: 'image', value: imageBlob },
  ],
}]);
```

The API accepts `Blob`, `HTMLImageElement`, `HTMLCanvasElement`, `ImageBitmap`, `ImageData`, and more. In the playground, you can upload any image file and ask the model about it.

![Image Input](https://hackmd.io/_uploads/HyDeNjOxfg.png)

---

## Response Prefix — Force an Output Format

One of my favourite features. You can prefill the **start** of the assistant's response by passing an assistant-role message with `prefix: true`. The model is forced to continue from that prefix.

This is a clean way to lock in an output format without relying on instruction-following:

```js
// Force the model to start its response with ```toml
const result = await session.prompt([
  {
    role: 'user',
    content: 'Create a character sheet for a gnome barbarian.',
  },
  {
    role: 'assistant',
    content: '```toml\n',
    prefix: true,
  },
]);
// result continues from the prefix: ```toml\n[character]\nname = "...
```

In the playground, you can set any prefix string and watch the model continue from exactly that point.

![Response Prefix](https://hackmd.io/_uploads/HJuIVjulGl.png)

---

## Boolean Classification

Need a fast yes/no answer? Pass `{ type: 'boolean' }` as the `responseConstraint` and you'll always get back a raw `true` or `false` — no parsing, no prompt engineering around output format:

```js
const raw = await session.prompt(
  `Is this post about pottery?\n\n"${text}"`,
  { responseConstraint: { type: 'boolean' } }
);

const result = JSON.parse(raw); // true or false
```

This is great for content moderation, topic detection, or gating features based on page content.

![Boolean Classification](https://hackmd.io/_uploads/S1YvEj_lzg.png)

---

## Structured Output with JSON Schema

The full power of `responseConstraint` is a complete **JSON Schema**. The model is constrained to produce valid JSON that matches your schema — no hallucinated keys, no wrong types.

```js
const schema = {
  type: 'object',
  properties: {
    sentiment: { type: 'string', enum: ['positive', 'negative', 'neutral'] },
    score:     { type: 'number', minimum: 0, maximum: 10 },
    summary:   { type: 'string' },
  },
  required: ['sentiment', 'score', 'summary'],
};

const raw = await session.prompt(
  `Analyze the sentiment of this review:\n\n"${reviewText}"`,
  { responseConstraint: schema }
);

const data = JSON.parse(raw);
// { sentiment: 'positive', score: 8.5, summary: '...' }
```

Note: the schema itself uses some tokens from your context window. You can measure how many with `session.measureContextUsage({ responseConstraint: schema })`.

![Structured Output](https://hackmd.io/_uploads/H1rq4j_gze.png)

---

## Putting it all together

Here's how these features combine in a real use case. Say you're building a Chrome Extension that summarises product reviews on any e-commerce page:

```js
// 1. Check availability
const avail = await LanguageModel.availability({
  expectedInputs:  [{ type: 'text', languages: ['en'] }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});
if (avail === 'unavailable') return;

// 2. Create a session with context
const session = await LanguageModel.create({
  initialPrompts: [{
    role: 'system',
    content: 'You analyse product reviews and extract structured insights.',
  }],
  expectedInputs:  [{ type: 'text', languages: ['en'] }],
  expectedOutputs: [{ type: 'text', languages: ['en'] }],
});

// 3. Pre-load the reviews while user looks at the page
await session.append([{
  role: 'user',
  content: `Here are the reviews:\n\n${scrapedReviews}`,
}]);

// 4. Get structured output
const schema = {
  type: 'object',
  properties: {
    verdict:  { type: 'string', enum: ['buy', 'skip', 'depends'] },
    pros:     { type: 'array', items: { type: 'string' } },
    cons:     { type: 'array', items: { type: 'string' } },
    summary:  { type: 'string' },
  },
  required: ['verdict', 'pros', 'cons', 'summary'],
};

const result = JSON.parse(
  await session.prompt('Summarise these reviews.', { responseConstraint: schema })
);
```

Zero API cost. Runs entirely on the user's machine. Works offline after first load.

---

## Try the playground

The full playground is one HTML file — no dependencies, no build step:

**[github.com/lovestaco/gemini-brow](https://github.com/lovestaco/gemini-brow)**

Clone it, open `playground.html` in Chrome 138+, enable the flags above, and every feature in this post is wired up and ready to experiment with.

The Prompt API is still evolving — language support is limited (`en`, `ja`, `es` for now), mobile isn't supported yet, and the model is small. But the fundamentals are solid and the use cases where it shines — classification, summarization, extraction, Q&A on focused content — are genuinely useful without touching your server budget.

---

*Have questions or ideas for what to build with it? Drop them in the comments.*
