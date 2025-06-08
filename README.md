# edge-tts (Node.js)

[![API Documentation](https://img.shields.io/badge/API-Documentation-blue)](./API.md)
[![npm](https://img.shields.io/npm/v/@travisvn/edge-tts)](https://www.npmjs.com/package/@travisvn/edge-tts)

This is a Node.js TypeScript conversion of the Python [`edge-tts`](https://github.com/rany2/edge-tts) library. It allows you to use Microsoft Edge's online text-to-speech service from within your Node.js applications.

This package provides high fidelity to the original, replicating the specific headers and WebSocket communication necessary to interact with the service.

## Installation

```bash
npm install @travisvn/edge-tts
# or
yarn add @travisvn/edge-tts
```

## Quick Start

```typescript
import { Communicate } from '@travisvn/edge-tts';
import fs from 'fs/promises';

const communicate = new Communicate('Hello, world!', {
  voice: 'en-US-EmmaMultilingualNeural',
});

const buffers: Buffer[] = [];
for await (const chunk of communicate.stream()) {
  if (chunk.type === 'audio' && chunk.data) {
    buffers.push(chunk.data);
  }
}

await fs.writeFile('output.mp3', Buffer.concat(buffers));
```

## Usage Examples

<details>
<summary><strong>📁 Basic Usage (Saving to a file)</strong></summary>

Here is a simple example of how to save synthesized speech to an MP3 file.

```typescript
// examples/simple.ts
import { Communicate } from '@travisvn/edge-tts';
import { promises as fs } from 'fs';
import path from 'path';

const TEXT =
  'Hello, world! This is a test of the new edge-tts Node.js library.';
const VOICE = 'en-US-EmmaMultilingualNeural';
const OUTPUT_FILE = path.join(__dirname, 'test.mp3');

async function main() {
  const communicate = new Communicate(TEXT, { voice: VOICE });

  const buffers: Buffer[] = [];
  for await (const chunk of communicate.stream()) {
    if (chunk.type === 'audio' && chunk.data) {
      buffers.push(chunk.data);
    }
  }

  const finalBuffer = Buffer.concat(buffers);
  await fs.writeFile(OUTPUT_FILE, finalBuffer);

  console.log(`Audio saved to ${OUTPUT_FILE}`);
}

main().catch(console.error);
```

</details>

<details>
<summary><strong>🎤 Listing and Finding Voices</strong></summary>

You can list all available voices and filter them by criteria.

```typescript
// examples/listVoices.ts
import { VoicesManager } from '@travisvn/edge-tts';

async function main() {
  const voicesManager = await VoicesManager.create();

  // Find all English voices
  const voices = voicesManager.find({ Language: 'en' });
  console.log(
    'English voices:',
    voices.map((v) => v.ShortName)
  );

  // Find female US voices
  const femaleUsVoices = voicesManager.find({
    Gender: 'Female',
    Locale: 'en-US',
  });
  console.log(
    'Female US voices:',
    femaleUsVoices.map((v) => v.ShortName)
  );
}

main().catch(console.error);
```

</details>

<details>
<summary><strong>📺 Streaming with Subtitles (WordBoundary events)</strong></summary>

The `stream()` method provides `WordBoundary` events for generating subtitles.

```typescript
// examples/streaming.ts
import { Communicate, SubMaker } from '@travisvn/edge-tts';

const TEXT = 'This is a test of the streaming functionality, with subtitles.';
const VOICE = 'en-GB-SoniaNeural';

async function main() {
  const communicate = new Communicate(TEXT, { voice: VOICE });
  const subMaker = new SubMaker();

  for await (const chunk of communicate.stream()) {
    if (chunk.type === 'audio' && chunk.data) {
      // Do something with the audio data, e.g., stream it to a client.
      console.log(`Received audio chunk of size: ${chunk.data.length}`);
    } else if (chunk.type === 'WordBoundary') {
      subMaker.feed(chunk);
    }
  }

  // Get the subtitles in SRT format.
  const srt = subMaker.getSrt();
  console.log('\nGenerated Subtitles (SRT):\n', srt);
}

main().catch(console.error);
```

</details>

## API Reference

📖 **[Complete API Documentation →](./API.md)**

The main exports of the package are:

- **`Communicate`** - The core class for TTS synthesis
- **`VoicesManager`** - A class to find and filter voices
- **`listVoices`** - A function to get all available voices
- **`SubMaker`** - A utility to generate SRT subtitles from `WordBoundary` events
- **Exception classes** - `NoAudioReceived`, `WebSocketError`, etc.
- **TypeScript types** - Complete type definitions for voices, options, and stream chunks

For detailed documentation, examples, and advanced usage patterns, see the [comprehensive API guide](./API.md).
