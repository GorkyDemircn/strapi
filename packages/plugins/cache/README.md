# @strapi/plugin-cache

A Redis-backed Cache-Aside plugin for Strapi v5 that reduces 
database load on high-frequency public read endpoints.

## Problem

Strapi's current architecture routes every public GET request 
through the full stack — Middleware → Controller → Service → 
Document Service API → Query Engine → PostgreSQL — regardless 
of whether the content has changed since the last request.

Under high-traffic scenarios (e.g. a breaking news event on a 
media platform), this causes the PostgreSQL connection pool 
(capped at ~400-500 connections via PgBouncer) to become the 
scalability bottleneck — even when the Koa HTTP server scales 
horizontally without issue.

## Proposed solution

Insert a Cache-Aside component between the Service layer and 
the Document Service API, backed by the existing Redis 
container already provisioned in production deployments.

## Cache-Aside read path

When a public GET /api/{contentType} request reaches the 
Service layer:

1. Construct a deterministic cache key:
   strapi:cache:{contentType}:{locale}:{status}:{queryHash}

2. Issue GET to Redis:
   - Cache HIT → return cached response directly to Controller,
     bypassing Document Service API and Query Engine entirely
   - Cache MISS → proceed with normal Document Service API 
     invocation, store result in Redis with configurable TTL

## Cache invalidation

Invalidation is triggered by Strapi lifecycle hooks:

- afterPublish → flush strapi:cache:{contentType}:* via 
  Redis SCAN + DEL
- afterUnpublish → flush same pattern
- afterUpdate (draft) → no flush required (draft content 
  not served by public API)

## Implementation approach

The plugin registers itself via the Plugin Server API — 
no core modification required:

```javascript
// src/index.ts
export default {
  register({ strapi }) {
    // Wrap the existing service find() method
    const originalService = strapi.service(
      'api::article.article'
    );
    
    strapi.service('api::article.article').find = 
      async (params) => {
        const cacheKey = buildCacheKey('article', params);
        const cached = await redis.get(cacheKey);
        
        if (cached) return JSON.parse(cached);
        
        const result = await originalService.find(params);
        await redis.set(cacheKey, JSON.stringify(result), 
          'EX', 60);
        return result;
      };
  },
  
  bootstrap({ strapi }) {
    // Register lifecycle hooks for cache invalidation
    strapi.db.lifecycles.subscribe({
      models: ['*'],
      async afterPublish({ result, params }) {
        const pattern = `strapi:cache:${params.uid}:*`;
        await flushPattern(pattern);
      },
      async afterUnpublish({ result, params }) {
        const pattern = `strapi:cache:${params.uid}:*`;
        await flushPattern(pattern);
      }
    });
  }
};
```

## Expected impact

Based on the quantitative analysis in the architectural report:

- Database connection pressure: reduced from ~500 concurrent 
  connections to ~50 (first-request misses only)
- Response time: sub-millisecond (Redis) vs ~45ms (PostgreSQL)
- Redis utilization: increases from ~5% to meaningful capacity 
  usage
- Scalability bottleneck shifts from SQL connection ceiling 
  (hard limit) to Redis memory (cheap to scale)

## Architectural diagrams

See the attached C4 Component diagrams showing:
- As-Is: every request terminates at PostgreSQL
- To-Be: cache-hit requests served from Redis, 
  cache-miss requests populate Redis for subsequent hits

## References

- Strapi Plugin Server API: 
  docs.strapi.io/cms/plugins-development/server-api
- Strapi Lifecycle Hooks: 
  docs.strapi.io/cms/backend-customization/models#lifecycle-hooks
- Cache-Aside Pattern: 
  learn.microsoft.com/azure/architecture/patterns/cache-aside
- Redis Documentation: redis.io/docs
