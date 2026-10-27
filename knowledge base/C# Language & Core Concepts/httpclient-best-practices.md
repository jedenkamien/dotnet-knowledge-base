# HttpClient Best Practices

## The Core Problem: Why Not `new HttpClient()` Per Call?

`HttpClient` implements `IDisposable`, which tempts developers to wrap it in `using` blocks:

```csharp
// DANGEROUS — causes socket exhaustion under load
using var client = new HttpClient();
var response = await client.GetAsync("https://api.example.com/data");
```

Even after `Dispose()`, the underlying TCP socket enters **`TIME_WAIT`** state for up to ~240 seconds (OS-level). Under load, you exhaust available ports → `SocketException`.

## The Three Approaches Compared

| Approach | Socket exhaustion | DNS refresh |
|---|---|---|
| `new HttpClient()` per call | Yes (bad) | Yes (fresh each time) |
| Static singleton | No | No — stale DNS (bad) |
| `IHttpClientFactory` | No | Yes (good) |

## DNS Context

DNS translates a hostname (`api.example.com`) into an IP address (`1.2.3.4`). The OS queries a DNS server at connection time. With a static `HttpClient`, this lookup happens once and the resolved IP is cached for the handler's lifetime — no further queries.

If the service's IP changes (cloud deployments, failovers, load balancer changes), a static handler keeps sending requests to the old IP until the app restarts.

DNS infrastructure is globally distributed — no single owner. Your OS typically uses your ISP's DNS server by default, but public alternatives exist (Google `8.8.8.8`, Cloudflare `1.1.1.1`).

## Recommended Solution: `IHttpClientFactory`

Registered via `AddHttpClient()` in `Program.cs` / `Startup.cs`:

```csharp
builder.Services.AddHttpClient<MyService>();
```

```csharp
public class MyService
{
    private readonly HttpClient _client;

    public MyService(HttpClient client)  // injected by factory
    {
        _client = client;
    }
}
```

The factory internally pools and recycles `HttpMessageHandler` instances (~2 min default). No custom factory code needed — `AddHttpClient()` wires everything up automatically.

**Result**: short-lived `HttpClient` instances (safe to create/dispose freely) backed by shared, periodically-recycled handlers.

## Fallback: Static Singleton (Simple Apps)

```csharp
private static readonly HttpClient _client = new HttpClient();
```

Acceptable when the target endpoint has a stable, fixed IP and DNS changes are not a concern.

## Key Takeaway

The real resource is the `HttpMessageHandler`, not `HttpClient` itself. `IHttpClientFactory` manages handler lifetime correctly — solving both socket exhaustion and DNS staleness.
