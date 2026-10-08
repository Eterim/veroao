---
name: vero
description: Integrate with the Vero API (AGT-certified electronic invoicing for Angola) - create customers, products, invoices, credit/debit notes, receipts, pro-formas and webhooks. Use this whenever the user asks to "create an invoice", "integrate with Vero", "issue a fiscal document" (or the Portuguese "criar uma factura", "emitir documento fiscal"), or works with the @veroao/* SDKs.
---

# Vero - Electronic Invoicing API (Angola)

Vero is an invoicing platform certified by the AGT (Administração Geral Tributária, Angola's tax authority). This skill gives you what you need to integrate correctly without reading the whole documentation.

Fiscal terms stay in Portuguese, as in the API and on the documents: *factura* (invoice), *factura-recibo* (invoice-receipt), *nota de crédito/débito*, *recibo*, *pró-forma*, *NIF* (tax ID), *IVA* (VAT), *Consumidor Final* (anonymous end customer).

## Authentication

Two families of keys - never use one in place of the other:

- **`sk_...` (secret key)** - **server-to-server only**. Never expose it in the browser/frontend. Sent as `Authorization: Bearer sk_...`. Gives access to the organisation's whole API.
- **`pk_...` (publishable key)** - safe in the browser. Only works on `/v1/widget/*` routes. Used by `@veroao/widget`.

Keys are `live` or `test` (`vero_live_sk_...` / `vero_test_sk_...`). `test` keys never touch the real AGT - use them for development.

Base URL: `https://api.vero.ao`

## Key concepts before writing code

- **Money is always an integer in Kwanza cents (AOA).** 1,000.00 Kz = `100000`. Never send floats.
- **`taxRate` is optional and you should omit it when possible.** If sent, only `0`, `5` or `14` are accepted. If omitted, the server resolves it from the organisation's VAT regime (`ivaRegime`/`defaultTaxRate`, set in the dashboard under Definições → Perfil) - never hard-code `14`. Organisations in the `isento` or `simplificado` regime ("Exclusão" in the UI) **cannot** have lines with a rate other than `0`; the API rejects an incompatible rate with `422 tax_rate_incompatible_with_regime`. If the organisation has no regime configured, the fallback is `14%`.
- **`idempotencyKey`** - always send a unique value (e.g. the order ID in your system) when creating invoices. Prevents duplicates on network retries.
- **Consumidor Final**: for anonymous/counter sales, create or use a customer with `isConsumidorFinal: true` instead of inventing a NIF. The system assigns the generic NIF `999999999`.
- **`taxExemptionCode`** is required when `taxRate` ends up `0` (sent or resolved from the regime), except for Consumidor Final - the API rejects with `422 tax_exemption_code_required` if missing. Common codes: `M04` exclusion, `M22` health, `M21` education, `M13` books, `M30` exports.
- **Sandbox**: sandbox organisations (or `test` keys) never submit to the real AGT - `hash`, `atcud` and `agtStatus` come back `null` and the number has the `TESTE` prefix. Use it to test without AGT certification.
- **`customers.ensure()` never persists in sandbox**: with a `test` key it returns an ephemeral customer whose `id` looks like `sandbox_cus_...` (not a UUID), rebuilt from the request and not stored. Use that `id` as `customerId` when creating the invoice - it works - but don't expect to find it later with `GET /customers`.
- **Without its own AGT key configured, an organisation cannot issue any real fiscal document** (outside sandbox) - the API returns `422 agt_keys_not_configured`. This is intentional: each organisation signs its own documents, never with a shared key.
- **Withholding tax** (*retenção na fonte*, Art. 67 of the Industrial Tax Code) - optional, off by default. Pass `applyWithholdingTax: true` when creating an invoice or pro-forma (or editing a pro-forma). It only applies if the organisation enabled it in the dashboard (Definições → Perfil → Retenção na fonte) and the amount is above the configured minimum - never send the computed amount yourself, the server always resolves it. It does not change `subtotal`/`taxAmount`/`total` (those remain the official fiscal values); it comes separately in the response's `withholdingTax` field (`{ type, rate, amount, description }` or `null`). On a pro-forma it carries over to the invoice automatically when you convert it.

## Typical flow: create an invoice

```bash
# 1. Ensure the customer (creates or updates by externalId)
curl -X POST https://api.vero.ao/v1/organisations/{orgId}/customers/ensure \
  -H "Authorization: Bearer sk_..." -H "Content-Type: application/json" \
  -d '{"externalId":"user_123","name":"Empresa ABC","taxId":"5000123456","email":"financeiro@empresaabc.ao"}'

# 2. Create the invoice
curl -X POST https://api.vero.ao/v1/organisations/{orgId}/invoices \
  -H "Authorization: Bearer sk_..." -H "Content-Type: application/json" \
  -d '{
    "customerId": "customer-uuid",
    "documentType": "FT",
    "idempotencyKey": "order_789",
    "items": [
      { "description": "Serviço X", "quantity": 1, "unitPrice": 15000000 }
    ]
  }'
```

The response includes `number` (e.g. `"FT FT6326S62896N/1"`), `pdfUrl`, `atcud`, `hash` and `agtStatus` (`pending` for a moment, then `validated` or `rejected` - submission to the AGT is asynchronous).

## Node.js SDK (recommended for backends)

```bash
npm install @veroao/node
```

```ts
import { createVeroClient } from '@veroao/node'
const vero = createVeroClient({ secretKey: process.env.VERO_API_KEY! })

const customer = await vero.customers.ensure(orgId, { externalId: 'user_123', name: 'Empresa ABC', taxId: '5000123456' })
const invoice = await vero.invoices.create(orgId, {
  customerId: customer.id,
  documentType: 'FT',
  items: [{ description: 'Serviço X', quantity: 1, unitPrice: 15000000 }],
  idempotencyKey: `order_${orderId}`,
})
```

## React SDK (`@veroao/react`) and Widget (`@veroao/widget`)

- `@veroao/react` - hooks for React apps that already have their own backend calling the API (the secret key never goes to the browser).
- `@veroao/widget` - to embed directly in a third-party site with the **publishable** key (`pk_`). No build step: `<script src=".../widget.iife.js">`. The only legitimate case for calling Vero directly from the browser.

## Documents

| Type | Base endpoint | Note |
|---|---|---|
| Invoice (FT) / Invoice-receipt (FR) | `/v1/organisations/{orgId}/invoices` | `documentType: 'FT'` or `'FR'` |
| Credit note | `/v1/organisations/{orgId}/invoices/{invoiceId}/credit-note` | Cancels/corrects an issued invoice |
| Debit note | `/v1/organisations/{orgId}/debit-notes` | Standalone document. **No VAT** (DP 71/25, art. 3): lines at 0%, `taxExemptionCode` defaults to `M02`; `taxRate` 5/14 → `422 debit_note_tax_not_allowed` |
| Receipt | `/v1/organisations/{orgId}/invoices/{invoiceId}/receipt` | Confirms payment of an FT (not an FR, which is already paid) |
| Cancel receipt | `/v1/organisations/{orgId}/receipts/{id}/cancel` | Receipt issued by mistake. `{ "reason": "N" \| "I" }` - N: not sent to the customer, I: wrong customer (the only reasons the AGT accepts). The invoice becomes unpaid again. SDK: `vero.receipts.cancel(orgId, id, { reason: 'N' })` (@veroao/node ≥ 1.2) |
| Cancel debit note | `/v1/organisations/{orgId}/debit-notes/{id}/cancel` | Debit note issued by mistake. `{ "reason": "N" \| "I" }` - same two legal reasons as receipts; otherwise it cannot be cancelled. Leaves the totals and, if already validated, the cancellation is reported to the AGT (`agtCancelStatus`). SDK: `vero.debitNotes.cancel(orgId, id, { reason: 'I' })` (@veroao/node ≥ 1.3) |
| Pro-forma | `/v1/organisations/{orgId}/proformas` | Not fiscal - use `.../{id}/convert` to turn it into a real invoice |
| Delivery note | `/v1/organisations/{orgId}/invoices/{invoiceId}/delivery-note` | |

All return a PDF at `.../{id}/pdf`, and there is a public unauthenticated version at `/v1/public/{type}/{id}/pdf` (for sending by email or direct link).

The PDF layout follows the template active in the organisation. Custom templates are built with `@veroao/invoice` - see the `vero-template` skill.

## Webhooks

```bash
curl -X POST https://api.vero.ao/v1/organisations/{orgId}/webhooks \
  -H "Authorization: Bearer sk_..." -H "Content-Type: application/json" \
  -d '{"url":"https://your-site.com/webhook","events":["invoice.issued","invoice.cancelled"]}'
```

Events: `invoice.issued`, `invoice.cancelled`, `proforma.converted`, `proforma.cancelled`.

Each delivery carries `X-Vero-Signature: sha256=...` (HMAC of the body with the `secret` returned on creation - store it, it is shown only once). Always verify the signature before trusting the payload. Automatic redelivery with backoff (1m, 5m, 30m, 2h, 8h), up to 6 attempts, if your endpoint doesn't answer `2xx`.

## Common errors

| Error | Meaning |
|---|---|
| `401 unauthorized` | Key invalid, revoked, or missing from the `Authorization` header |
| `403 forbidden` | A `pk_` key calling a route outside `/v1/widget/*` (cancelling invoices/receipts/debit notes requires the secret key, on the server) |
| `422 agt_keys_not_configured` | Organisation without its own AGT key - can't issue outside sandbox |
| `400 insufficient_stock` | Product with `trackStock: true` without enough quantity |
| `409 receipt_already_exists` | There is already an active receipt for this invoice (cancel it first if it was issued by mistake) |
| `400 invalid_reason` | Receipt or debit-note cancellation with a reason other than `N`/`I` |
| `409 already_cancelled` | The receipt or debit note was already cancelled |
| `409 agt_pending` | The AGT is still validating the document - try cancelling again in a few minutes |
| `422 tax_rate_incompatible_with_regime` | Line with `taxRate` ≠ 0 in an organisation in the `isento`/`simplificado` regime |
| `422 tax_exemption_code_required` | `taxRate` resolved to `0` but `taxExemptionCode` is missing (and it isn't Consumidor Final) |
| `422 invalid_tax_rate` | `taxRate` sent is not `0`, `5` or `14` |

## More

Full interactive documentation (Portuguese): `https://vero.ao/docs`. If you need an endpoint not summarised here, check it there before guessing the payload format. Changes that affect existing code (new fields, behaviour fixes) are logged at `https://vero.ao/docs/changelog` - the SDKs follow semver, but always check there before assuming a payload format that may be outdated.

User-facing text on Vero (dashboard, documents, error messages shown to end users) is in Portuguese (Angola); keep that when you write UI copy for Vero integrations.
