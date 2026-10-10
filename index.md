---
layout: default
title: hAI! Server API
---

# hAI! Server API

This site documents the public server contract used by hAI! clients. It is for
implementers of compatible servers, including self-hosted servers that connect
to local or third-party AI providers.

## Contract files

- [OpenAPI 3.1: HTTP endpoints](contracts/openapi.yaml) — model catalog and TTS
  voice discovery.
- [AsyncAPI 3.0: WebSocket endpoint](contracts/asyncapi.yaml) — session route and
  client/server message unions.
- [Session protocol guide](contracts/session-protocol.html) — MessagePack encoding,
  session configurations, event flow, tools, and errors.

The contract covers API version 5 under `/api/v5`. The auth, IAP, and migration
routes are intentionally outside this contract. Authentication and authorization
for the covered routes are deployment policy; a host may require credentials or
allow access according to its own setup. The contract does not define user
registration, purchases, TalkTime balance storage, or migration behavior.

This contract targets app release `5.5.9`. The server reports its full
`major.minor.patch` release as `serverVersion`; the paired server build is
`5.5.9`. The API generation remains `v5`.

To publish this site with GitHub Pages, choose the repository's Pages settings
to deploy from the desired branch and select the `/docs` folder.

## Compatibility notes

HTTP catalog and voice responses and all WebSocket application messages use
MessagePack with MessagePack-CSharp `Lz4BlockArray` compression. A compatible
implementation must follow the wire encoding in the protocol guide. The YAML
files describe the decoded MessagePack data model and mark the MessagePack
transport with `x-` extensions.

The catalog advertises configured model capabilities; it does not expose
provider credentials or require any particular provider. An implementation may
connect to Ollama or another provider while preserving the public model IDs and
session behavior expected by its hAI! client.
