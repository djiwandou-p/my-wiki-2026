# How to Implement Strangler Fig pattern

![strangler.png](https://raw.githubusercontent.com/djiwandou-p/my-wiki-2026/main/wiki/images/1776926104919-strangler.png)

To implement the Strangler Fig pattern for migrating modules from Azure App Service A (live) to on-prem IIS B, introduce a routing facade/proxy first to intercept all traffic, then incrementally route paths to B while A remains operational. This enables zero-downtime handover with rollback via config changes.[^1]

## Step 1: Deploy Facade/Proxy

Place a reverse proxy in front of A as the single entry point (update DNS/clients to point here).

### Option 1: YARP (.NET Reverse Proxy) - Best for Your .NET Stack

Deploy as new Azure App Service (C1 tier) or on-prem alongside B.

```csharp
// Program.cs (.NET 8+)
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();
app.MapReverseProxy();
app.Run();
```

**appsettings.json** (initially 100% to A):

```json
{
  "ReverseProxy": {
    "Routes": {
      "legacy-route": {
        "ClusterId": "azure-a",
        "Match": { "Path": "{**catch-all}" }
      }
    },
    "Clusters": {
      "azure-a": { "Destinations": { "dest1": { "Address": "https://yourapp.azurewebsites.net/" } } }
    }
  }
}
```

Add B cluster later; route `/new-module` to `http://your-onprem-iis:80/`.[^2][^3]

### Option 2: Azure API Management (APIM)

- Create APIM instance; import A's OpenAPI.
- Policy example (path-based routing):

```xml
<choose>
  <when condition="@(context.Request.Url.Path.Contains('/legacy'))">
    <set-backend-service base-url="https://yourapp.azurewebsites.net" />
  </when>
  <when condition="@(context.Request.Url.Path.Contains('/new-module'))">
    <set-backend-service base-url="http://your-onprem-server/" />
  </when>
</choose>
```

Update custom domain/DNS to APIM.[^4]

### Option 3: NGINX (On-Prem Simple)

Install on Linux VM; config:

```
location /legacy/ { proxy_pass https://yourapp.azurewebsites.net/; }
location /new-module/ { proxy_pass http://localhost:8080/; }  # B's IIS
```

Point DNS to NGINX IP.[^5]

## Step 2: Modular Migration Process

- **Identify Modules**: Low-risk first (e.g., read-only APIs). Use logs to prioritize.
- **Dev on B**: Build/test module in IIS (e.g., ASP.NET Core hosted).
- **Route Shift**: Update proxy config (e.g., `/users/*` → B); deploy (seconds).
- **Validate**: Canary (5% traffic via query param), full switch.
- **Data Sync**: Anti-corruption layer (proxy transforms payloads).
- **Monitor/Rollback**: Azure Monitor + proxy metrics; revert route on issues.

| Phase | % to B | Duration | Checkpoints |
| :-- | :-- | :-- | :-- |
| 1: Proxy Live | 0% | 1 day | All traffic A |
| 2: First Module | 100% module | 1 week | Metrics match |
| 3: 50% Modules | 50% total | 1-2 months | User feedback |
| 4: Cutover | 100% | - | Decom A |

## SSL/Domain Integration

Proxy terminates SSL (cert from Azure/Let's Encrypt); forwards HTTP/HTTPS to backends. Custom domain on proxy—no changes for `azurewebsites.net`.[^6]

## Tools and Best Practices

- Feature Flags: LaunchDarkly for dynamic routing.
- Hybrid: Azure Arc for on-prem monitoring.
- GitHub Repo Example: Strangler-Fig-Azure for APIM/YARP starters.[^4]
- Scale: Proxy handles load; auto-scale Azure deployment.

Test in staging; full migration in 2-6 months depending on modules.[^7][^1][^10][^8][^9]



[^1]: https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig

[^2]: https://www.coherentsolutions.com/insights/yarp-dotnet-reverse-proxy-guide

[^3]: https://oneuptime.com/blog/post/2026-01-25-reverse-proxy-yarp-dotnet/view

[^4]: https://github.com/error505/Strangler-Fig-Azure

[^5]: https://www.scaledbydesign.com/blog/strangler-fig-migration

[^6]: https://www.softwareseni.com/strangler-pattern-implementation-guide-for-incremental-legacy-migration/

[^7]: https://capstone-s.com/strangler-pattern-migration/

[^8]: https://dev.to/kylegalbraith/how-to-breakthrough-the-old-monolith-using-the-strangler-pattern-63e

[^9]: https://softwarelogic.co/en/blog/strangler-fig-pattern-the-secret-to-successful-service-migration

[^10]: https://aws.amazon.com/blogs/architecture/seamlessly-migrate-on-premises-legacy-workloads-using-a-strangler-pattern/


