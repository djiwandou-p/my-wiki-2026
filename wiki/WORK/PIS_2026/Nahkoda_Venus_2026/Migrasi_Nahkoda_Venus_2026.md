# Nahkoda to Venus Migration Strategy

## Prompt
I am thinking of this strategy to handover or migrate from web app A (the one that using azurewebsites.net) to web app B (other web app in IIS Server on-premise)

the condition is like this

* A is still under development of new feature
* A is live and still used by the users in production
* B is still in full development mode
modular approach, migrate some modules from A to B, while keeping A alive, give options or other references to the handover or strategy migration process


## Answer
Your modular migration from Azure App Service (A, live/production with dev ongoing) to on-prem IIS (B, full dev) requires a low-risk, incremental strategy to avoid downtime. The Strangler Fig pattern fits best: Gradually replace A modules with B via a routing facade, keeping A live until fully migrated.[^1]

## Strangler Fig Pattern (Primary Option)

Build a reverse proxy (e.g., Azure API Management, NGINX on-prem, or YARP in .NET) that routes traffic: Start with 100% to A, shift modules to B incrementally.

- Identify modules (e.g., /api/users → B, /dashboard → A).
- Proxy intercepts requests; route based on path/query (e.g., /new-module/* → B's IIS URL).
- Update DNS gradually or use weighted routing.
- Monitor with Azure Monitor/Application Insights; rollback by adjusting routes.

This hides migration from users; A stays live for stable features while B matures.[^2][^1]

## Branch by Abstraction

Refactor A into abstract interfaces per module during dev.

- Extract module to shared lib/repo, deploy to B first (test in isolation).
- Swap A's impl to call B's endpoint (hybrid phase).
- Migrate traffic module-by-module; remove A stubs last.

Ideal for .NET (your stack): Use HttpClientFactory for cross-calls.[^3]

## Blue-Green with Canary Releases

Deploy B modules in parallel; shift traffic safely.


| Phase | Traffic Split | Tools | Pros | Cons |
| :-- | :-- | :-- | :-- | :-- |
| Prep | 100% A | Azure Front Door/Traffic Manager + On-Prem Load Balancer | Zero downtime | Needs public endpoint for B |
| Canary | 5-10% to B module | Feature flags (e.g., LaunchDarkly) + Azure AD B2C | Test prod load | Complex setup |
| Full Cutover | 100% B | DNS TTL switch | Fast rollback | On-prem scaling limits |

Expose B via VPN/Hybrid Connection (Azure Arc) during transition.[^4]

## Domain/SSL Handling

- **Default azurewebsites.net**: No migration needed—users shift via proxy/DNS to B's IP/FQDN.
- **Custom Domain**: Update CNAME/A record to B's IIS (5-15 min TTL); re-issue SSL cert on B (Let's Encrypt free/auto).[^5]
- Hybrid: Proxy handles SSL termination; terminate on Azure Front Door for seamless handover.


## Additional Steps and Tools

1. **Shared Data**: Migrate DB to on-prem SQL (Azure Database Migration Service); use API Gateway for sync.
2. **CI/CD**: GitHub Actions for A → Azure; Azure DevOps for B → IIS.
3. **Monitoring**: Azure Monitor + on-prem tools (e.g., Serilog).
4. **References**:
    - Strangler Fig: Azure Architecture Center guide.[^1]
    - YARP Proxy: For .NET reverse proxy (GitHub samples).[^2]
    - Hybrid Connectivity: Azure Arc-enabled IIS servers.

Start with non-critical modules; aim for 20% shifts bi-weekly. Decommission A post-validation.[^6][^10][^11][^7][^8][^9]


[^1]: https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig

[^2]: https://learn.microsoft.com/en-au/answers/questions/2341457/how-to-deploy-strangler-fig-side-by-side-migration

[^3]: https://www.youtube.com/watch?v=u5Hspwt50O0

[^4]: https://www.seaflux.tech/blogs/blue-green-deployment-in-azure-cloud/

[^5]: https://learn.microsoft.com/en-us/azure/app-service/manage-custom-dns-migrate-domain

[^6]: https://stackoverflow.com/questions/4620149/how-can-i-migrate-an-azure-application-to-iis

[^7]: https://github.com/uglide/azure-content/blob/master/articles/app-service-web/web-sites-migration-from-iis-server.md

[^8]: https://www.linkedin.com/learning/configuring-windows-server-hybrid-advanced-services-az-801-cert-prep-by-microsoft-press/migrate-iis-workloads-to-azure-web-apps

[^9]: https://learn.microsoft.com/en-us/shows/migrating-from-aspnet-to-aspnet-core/deploying-an-incrementally-migrated-aspnet-app-to-azure-app-service-migrating-from-aspnet-to-aspnet-core

[^10]: https://learn.microsoft.com/en-us/azure/container-apps/blue-green-deployment

[^11]: https://github.com/rgl/azure-content/blob/master/articles/app-service-web/web-sites-migration-from-iis-server.md

