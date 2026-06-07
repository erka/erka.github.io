+++
slug = 'ofrep-updates'
title = 'OFREP gets streaming support'
description = 'Adding real-time flag updates to the OpenFeature Remote Evaluation Protocol'
date = 2026-06-07
tags = ['openfeature']
+++

{{< figure src="cover.svg" alt="sequence diagram" title="OFREP streaming lifecycle" class="aside" >}}

### Background

[OpenFeature](https://openfeature.dev) is a standard abstraction layer for feature flag management. Applications integrate with one SDK, and teams can swap flag backends without changing application code. The [OpenFeature Remote Evaluation Protocol](https://openfeature.dev/docs/reference/other-technologies/ofrep/) (OFREP) extends this to a RESTful API that defines how applications talk to flag management systems. If your service implements it, any community-maintained OFREP provider works out of the box.

OFREP has two evaluation endpoints. `POST /ofrep/v1/evaluate/flags/{key}` evaluates a single flag with per-request context - server-side providers use this for dynamic targeting. `POST /ofrep/v1/evaluate/flags` evaluates all flags at once with a static context. Client-side providers call this once and cache the result.

Caching on the client side created a detection problem. The original approach was polling - refetch on a timer. Poll too often and you waste resources, poll too infrequently and users see stale values.

### The polling problem

Client-side providers (the JavaScript SDK, the iOS SDK) send the evaluation context once and cache the response. If a flag value changes on the server, the client doesn't know until the next poll cycle.

```json
{
  "flags": [
    {
      "key": "discount-banner",
      "value": false,
      "reason": "STATIC",
      "variant": "default"
    }
  ]
}
```

When this flag changes, the update should propagate immediately. With polling, the propagation delay equals the poll interval. Shorter intervals increase server load and battery drain. Not great.

### ADR-0008: the streaming protocol

The protocol extension started as ADR-0008, a proposal by Jonathan Norris to address the polling gap without breaking existing implementations. The solution is minimal and backward-compatible. The bulk evaluation response gains an optional `eventStreams` array:

```json
{
  "flags": [
    {
      "key": "my-flag",
      "value": false,
      "reason": "DEFAULT",
      "variant": "disabled"
    }
  ],
  "eventStreams": [
    {
      "type": "sse",
      "endpoint": {
        "requestUri": "/stream"
      }
    }
  ]
}
```

Each entry describes a real-time notification channel. The `type` identifies the mechanism - currently only `sse` is defined. Providers ignore unknown types for forward compatibility. Each entry also provides connection details: either a direct `url` or a structured `endpoint` block. The example above uses `endpoint` with a `requestUri` that providers resolve against their configured OFREP base origin.

When the provider sees `eventStreams`, it connects using the `EventSource` API and receives events:

```
event: message
data: {"etag":"abcdef1","type":"refetchEvaluation"}
```

The `type` inside `data` tells the provider what to do. The only defined type is `refetchEvaluation`: re-fetch the bulk evaluation. The optional `etag` and `lastModified` carry over to the re-fetch as query parameters.

### Re-fetching with cache validation

On receiving a `refetchEvaluation` event, the provider sends the `etag` and `lastModified` as query parameters on the bulk evaluation POST:

```
POST /ofrep/v1/evaluate/flags?flagConfigEtag=abcdef1
```

The server can return a full 200 with new flag values, or a 304 Not Modified if the cached flags are still current.

The `flagConfigEtag` and `flagConfigLastModified` query parameters are only used on re-fetches triggered by SSE events. An unconditional re-fetch — for example, on cold start or tab resume after inactivity — omits them so the server always returns a full response.

### Building a minimal demo

There is a [minimal demo](https://github.com/erka/ofrep-sse) that shows the full round-trip. A single nginx instance serves three roles: the static frontend, the OFREP API, and the SSE stream.

**flags.json** - the bulk evaluation response with `eventStreams` pointing to `/stream`:

```json
{
  "flags": [
    {
      "key": "my-flag",
      "reason": "DEFAULT",
      "variant": "true",
      "value": true
    }
  ],
  "eventStreams": [
    {
      "type": "sse",
      "endpoint": { "requestUri": "/stream" }
    }
  ]
}
```

**stream.txt** - static SSE events. When `flags.json` is edited to change `my-flag` from `true` to `false`, `stream.txt` is updated to emit a `refetchEvaluation` event:

```
event: message
data: {"etag":"abcdef1","type":"refetchEvaluation"}
```

**main.ts** - the client side using `@openfeature/ofrep-web-provider`:

```typescript
import { ClientProviderEvents, OpenFeature } from "@openfeature/web-sdk";
import { OFREPWebProvider } from "@openfeature/ofrep-web-provider";

const app = document.getElementById("app")!;

async function main() {
  const client = OpenFeature.getClient();
  let prev: boolean | null = null;

  const display = () => {
    const enabled = client.getBooleanValue("my-flag", false);
    if (enabled === prev) return;
    prev = enabled;
    const color = enabled ? "gold" : "purple";
    app.innerHTML = `<code>my-flag</code>: <b style="color:${color}">${enabled}</b>`;
  };

  OpenFeature.addHandler(ClientProviderEvents.Ready, display);
  OpenFeature.addHandler(ClientProviderEvents.ConfigurationChanged, display);
  OpenFeature.setContext({ targetingKey: "asdfgh", plan: "pro" });

  const provider = new OFREPWebProvider(
    { baseUrl: window.location.origin },
    console,
  );
  OpenFeature.setProvider(provider);
}
```

`ClientProviderEvents.ConfigurationChanged` is the key piece. The provider emits this when it receives a `refetchEvaluation` event and re-fetches the flags. The handler calls `display()` which reads the updated value.

Start the demo, open the browser, and you see `my-flag: true` in bold gold. Edit `flags.json` to set `"value": false`, update `stream.txt` to emit a new event, and the browser switches to `my-flag: false` in purple - no polling, no refresh.

### Design considerations

A few design details are worth noting.

**Backward compatibility.** The whole feature is additive. Servers that don't support SSE omit the `eventStreams` field; providers fall back to polling. Unknown event stream types are ignored. Unknown event types inside `data` are ignored.

**inactivityDelaySec.** The response can include an `inactivityDelaySec` field (default 120). When the browser tab is hidden (Page Visibility API), the provider starts a timer. If the timer fires before the tab becomes visible again, the provider closes the SSE connection. When the tab comes back, it reconnects and does a full unconditional re-fetch.

**Reconnection.** If the SSE connection drops, the provider retries. If polling is configured as a fallback, it polls while reconnecting.

**Coalescing.** Multiple SSE events arriving in rapid succession collapse into a single re-fetch. This prevents redundant requests when the server batches configuration changes.

**Context change.** When the evaluation context changes (e.g., the user logs in), the provider re-fetches unconditionally and re-reads the `eventStreams` array. The SSE connection may now point to a different URL, or the response may omit streaming entirely.

**Provider support.** The web provider implemented streaming first. iOS and Android providers don't support SSE yet — they fall back to polling as the spec allows.

### Conclusion

SSE support in OFREP is a small protocol change that removes a frustrating trade-off. Providers get real-time updates without sacrificing backward compatibility. The [ofrep-sse](https://github.com/erka/ofrep-sse) demo shows the full round-trip with just nginx and static files.

#### Relevant links

- [OFREP protocol spec](https://github.com/open-feature/protocol) - OpenAPI spec
- [ofrep-web-provider](https://github.com/open-feature/js-sdk-contrib) - the `@openfeature/ofrep-web-provider` package
- [ADR-0008 proposal](https://github.com/open-feature/protocol/issues/62) - original design discussion
