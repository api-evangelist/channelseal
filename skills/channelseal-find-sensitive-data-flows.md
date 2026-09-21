---
name: channelseal-find-sensitive-data-flows
description: Find which channels carry sensitive data of a given type or risk, in which direction, and which non-human identities and applications are behind those calls. Use for a data-posture review before an agent is allowed to call an interface.
api: openapi/channelseal-platform-api-openapi.yml
operations: [searchSensitiveDataElements, getSensitiveDataElementById, getChannelById, getNonHumanIdentities_1, getApplications_1]
author: API Evangelist (generated 2026-09-20; grounded in the provider's OpenAPI — not a ChannelSeal-published skill)
requires:
  env: [CHANNELSEAL_BASE_URL, CHANNELSEAL_TOKEN]
---

# Find where sensitive data flows and which identities carry it

## Before you start
- Bearer token from the client-credentials flow with `scope: read` (authentication/channelseal-authentication.yml). Base: `https://{env}.channelseal.com/platform`.
- All list operations page with `pageable.page` / `pageable.size` (max 100 per the docs) and return `content[]` plus `page{totalElements,totalPages}`; keep paging until `page.number + 1 == page.totalPages`.
- Read-only flow: nothing here writes.

## Steps
1. **Search sensitive data elements.** `GET /api/v1/sensitive-data-elements/search?search=<info type, e.g. SSN or email>&pageable.page=0&pageable.size=100` → `searchSensitiveDataElements`. Each element has `elementLocation` (URI, PATH, QUERY, HEADER, BODY), `messageDirection` (RECEIVE/SEND), `likelihood`, `risk` (HIGH/MEDIUM/LOW), `channelAltId` and `serviceProviderAltId`.
2. **Filter for what matters.** Keep elements with `risk: HIGH` or `messageDirection: SEND` (data leaving toward a third party). For each, `GET /api/v1/sensitive-data-elements/{id}` → `getSensitiveDataElementById` to read `sensitiveInfoType` (its `classification`, `jurisdiction`) and `detectorUsed`.
3. **Resolve the channel.** `GET /api/v1/channels/{channelAltId}` → `getChannelById`: the endpoint (`uri`, `protocol`, `method`), its `service` and `api`, and the `serviceProvider` behind it.
4. **Find the identities calling it.** `GET /api/v1/channels/{id}/non-human-identities?pageable.page=0&pageable.size=100` → `getNonHumanIdentities_1`. Each NHI has `identityType` (API_KEY, CLIENT_ID, SERVICE_ACCOUNT_ID, USER_ID, X509_CERTIFICATE_FINGERPRINT) and a `clientId`.
5. **Map identities to applications.** For each NHI, `GET /api/v1/non-human-identities/{id}/applications` → `getApplications_1` to name the owning applications and their teams.

## Output
A table of channel → sensitive info type → direction → risk → NHIs → applications/teams. Flag any channel where a SEND-direction HIGH-risk element is carried by an identity attached to more than one application (the platform's own CLIENT_ID_ACROSS_APPLICATIONS alert type).
