# QuickChat AI vs LobeHub: Microservice Decomposition and Feature Gap Analysis

## 1. Scope and Source of Truth

This document summarizes how the QuickChat AI product documents map against the current LobeHub repository, with the QuickChat documents treated as the primary source of truth.

The intended product is not a direct fork of LobeHub's business model. It is a Persian-first AI access layer for users in Iran, with local authentication, local payments, pay-as-you-go credits, OpenRouter-backed managed model access, domestic fallback infrastructure, Telegram access, and localized operational tooling.

LobeHub is best treated as a strong AI workspace foundation for chat, agents, model runtime, file attachments, image/video generation, memory, and integrations. The Iran-specific commercial and reliability layers should be implemented as separate services.

---

## 2. Important Findings from the Current LobeHub Repository

In the current LobeHub repository:

- `ENABLE_BUSINESS_FEATURES` is set to `false`.
- `topUpRouter` and `spendRouter` are empty routers.
- ZarinPal appears only in localization strings, not as a real payment integration.
- OpenRouter exists as a provider/runtime adapter, but not as a full managed credit-wallet gateway.
- STT exists through Browser Web Speech API and OpenAI Whisper, but not as a domestic Persian-first STT service.

This means that LobeHub contains useful infrastructure, but the QuickChat-specific billing, routing, entitlement, payment, and domestic fallback rules are not available as production-ready modules in this repo.

---

## 3. Recommended Microservices

| Proposed service | Responsibilities according to QuickChat documents | LobeHub status | Recommendation |
|---|---|---|---|
| `auth-service` / `otp-service` | Mobile number registration, mobile OTP login, mobile password login, password recovery via OTP, OTP expiry, resend flow, rate limiting, fraud monitoring | LobeHub has Better Auth with email/password and email OTP. Phone fields exist in the schema, but SMS OTP is not implemented. | Build as a separate service or wrap LobeHub auth with a phone/SMS OTP service. |
| `billing-service` / `wallet-service` | Credit wallet, credit packs, credit batches, FIFO expiry, no negative balance, charge-on-success, free daily credits, transaction ledger, refunds | Current repo has stubs and UI copy, but no working wallet ledger or spend/top-up API. | Build as a core standalone service. |
| `payment-service` / `zarinpal-service` | ZarinPal payment creation, callback verification, idempotent verify, VAT handling, ZarinPal reference IDs, refund of unused credits | ZarinPal is present only in translations/locales. | Build as a standalone service. |
| `catalog-service` | Model catalog, model availability, free whitelist, paid catalog, OpenRouter daily price sync, representative cost in credits and Toman | LobeHub has model-bank and OpenRouter model fetching, but not QuickChat's free/paid/toman pricing policy. | Build separately and expose a clean catalog API to the app. |
| `ai-router` / `inference-facade` | Single entry point for inference, OpenRouter routing, provider health, domestic fallback, wallet preflight, post-success debit, cost calculation | LobeHub has a generic RouterRuntime and provider adapters, but not QuickChat's entitlement and fallback policy. | Build as a core service and reuse LobeHub runtime ideas where useful. |
| `domestic-model-service` | Self-hosted model inside Iran, enabled only during international outage for paid users, zero-credit fallback, queue/capacity management | No QuickChat-style domestic fallback policy exists in LobeHub. | Build as a new service. |
| `stt-service` | Persian-first voice-to-text, domestic processing, mobile-friendly short audio transcription | LobeHub supports Browser STT and OpenAI Whisper. | Build a domestic STT service and connect it to chat input. |
| `media-job-service` | Image/audio/video generation, async queue, status tracking, history, wallet preflight, debit on completion | LobeHub already has strong image/video generation; audio generation is mostly TTS, not music/audio generation. | Split media workloads into a service when integrating billing and queues. |
| `telegram-adapter` | Telegram bot access, account linking, message relay, AI responses, optional history sync | LobeHub has Telegram channel and messenger infrastructure. | Reuse concepts, but connect to QuickChat auth, wallet, conversation, and router services. |
| `file-service` | Uploads, file metadata, secure links, size/type policy, future scanning, chat association | LobeHub has strong file attachment support. | Reuse concepts; centralize policy in a file service. |
| `prompt-service` | Prompt templates, search, categories, prompt detail, insert/start-chat flow | LobeHub has task templates, but not a full user-facing prompt library. | Build as a new service/module. |
| `agent-catalog-service` | Specialized Persian agents, categories, recommended models, workflows, disclaimers | LobeHub has agent marketplace, system prompts, persona, and disclaimers. | Reuse the agent model, add Persian domain content and safety rules. |
| `cms-service` | Landing pages, FAQ, blog, legal pages, contact, feedback | LobeHub mostly uses static docs and external content. | Build separately. |
| `admin-api` / `ops-console` | User admin, wallet admin, model availability, billing operations, SMS health, treasury, pricing config audit | LobeHub has primitives and RBAC-related pieces, but no QuickChat operator console. | Build separately. |
| `fx-feed-worker` / `treasury-worker` | Daily USD/Toman rate, pricing config updates, Toman to USDT workflow, OpenRouter balance monitoring | Not present in LobeHub. | Build separately. |

---

## 4. Features Missing from LobeHub That Must Be Rebuilt

### 4.1 Mobile OTP Authentication

QuickChat requires:

- Mobile number registration with OTP.
- Login with mobile number and OTP.
- Login with mobile number and password.
- Password recovery via mobile OTP.
- OTP expiry timer.
- Resend OTP flow.
- Rate limiting and fraud prevention.
- SMS provider monitoring.

LobeHub currently provides email/password and email OTP flows. It has database fields for phone numbers, but the actual SMS OTP login/signup flow is not implemented.

### 4.2 Pay-As-You-Go Credit Wallet

QuickChat requires:

- A single wallet per user.
- `1 Credit = $0.0001 USD` face value.
- Integer credits with ceiling rounding.
- Paid credit packs.
- Free daily 200-credit allotment.
- Credit batches with 365-day expiry.
- FIFO credit consumption.
- No negative balances.
- Charge only after successful responses.
- Transaction history.
- Refund handling for unused credits within 7 days.

The current LobeHub repo does not include this wallet ledger or pack lifecycle. The related business routers are empty stubs.

### 4.3 ZarinPal Payment Integration

QuickChat requires:

- Pack purchase through ZarinPal.
- Payment callback and verification.
- Idempotent verify.
- VAT line item.
- ZarinPal reference ID.
- Refund support for unused credits.

The LobeHub repo does not include a real ZarinPal payment implementation.

### 4.4 Pricing Configuration and FX Handling

QuickChat requires a centralized `pricing.config.yaml` consumed by billing, catalog, and AI routing:

- Daily USD/Toman FX updates.
- Flat 5,000 Toman FX buffer.
- ZarinPal merchant fee.
- VAT.
- Credit value.
- Consumption markup.
- Free-tier limits.
- Paid-tier caps.
- Pack bonuses.
- OpenRouter platform fee.
- USDT conversion fee.

LobeHub has model pricing utilities, but not the QuickChat pricing model with Toman display and centralized Iranian-market pricing rules.

### 4.5 Domestic Fallback

QuickChat requires:

- International health checks.
- `INTL_UP` / `INTL_DOWN` state machine.
- Paid users get domestic fallback for free during international outage.
- Free users are blocked during international outage.
- Clear UI labels for fallback.
- Capacity and queue management for domestic infrastructure.

LobeHub has generic runtime fallback concepts, but not this product policy.

### 4.6 Managed OpenRouter Gateway

QuickChat requires OpenRouter to be used as a managed backend provider:

- Users do not bring their own OpenRouter API keys.
- The platform pays OpenRouter.
- Users pay QuickChat through credits.
- The router performs balance checks and deducts credits after successful responses.

LobeHub's OpenRouter integration is primarily a provider adapter/runtime integration. It is not the QuickChat credit-wallet gateway.

---

## 5. Existing LobeHub Capabilities That Need Product-Specific Changes

### 5.1 Core Chat

LobeHub already has strong chat infrastructure:

- Streaming responses.
- Message history.
- Model selection.
- Retry/regenerate/copy.
- File attachment.
- Agents.
- Memory/persona.
- Image/video creation routes.

Required QuickChat changes:

- Show active model cost in credits and Toman.
- Add wallet preflight checks.
- Handle insufficient-credit states.
- Enforce free/paid model gating.
- Show international outage and domestic fallback banners.
- Store `credits_consumed`, `cogs_usd`, and resolved model metadata on messages.
- Preserve drafts during payment or outage flows.

### 5.2 Model Catalog and OpenRouter

LobeHub has:

- A broad model-bank.
- OpenRouter model fetching.
- Provider/model settings pages.
- Runtime adapters.

Required QuickChat changes:

- Convert OpenRouter access to a managed catalog.
- Add free model whitelist.
- Add paid model catalog.
- Add availability states:
  - Available.
  - Limited.
  - Unavailable.
  - Internal fallback only.
  - Free-tier only.
  - Paid-tier only.
- Add daily price sync.
- Display representative Persian-message cost in credits and Toman.

### 5.3 Telegram

LobeHub has:

- Telegram bot channel support.
- Messenger account linking.
- Multi-platform bot infrastructure.

Required QuickChat changes:

- Define one main QuickChat Telegram bot flow.
- Link Telegram identity to QuickChat user account.
- Route Telegram messages through the same conversation and AI router services.
- Enforce wallet, free-tier, and outage rules in Telegram.
- Decide Telegram pricing policy.
- Sync history where supported.

### 5.4 Image Generation

LobeHub has:

- `/image` workspace.
- Generation topics and batches.
- Generation history.
- Multiple provider integrations.

Required QuickChat changes:

- Wallet preflight before generation.
- Credit debit after successful generation.
- Cost display before expensive requests.
- Model availability through `catalog-service`.
- History aligned with prompt, model, timestamp, thumbnail, and download metadata.

### 5.5 Video Generation

LobeHub has:

- `/video` workspace.
- Async polling.
- Generation history.
- Provider webhooks and background processing.

Required QuickChat changes:

- Wallet preflight.
- Debit on completion.
- Clear queued/processing/done/failed states.
- Expensive operation confirmation.
- Clear messaging when domestic fallback does not support video.

### 5.6 TTS and STT

LobeHub has:

- TTS support.
- Browser STT.
- OpenAI Whisper STT.

Required QuickChat changes:

- Domestic Persian-first STT.
- Mobile-first voice recording flow.
- Transcript review and edit before sending.
- Free-tier STT policy.
- Dedicated audio/music generation if required beyond TTS.

### 5.7 Specialized Agents

LobeHub has:

- Agent marketplace.
- System prompt configuration.
- Persona/memory.
- Opening questions.
- Disclaimers.

Required QuickChat changes:

- Persian domain agents:
  - Legal.
  - Psychology.
  - Travel.
  - Car maintenance.
  - English language.
  - SEO.
  - Marketing.
  - Education.
  - Coding.
- Sensitive-domain disclaimers.
- Recommended workflows.
- Recommended models.
- Catalog-driven availability and pricing.

### 5.8 Personalization and Meta-Prompt

LobeHub has:

- User memory.
- Persona.
- Input templates.
- Context injection.

Required QuickChat changes:

- Simpler Persian-first UI fields:
  - About me.
  - Occupation.
  - Interests.
  - Preferred response style.
  - Preferred language.
  - Goals.
- Clear product rule for when personalization is injected into chat.

### 5.9 File Attachments

LobeHub has:

- File upload.
- Chat attachments.
- Knowledge/file handling.

Required QuickChat changes:

- File size/type policy.
- Security scanning roadmap.
- Model compatibility errors.
- Storage and retention policy.
- Centralized file-service API.

---

## 6. Partial or Incomplete Matches

| QuickChat feature | LobeHub status | Required work |
|---|---|---|
| Prompt Library | LobeHub has task templates, but not a full searchable prompt library. | Build prompt-service and prompt UI. |
| Folder / Project Management | LobeHub has topic/project/cwd grouping and agent groups, but not simple chat folders as described. | Add folder/project abstraction or map it onto topics. |
| Mentioning Other Models with `@ModelName` | LobeHub supports `@agent`, `@topic`, `@tool`, but not `@model`. | Add model mention routing intent. |
| Audio Generation | TTS exists, but music/audio generation is missing. | Build media/audio generation support. |
| CMS / Blog / FAQ / Legal | Mostly static docs or external content. | Build CMS service. |
| Admin / Ops | Some primitives exist, but no QuickChat ops console. | Build admin-api and admin UI. |
| Google Meet | Google Docs/Drive/Calendar exist through Klavis; Google Meet is not present. | Add integration if required. |
| Wallet UI | LobeHub has cloud/business shells and localized copy, but no self-contained wallet implementation in this repo. | Build wallet and packs UI backed by billing-service. |

---

## 7. Suggested Implementation Priority

### Tier 1 — Product Foundation

1. `auth-service` with SMS OTP.
2. `billing-service` / wallet ledger.
3. `payment-service` with ZarinPal.
4. `catalog-service`.
5. `ai-router`.
6. Managed OpenRouter gateway.
7. Free-tier and paid-tier gating.
8. Domestic fallback policy.
9. Wallet and packs UI.
10. Profile/settings integration.

These are required before the product can behave according to the QuickChat documents.

### Tier 2 — Adoption and Retention

1. Prompt library.
2. Persian agent catalog.
3. File-service policy.
4. Folder/project abstraction.
5. Domestic Persian STT.
6. Simplified personalization.
7. Better chat history grouping.

### Tier 3 — Expansion

1. Telegram adapter productionization.
2. Full image/audio/video monetization.
3. `@model` mention.
4. CMS/blog/FAQ/legal.
5. Google Docs/Meet integrations.
6. Advanced admin/ops.
7. Treasury and FX automation.

---

## 8. Summary

LobeHub already provides a strong AI workspace foundation:

- Chat.
- Agents.
- Model runtime.
- OpenRouter adapter.
- File attachments.
- Image and video generation.
- TTS and basic STT.
- Telegram infrastructure.
- Memory/persona.
- Persian locale resources.

However, the QuickChat documents define a different product layer: a localized Iranian AI access platform.

The most important QuickChat-specific services are:

- Billing and wallet.
- ZarinPal payments.
- SMS OTP.
- Catalog and pricing.
- AI router and managed OpenRouter access.
- Domestic fallback.
- FX and treasury workers.
- Admin operations.
- CMS and localized public content.

The key architectural recommendation is to keep LobeHub as the user-facing AI workspace layer while moving the Iran-specific entitlement, routing, pricing, billing, and operational logic into separate microservices.
