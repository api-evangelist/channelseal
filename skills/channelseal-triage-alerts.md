---
name: channelseal-triage-alerts
description: Work the alert queue — list open alerts by severity, understand the application and channel each one points at, and move each alert through its lifecycle (ACKNOWLEDGED, PROCESSING, ESCALATED, RESOLVED, CLOSED). Use for daily triage of ChannelSeal findings.
api: openapi/channelseal-platform-api-openapi.yml
operations: [getAlerts, getAlertById, getApplicationById, getChannelById, updateAlert]
author: API Evangelist (generated 2026-09-20; grounded in the provider's OpenAPI — not a ChannelSeal-published skill)
requires:
  env: [CHANNELSEAL_BASE_URL, CHANNELSEAL_TOKEN]
---

# Triage open alerts and move them through their lifecycle

## Before you start
- Bearer token with `scope: read write` (updating an alert is a write). Base: `https://{env}.channelseal.com/platform`.
- `updateAlert` is a PUT of the whole Alert object: fetch first, change only `status`, send everything back. There is no documented reopen rule and no undo — a status you set is what stays (conventions/channelseal-conventions.yml#reversibility).
- Never call `deleteAlerts` (DELETE /api/v1/alerts) in triage: it deletes every alert in the tenant with no recovery.

## Steps
1. **List what is open, worst first.** `GET /api/v1/alerts?severity=CRITICAL&pageable.page=0&pageable.size=100` → `getAlerts`; repeat for HIGH, then MEDIUM. Keep alerts whose `status` is UNRESOLVED.
2. **Read the alert.** `GET /api/v1/alerts/{id}` → `getAlertById`. `alertType` is one of UNKNOWN_SERVICE_PROVIDER, UNKNOWN_CHANNEL, NEW_CHANNEL_WITH_SENSITIVE_DATA, SENSITIVE_DATA_IN_CLEAR_TEXT, CLIENT_ID_ACROSS_APPLICATIONS; `applicationAltId` and `channelAltId` say where.
3. **Get the context.** `GET /api/v1/applications/{applicationAltId}` → `getApplicationById` (owning team, technical and business owners) and `GET /api/v1/channels/{channelAltId}` → `getChannelById` (endpoint, protocol, service provider).
4. **Acknowledge.** PUT the alert back with `status: ACKNOWLEDGED` → `updateAlert` (204/200). Include the unchanged `alertType`, `severity`, `timestamp`, `description` and ids.
5. **Route.** SENSITIVE_DATA_IN_CLEAR_TEXT and CLIENT_ID_ACROSS_APPLICATIONS on a CRITICAL alert → `status: ESCALATED` and notify the application's technicalOwnerEmail; UNKNOWN_* types the owner confirms as expected → `status: RESOLVED`, then `CLOSED` once the inventory has been updated (see channelseal-import-and-inventory-apis).

## Output
One line per alert: id, type, severity, application, channel, action taken, new status. Errors: 404 means the alert or its application/channel was deleted since listing; 403 means the token lacks the write scope (errors/channelseal-problem-types.yml).
