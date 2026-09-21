---
name: channelseal-import-and-inventory-apis
description: Import an API specification into ChannelSeal, then find the resulting API record and the channels (endpoints) it exposes. Use when onboarding a third-party or internal API into the Interface Scorecard inventory.
api: openapi/channelseal-api-catalog-api-openapi.yml
operations: [importSpec, searchApis, getApiById, searchChannels, getChannelById]
author: API Evangelist (generated 2026-09-20; grounded in the provider's OpenAPI — not a ChannelSeal-published skill)
requires:
  env: [CHANNELSEAL_BASE_URL, CHANNELSEAL_TOKEN]
---

# Import an API specification and inventory what it exposes

## Before you start
- Get a bearer token with the OAuth 2.0 client-credentials flow: POST https://dev-channelseal.us.auth0.com/oauth/token with your client_id, client_secret, `audience: https://api.channelseal.com` and `scope: read write`. Tokens last 3600 s. (authentication/channelseal-authentication.yml)
- The Catalog API and the Platform API live under different bases: `https://{env}.channelseal.com/catalog` and `https://{env}.channelseal.com/platform` where env is `api` (production) or `uat`.
- Every error is `application/problem+json` (errors/channelseal-problem-types.yml). On 429 honour `Retry-After`.
- There is no idempotency key: importing the same specification twice may create a second record. Search before you import.

## Steps
1. **Check whether the API is already known.** `GET {platform}/api/v1/apis/search?search=<name or domain>&pageable.page=0&pageable.size=20` → `searchApis`. If `content[]` is non-empty, skip to step 4 with that record's `metadata.altId`.
2. **Import the specification.** `POST {catalog}/v1/api-specifications/import` with an `ImportContext` body → `importSpec`. The response is the import result; a 400 problem means the specification could not be parsed.
3. **Find the imported record.** Re-run `searchApis` with the specification title or the service provider's domain and take `content[0].metadata.altId`.
4. **Read the API record.** `GET {platform}/api/v1/apis/{id}` → `getApiById`. Note `specification` (OPENAPI, ASYNCAPI, GRAPHQL, WSDL…), `securityScheme`, `type` (PARTNER/PRIVATE/PUBLIC/VENDOR) and `serviceProvider`.
5. **List the channels it exposes.** `GET {platform}/api/v1/channels/search?search=<api name>&pageable.page=0&pageable.size=100` → `searchChannels`. Each channel is one endpoint with `protocol`, `method`, `uri` and `type`.
6. **Inspect a channel.** `GET {platform}/api/v1/channels/{id}` → `getChannelById` to see its `service`, `api` and any `clientId` already observed on it.

## Output
Report the API altId, its specification type and security scheme, and the count of channels by method. Do not delete anything in this flow — `deleteAllApis` and `deleteAllChannels` exist and have no undo (conventions/channelseal-conventions.yml#reversibility).
