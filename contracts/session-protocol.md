---
layout: default
title: Session protocol (API v5)
permalink: /contracts/session-protocol.html
---

# Session protocol (API v5)

This guide is the behavioral companion to [AsyncAPI](asyncapi.yaml). The
AsyncAPI schemas describe decoded values. Wire frames are binary MessagePack
using MessagePack-CSharp `Lz4BlockArray` compression.

This contract supports hAI! app version `1.7.4` and later. `serverVersion`
reports the server's full `major.minor.patch` release; the `/api/v5` path
identifies the API generation.

## Connection and framing

- Connect to `GET /api/v5/ai/session` and upgrade to WebSocket. Use `wss` over
  public networks. Authentication is determined by the server operator and is
  outside this contract.
- Each application message is one complete binary WebSocket message. Text
  WebSocket messages are invalid. A decoded union value is a two-element array
  `[tag, payload]`; object fields are string-keyed maps. Union tag values are
  listed below.
- Use a MessagePack-CSharp-compatible `Lz4BlockArray` encoder/decoder. The
  MessagePack extension/compression envelope is applied to each application
  message independently. HTTP MessagePack responses use the same compression.
- The initial client message must be `SessionCreateEvent` (client union tag 0).
  The server sends `SessionReadyEvent` (server union tag 1) after session
  configuration and any host-required startup checks succeed. Wait for this
  event before sending ordinary input.
- Clients should handle new union tags and unknown catalog fields defensively.
  The server closes the WebSocket when a session ends. A session error is
  terminal; stop input and playback and start a new connection.

## Client events

Client events use `IClientSessionEvent` tags:

| Tag | Event | Payload fields |
| --- | --- | --- |
| 0 | `SessionCreateEvent` | `configs: AiConfig[]` |
| 1 | `MessageDeltaEvent` | `delta: ChatMessage`, `isComplete: boolean` |
| 2 | `AddMessageEvent` | `message: ChatMessage` |
| 3 | `ForceBeginTurnEvent` | `messages: ChatMessage[]`, `functionResponses: FunctionResponse[]` |
| 4 | `AcknowledgeUserTurnEvent` | Empty map `{}` |

`ChatMessage` is also a union array: `[0, {role, text}]` for a text message or
`[1, {role, content, mimeType}]` for media. Roles are `system`, `user`, or
`assistant`. `content` is a MessagePack binary value. Supported image MIME types
in LLM history are `image/png`, `image/jpeg`, `image/webp`, and `image/gif`, for
user messages. Live audio input uses `mimeType: "audio/pcm"`, mono signed PCM16
little-endian samples, and an even number of bytes per delta.

### Session configurations

Each config in `SessionCreateEvent.configs` is an `AiConfig` union array:

| Tag | Config | Fields |
| --- | --- | --- |
| 0 | `LlmConfig` | `modelId`; optional `thinkingLevel`, `instructions`, `messages`, `tools` |
| 1 | `LiveLlmConfig` | LLM fields plus required `voiceName` |
| 2 | `TtsConfig` | `modelId`, `language`, `voiceName`; optional `prompt` |

Provide one config for LLM, Live LLM, or TTS-only; an LLM may be paired with one
TTS config. Live LLM cannot be combined with another config. Model IDs and voice
names come from the catalog or voice endpoint. Catalog order is not significant;
IDs are stable identifiers. Do not start generation by sending history: history
sets context only.

Thinking levels are `minimal`, `low`, `medium`, and `high`; omitted means
`minimal`. A server may reject levels unsupported by its model. `isRetired: true`
catalog entries should not be selected for new sessions.

### Tools

`tools` contains `ILlmToolDefinition` union arrays: `[0, {}]` enables the
server-owned `brave_search` tool where supported; `[1, LlmFunction]` defines a
client-executed function. `LlmFunction` fields are `id` (required), optional
`description`, and optional `parameterSchema` (JSON Schema subset: `type`,
`format`, `title`, `description`, `enum`, `maxItems`, `minItems`, `properties`,
`required`, `maxProperties`, `minProperties`, `maxLength`, `minLength`,
`pattern`, `example`, `anyOf`, `items`, `maximum`, `minimum`).

The server emits `FunctionCallsEvent` for client-owned calls. Each call has `id`,
`toolId`, and optional `arguments` (JSON text). Return results in
`ForceBeginTurnEvent.functionResponses`, preserving the call `id`; `data` is
optional text. A response is accepted once per pending call. After all pending
results arrive, the server continues generation. Tool execution failures should
be returned as tool result data so the model can respond; they are not session
errors.

## Server events

Server events use `IServerSessionEvent` tags:

| Tag | Event | Payload fields |
| --- | --- | --- |
| 0 | `MessageDeltaEvent` | `delta: ChatMessage`, `isComplete: boolean` |
| 1 | `SessionReadyEvent` | `serverVersion`; optional `inputAudioFormat`, `outputAudioFormat` |
| 2 | `FunctionCallsEvent` | `functionCalls: FunctionCall[]` |
| 3 | `UserTurnStartEvent` | Empty map `{}` |
| 4 | `UserTurnEndEvent` | Empty map `{}` |
| 5 | `UserTurnTranscriptionEvent` | `transcript: string` |
| 6 | `TurnGenerationStartEvent` | Empty map `{}` |
| 7 | `TurnGenerationEndEvent` | Empty map `{}` |
| 8 | `UsageEvent` | `talkTimeBalance: number` (optional billing integration) |
| 9 | `ErrorEvent` | `code`, `error` |

Text and audio deltas end with an empty delta and `isComplete: true`. The event
marks stream completion, not playback completion. `TurnGenerationEndEvent` marks
server-side generation completion; speech playback can continue after it when an
LLM session includes TTS. For tool calls, send all results before expecting the
automatic continuation. Live sessions use VAD: `UserTurnStartEvent` means stop
playback and discard queued assistant output; `UserTurnEndEvent` follows VAD
speech completion. `AcknowledgeUserTurnEvent` currently has no additional data.

Audio formats in `SessionReadyEvent` are mono. Live input and output use 24 kHz
PCM16. TTS output uses the selected model's sample rate and is PCM16, except a
legacy WAV synthesis model, which reports `format: "wav"` and sends
`mimeType: "audio/wav"`. Text-only sessions have null audio formats.

## Errors and host policy

`ErrorEvent` is terminal. Its `error` field is display-ready text. Defined codes
are `unknown`, `configuration_failed`, `invalid_request`, `provider_failed`,
`session_timed_out`, and `safety_blocked`. The current hosted service also
defines `insufficient_talk_time`, `registration_required`, and
`accounting_failed` for its access and billing integrations. Those integrations
are outside this contract; a self-hosted implementation need not implement those
features or emit those billing-specific codes. If the WebSocket is already
unavailable, delivery of an error event cannot be guaranteed.

Authentication, authorization, account state, billing, rate limits, and
registration are deployment choices. This contract does not include their
endpoints or prescribe credentials. Do not expose provider credentials through
the catalog or session messages.
