# edge-tts (Node.js)

This is a Node.js TypeScript conversion of the Python [`edge-tts`](https://github.com/rany2/edge-tts) library. It allows you to use Microsoft Edge's online text-to-speech service from within your Node.js applications.

This package provides high fidelity to the original, replicating the specific headers and WebSocket communication necessary to interact with the service.

## Installation

To install the package, use npm or yarn:

```bash
npm install @travisvn/edge-tts
# or
yarn add @travisvn/edge-tts
```

## Usage

### Basic Usage (Saving to a file)

Here is a simple example of how to save synthesized speech to an MP3 file.

```typescript
// examples/simple.ts
import { Communicate, listVoices } from '@travisvn/edge-tts';
import { promises as fs } from 'fs';
import path from 'path';

const TEXT =
  'Hello, world! This is a test of the new edge-tts Node.js library.';
const VOICE = 'en-US-EmmaMultilingualNeural';
const OUTPUT_FILE = path.join(__dirname, 'test.mp3');

async function main() {
  const communicate = new Communicate(TEXT, { voice: VOICE });

  // The stream() method returns an async generator that yields audio chunks.
  const audioStream = communicate.stream();

  // Pipe the audio data to a file.
  const fileStream = fs.open(OUTPUT_FILE, 'w');

  try {
    for await (const chunk of audioStream) {
      if (chunk.type === 'audio') {
        await (await fileStream).write(chunk.data);
      }
    }
  } finally {
    (await fileStream).close();
  }

  console.log(`Audio saved to ${OUTPUT_FILE}`);
}

main().catch(console.error);
```

### Listing Voices

You can list all available voices and their attributes.

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

  // Find a specific voice
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

### Streaming with Subtitles (WordBoundary events)

The `stream()` method also provides `WordBoundary` events, which can be used to generate subtitles.

```typescript
// examples/streaming.ts
import { Communicate, SubMaker } from '@travisvn/edge-tts';

const TEXT = 'This is a test of the streaming functionality, with subtitles.';
const VOICE = 'en-GB-SoniaNeural';

async function main() {
  const communicate = new Communicate(TEXT, { voice: VOICE });
  const subMaker = new SubMaker();

  for await (const chunk of communicate.stream()) {
    if (chunk.type === 'audio') {
      // Do something with the audio data, e.g., stream it to a client.
      // For this example, we'll just log its size.
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

## API

The main exports of the package are:

- `Communicate`: The core class for TTS synthesis.
- `VoicesManager`: A class to find and filter voices.
- `listVoices`: A function to get all available voices.
- `SubMaker`: A utility to generate SRT subtitles from `WordBoundary` events.
- Various exception classes (e.g., `NoAudioReceived`, `WebSocketError`).
- TypeScript types for voices, options, and stream chunks.
