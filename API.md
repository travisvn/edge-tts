# edge-tts API Reference

A comprehensive guide to using the `@travisvn/edge-tts` package for text-to-speech synthesis using Microsoft Edge's online TTS service.

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Core Classes](#core-classes)
  - [Communicate](#communicate)
  - [VoicesManager](#voicesmanager)
  - [SubMaker](#submaker)
- [Functions](#functions)
  - [listVoices](#listvoices)
- [Types](#types)
- [Exceptions](#exceptions)
- [Advanced Usage](#advanced-usage)
- [Examples](#examples)

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

// Basic text-to-speech
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

## Core Classes

### Communicate

The main class for text-to-speech synthesis.

#### Constructor

```typescript
new Communicate(text: string, options?: CommunicateOptions)
```

**Parameters:**

- `text` (string): The text to synthesize
- `options` (CommunicateOptions, optional): Configuration options

#### CommunicateOptions

```typescript
interface CommunicateOptions {
  voice?: string; // Voice to use (default: "en-US-EmmaMultilingualNeural")
  rate?: string; // Speech rate (e.g., "+20%", "-10%")
  volume?: string; // Volume level (e.g., "+50%", "-25%")
  pitch?: string; // Pitch adjustment (e.g., "+5Hz", "-10Hz")
  proxy?: string; // Proxy URL for requests
  connectionTimeout?: number; // WebSocket connection timeout in ms
}
```

**Voice Format:**

- Short name: `"en-US-EmmaMultilingualNeural"`
- Full name: `"Microsoft Server Speech Text to Speech Voice (en-US, EmmaMultilingualNeural)"`

**Rate, Volume, Pitch Format:**

- Must include sign: `"+0%"`, `"-10%"`, `"+5Hz"`
- Rate/Volume: percentage with % suffix
- Pitch: frequency with Hz suffix

#### Methods

##### stream()

```typescript
async *stream(): AsyncGenerator<TTSChunk, void, unknown>
```

Returns an async generator that yields audio chunks and word boundary events.

**Returns:** `AsyncGenerator<TTSChunk>`

**Important:** Can only be called once per Communicate instance.

**Example:**

```typescript
const communicate = new Communicate('Hello world');

for await (const chunk of communicate.stream()) {
  if (chunk.type === 'audio') {
    // Handle audio data
    console.log(`Audio chunk: ${chunk.data?.length} bytes`);
  } else if (chunk.type === 'WordBoundary') {
    // Handle word timing information
    console.log(`Word: ${chunk.text} at ${chunk.offset}ms`);
  }
}
```

### VoicesManager

Utility class for finding and filtering available voices.

#### Static Methods

##### create()

```typescript
static async create(customVoices?: Voice[], proxy?: string): Promise<VoicesManager>
```

Creates a new VoicesManager instance.

**Parameters:**

- `customVoices` (Voice[], optional): Custom voice list instead of fetching from API
- `proxy` (string, optional): Proxy URL for API requests

**Returns:** `Promise<VoicesManager>`

**Example:**

```typescript
const voicesManager = await VoicesManager.create();
// or with proxy
const voicesManager = await VoicesManager.create(
  undefined,
  'http://proxy:8080'
);
```

#### Instance Methods

##### find()

```typescript
find(filter: VoicesManagerFind): VoicesManagerVoice[]
```

Finds voices matching the specified criteria.

**Parameters:**

- `filter` (VoicesManagerFind): Filter criteria

**Returns:** `VoicesManagerVoice[]`

**VoicesManagerFind:**

```typescript
interface VoicesManagerFind {
  Gender?: 'Female' | 'Male';
  Locale?: string; // e.g., "en-US", "fr-FR"
  Language?: string; // e.g., "en", "fr"
}
```

**Example:**

```typescript
const voicesManager = await VoicesManager.create();

// Find all English voices
const englishVoices = voicesManager.find({ Language: 'en' });

// Find female US voices
const femaleUSVoices = voicesManager.find({
  Gender: 'Female',
  Locale: 'en-US',
});

// Find specific locale
const frenchVoices = voicesManager.find({ Locale: 'fr-FR' });
```

### SubMaker

Utility class for generating SRT subtitles from WordBoundary events.

#### Constructor

```typescript
new SubMaker();
```

#### Methods

##### feed()

```typescript
feed(msg: TTSChunk): void
```

Adds a WordBoundary chunk to the subtitle maker.

**Parameters:**

- `msg` (TTSChunk): Must be a WordBoundary type chunk

**Throws:** `ValueError` if chunk is not a WordBoundary with required fields

##### mergeCues()

```typescript
mergeCues(words: number): void
```

Merges subtitle cues to combine multiple words per subtitle.

**Parameters:**

- `words` (number): Maximum number of words per subtitle cue

**Throws:** `ValueError` if words <= 0

##### getSrt()

```typescript
getSrt(): string
```

Returns the subtitles in SRT format.

**Returns:** `string` - SRT formatted subtitles

##### toString()

```typescript
toString(): string
```

Alias for `getSrt()`.

**Example:**

```typescript
const communicate = new Communicate('Hello world, this is a test.');
const subMaker = new SubMaker();

for await (const chunk of communicate.stream()) {
  if (chunk.type === 'WordBoundary') {
    subMaker.feed(chunk);
  }
}

// Merge every 3 words into one subtitle
subMaker.mergeCues(3);

const srt = subMaker.getSrt();
console.log(srt);
// Output:
// 1
// 00:00:00,000 --> 00:00:01,500
// Hello world, this
//
// 2
// 00:00:01,500 --> 00:00:02,800
// is a test.
```

## Functions

### listVoices()

```typescript
async function listVoices(proxy?: string): Promise<Voice[]>;
```

Fetches all available voices from the Microsoft Edge TTS service.

**Parameters:**

- `proxy` (string, optional): Proxy URL for the request

**Returns:** `Promise<Voice[]>`

**Example:**

```typescript
import { listVoices } from '@travisvn/edge-tts';

const voices = await listVoices();
console.log(`Found ${voices.length} voices`);

// With proxy
const voices = await listVoices('http://proxy:8080');
```

## Types

### TTSChunk

```typescript
type TTSChunk = {
  type: 'audio' | 'WordBoundary';
  data?: Buffer; // Audio data (only for audio type)
  duration?: number; // Duration in 100-nanosecond units (only for WordBoundary)
  offset?: number; // Offset in 100-nanosecond units (only for WordBoundary)
  text?: string; // Word text (only for WordBoundary)
};
```

### Voice

```typescript
type Voice = {
  Name: string; // Full voice name
  ShortName: string; // Short voice identifier
  Gender: 'Female' | 'Male';
  Locale: string; // e.g., "en-US"
  SuggestedCodec: 'audio-24khz-48kbitrate-mono-mp3';
  FriendlyName: string; // Human-readable name
  Status: 'GA'; // General availability
  VoiceTag: VoiceTag;
};
```

### VoicesManagerVoice

```typescript
type VoicesManagerVoice = Voice & {
  Language: string; // Extracted from Locale (e.g., "en" from "en-US")
};
```

### VoiceTag

```typescript
type VoiceTag = {
  ContentCategories: (
    | 'Cartoon'
    | 'Conversation'
    | 'Copilot'
    | 'Dialect'
    | 'General'
    | 'News'
    | 'Novel'
    | 'Sports'
  )[];
  VoicePersonalities: (
    | 'Approachable'
    | 'Authentic'
    | 'Authority'
    | 'Bright'
    | 'Caring'
    | 'Casual'
    | 'Cheerful'
    | 'Clear'
    | 'Comfort'
    | 'Confident'
    | 'Considerate'
    | 'Conversational'
    | 'Cute'
    | 'Expressive'
    | 'Friendly'
    | 'Honest'
    | 'Humorous'
    | 'Lively'
    | 'Passion'
    | 'Pleasant'
    | 'Positive'
    | 'Professional'
    | 'Rational'
    | 'Reliable'
    | 'Sincere'
    | 'Sunshine'
    | 'Warm'
  )[];
};
```

## Exceptions

All exceptions extend `EdgeTTSException`:

### EdgeTTSException

Base exception class for all edge-tts errors.

### SkewAdjustmentError

Thrown when there are issues with clock skew adjustment.

### UnknownResponse

Thrown when receiving an unexpected response from the service.

### UnexpectedResponse

Thrown when response format is unexpected.

### NoAudioReceived

Thrown when no audio data is received from the service.

### WebSocketError

Thrown when WebSocket connection issues occur.

### ValueError

Thrown for invalid parameter values.

**Example Error Handling:**

```typescript
import {
  Communicate,
  NoAudioReceived,
  WebSocketError,
} from '@travisvn/edge-tts';

try {
  const communicate = new Communicate('Hello world');
  for await (const chunk of communicate.stream()) {
    // Process chunks
  }
} catch (error) {
  if (error instanceof NoAudioReceived) {
    console.error('No audio received from service');
  } else if (error instanceof WebSocketError) {
    console.error('WebSocket connection failed:', error.message);
  } else {
    console.error('Unexpected error:', error);
  }
}
```

## Advanced Usage

### Using Proxies

```typescript
const communicate = new Communicate('Hello world', {
  proxy: 'http://proxy.example.com:8080',
});

// Also works with HTTPS proxies
const communicate2 = new Communicate('Hello world', {
  proxy: 'https://user:pass@proxy.example.com:3128',
});
```

### Custom Voice Parameters

```typescript
const communicate = new Communicate('Hello world', {
  voice: 'en-US-EmmaMultilingualNeural',
  rate: '+25%', // 25% faster
  volume: '+10%', // 10% louder
  pitch: '+2Hz', // 2Hz higher
});
```

### Connection Timeout

```typescript
const communicate = new Communicate('Hello world', {
  connectionTimeout: 10000, // 10 second timeout
});
```

### Processing Large Text

The library automatically splits large text into chunks:

```typescript
const longText = 'Your very long text here...'.repeat(1000);
const communicate = new Communicate(longText);

for await (const chunk of communicate.stream()) {
  // Chunks are automatically split and processed
}
```

## Examples

### Save to File with Error Handling

```typescript
import { Communicate, NoAudioReceived } from '@travisvn/edge-tts';
import fs from 'fs/promises';

async function saveToFile(text: string, filename: string) {
  try {
    const communicate = new Communicate(text, {
      voice: 'en-US-EmmaMultilingualNeural',
    });

    const buffers: Buffer[] = [];
    for await (const chunk of communicate.stream()) {
      if (chunk.type === 'audio' && chunk.data) {
        buffers.push(chunk.data);
      }
    }

    if (buffers.length === 0) {
      throw new NoAudioReceived('No audio chunks received');
    }

    await fs.writeFile(filename, Buffer.concat(buffers));
    console.log(`Saved ${buffers.length} chunks to ${filename}`);
  } catch (error) {
    console.error('Failed to generate audio:', error);
    throw error;
  }
}
```

### Real-time Streaming with Subtitles

```typescript
import { Communicate, SubMaker } from '@travisvn/edge-tts';

async function streamWithSubtitles(text: string) {
  const communicate = new Communicate(text);
  const subMaker = new SubMaker();

  for await (const chunk of communicate.stream()) {
    if (chunk.type === 'audio' && chunk.data) {
      // Stream audio in real-time (e.g., to speakers or network)
      await streamAudioChunk(chunk.data);
    } else if (chunk.type === 'WordBoundary') {
      subMaker.feed(chunk);

      // Real-time subtitle display
      console.log(`Word: ${chunk.text} at ${chunk.offset! / 10000}ms`);
    }
  }

  // Final subtitles
  subMaker.mergeCues(5);
  return subMaker.getSrt();
}

async function streamAudioChunk(data: Buffer) {
  // Implement your audio streaming logic here
  // e.g., write to audio device, send to client, etc.
}
```

### Voice Discovery and Selection

```typescript
import { VoicesManager } from '@travisvn/edge-tts';

async function findBestVoice(language: string, gender?: 'Male' | 'Female') {
  const voicesManager = await VoicesManager.create();

  const criteria: any = { Language: language };
  if (gender) criteria.Gender = gender;

  const voices = voicesManager.find(criteria);

  if (voices.length === 0) {
    throw new Error(
      `No voices found for ${language}${gender ? ` (${gender})` : ''}`
    );
  }

  // Prefer neural voices (they typically have "Neural" in the name)
  const neuralVoices = voices.filter((v) => v.ShortName.includes('Neural'));

  return neuralVoices.length > 0 ? neuralVoices[0] : voices[0];
}

// Usage
const voice = await findBestVoice('en', 'Female');
console.log(`Selected voice: ${voice.ShortName}`);
```

### Batch Processing

```typescript
import { Communicate } from '@travisvn/edge-tts';

async function processMultipleTexts(texts: string[], voice: string) {
  const results: Buffer[] = [];

  for (const text of texts) {
    const communicate = new Communicate(text, { voice });
    const buffers: Buffer[] = [];

    for await (const chunk of communicate.stream()) {
      if (chunk.type === 'audio' && chunk.data) {
        buffers.push(chunk.data);
      }
    }

    results.push(Buffer.concat(buffers));
  }

  return results;
}
```

This API reference provides comprehensive coverage of all available functionality in the `@travisvn/edge-tts` package. For additional examples and use cases, refer to the examples directory in the package repository.
