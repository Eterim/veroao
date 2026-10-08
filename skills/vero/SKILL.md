---
name: vero
description: Integrar com a API da Vero (facturação electrónica certificada pela AGT em Angola) — criar clientes, produtos, facturas, notas de crédito/débito, recibos, proformas e webhooks. Usa isto sempre que o utilizador pedir para "criar uma factura", "integrar com a Vero", "emitir documento fiscal" ou trabalhar com o SDK @veroao/*.
---

# Vero — API de Facturação Electrónica (Angola)

A Vero é uma plataforma de facturação certificada pela AGT (Administração Geral Tributária de Angola). Este skill dá-te o essencial para integrar correctamente sem teres de ler toda a documentação.

## Autenticação

Duas famílias de chaves, nunca trocar o uso uma pela outra:

- **`sk_...` (secret key)** — uso **servidor-a-servidor apenas**. Nunca expor no browser/frontend. Enviada como `Authorization: Bearer sk_...`. Dá acesso a toda a API da organização.
- **`pk_...` (publishable key)** — segura para usar no browser. Só funciona nas rotas `/v1/widget/*`. Usada pelo `@veroao/widget`.

Chaves têm ambiente `live` ou `test` (`vero_live_sk_...` / `vero_test_sk_...`). Chaves `test` nunca tocam a AGT real — usa-as para desenvolvimento.

Base URL: `https://api.vero.ao`

## Conceitos importantes antes de escrever código

- **Valores monetários são sempre inteiros em cêntimos de Kwanza (AOA)**. 1.000,00 Kz = `100000`. Nunca enviar floats.
- **`taxRate` é opcional e, se possível, deves omiti-lo.** Se enviado, só aceita `0`, `5` ou `14`. Se omitido, o servidor resolve-o sozinho a partir do regime de IVA configurado na organização (`ivaRegime`/`defaultTaxRate`, definidos no dashboard em Definições → Perfil) — nunca assumas `14` no teu código. Organizações em regime `isento` ou `simplificado` (isto é, "Exclusão" na UI) **não podem** ter linhas com taxa diferente de `0`; a API rejeita com `422 tax_rate_incompatible_with_regime` se enviares uma taxa incompatível. Se a organização não tiver regime nenhum configurado, o fallback é `14%`.
- **`idempotencyKey`** — envia sempre um valor único (ex: o ID do pedido no teu sistema) ao criar facturas. Evita duplicados em caso de retry de rede.
- **Consumidor Final**: para vendas anónimas/balcão, cria ou usa um cliente com `isConsumidorFinal: true` em vez de inventar um NIF. O sistema atribui automaticamente o NIF genérico `999999999`.
- **`taxExemptionCode`** é obrigatório quando `taxRate` resultar em `0` (enviado ou resolvido a partir do regime), excepto para Consumidor Final — a API rejeita com `422 tax_exemption_code_required` se faltar. Códigos comuns: `M04` exclusão, `M22` saúde, `M21` ensino, `M13` livros, `M30` exportação.
- **Sandbox**: organizações de sandbox (ou chaves `test`) nunca submetem à AGT real — `hash`, `atcud` e `agtStatus` vêm `null`, o número tem o prefixo `TESTE`. Usa isto para testar sem certificação AGT.
- **`customers.ensure()` em sandbox nunca persiste**: com chave `test`, devolve um cliente efémero cujo `id` tem o formato `sandbox_cus_...` (não é UUID) — reconstruído a partir do próprio pedido, não fica gravado na base de dados. Usa esse `id` directamente como `customerId` ao criar a factura, funciona normalmente, mas não esperes encontrá-lo depois num `GET /customers`.
- **Sem chave AGT própria configurada, a organização não consegue emitir nenhum documento fiscal real** (fora do sandbox) — API devolve `422 agt_keys_not_configured`. Isto é intencional: cada organização assina os seus próprios documentos, nunca com uma chave partilhada.
- **Retenção na fonte** (Art. 67º do Código do Imposto Industrial) — opcional, desligada por omissão. Passa `applyWithholdingTax: true` ao criar (ou editar, só proformas) uma factura ou proforma. Só tem efeito se a organização a tiver activada no dashboard (Definições → Perfil → Retenção na fonte) e o valor ultrapassar o limiar mínimo configurado — nunca envies tu o valor calculado, é sempre resolvido no servidor. Não altera `subtotal`/`taxAmount`/`total` (esses continuam o valor fiscal oficial); viaja à parte no campo `withholdingTax` da resposta (`{ type, rate, amount, description }` ou `null`). Numa proforma, transporta-se automaticamente para a factura ao converteres — não precisas de pedir outra vez.

## Fluxo típico: criar uma factura

```bash
# 1. Garantir o cliente (cria ou actualiza por externalId)
curl -X POST https://api.vero.ao/v1/organisations/{orgId}/customers/ensure \
  -H "Authorization: Bearer sk_..." -H "Content-Type: application/json" \
  -d '{"externalId":"user_123","name":"Empresa ABC","taxId":"5000123456","email":"financeiro@empresaabc.ao"}'

# 2. Criar a factura
curl -X POST https://api.vero.ao/v1/organisations/{orgId}/invoices \
  -H "Authorization: Bearer sk_..." -H "Content-Type: application/json" \
  -d '{
    "customerId": "uuid-do-cliente",
    "documentType": "FT",
    "idempotencyKey": "order_789",
    "items": [
      { "description": "Serviço X", "quantity": 1, "unitPrice": 15000000 }
    ]
  }'
```

Resposta inclui `number` (ex: `"FT FT6326S62896N/1"`), `pdfUrl`, `atcud`, `hash`, `agtStatus` (fica `pending` por instantes, depois `validated` ou `rejected` — submissão à AGT é assíncrona).

## SDK Node.js (recomendado para backend)

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

## SDK React (`@veroao/react`) e Widget (`@veroao/widget`)

- `@veroao/react` — hooks para apps React que já têm o teu próprio backend a chamar a API (a chave secreta nunca vai para o browser).
- `@veroao/widget` — para embeber directamente num site de terceiros, com a chave **publishable** (`pk_`). Não precisa de build step: `<script src=".../widget.iife.js">`. Único caso legítimo de chamar a Vero directamente do browser.

## Documentos disponíveis

| Tipo | Endpoint base | Nota |
|---|---|---|
| Factura (FT) / Factura-Recibo (FR) | `/v1/organisations/{orgId}/invoices` | `documentType: 'FT'` ou `'FR'` |
| Nota de Crédito | `/v1/organisations/{orgId}/invoices/{invoiceId}/credit-note` | Anula/corrige uma factura já emitida |
| Nota de Débito | `/v1/organisations/{orgId}/debit-notes` | Documento independente. **Sem IVA** (DP 71/25, art. 3.º): linhas a 0%, `taxExemptionCode` por omissão `M02`; `taxRate` 5/14 → `422 debit_note_tax_not_allowed` |
| Recibo | `/v1/organisations/{orgId}/invoices/{invoiceId}/receipt` | Confirma pagamento de uma FT (não FR, que já é paga) |
| Anular recibo | `/v1/organisations/{orgId}/receipts/{id}/cancel` | Recibo emitido por engano. `{ "reason": "N" \| "I" }` - N: não enviado ao cliente, I: cliente mal identificado (únicos motivos que a AGT aceita). A factura volta a ficar por pagar. SDK: `vero.receipts.cancel(orgId, id, { reason: 'N' })` (@veroao/node ≥ 1.2) |
| Anular nota de débito | `/v1/organisations/{orgId}/debit-notes/{id}/cancel` | ND emitida por engano. `{ "reason": "N" \| "I" }` - mesmos dois motivos legais dos recibos; fora deles a ND não se anula. Sai dos totais e, se já estava validada, a anulação é comunicada à AGT (`agtCancelStatus`). SDK: `vero.debitNotes.cancel(orgId, id, { reason: 'I' })` (@veroao/node ≥ 1.3) |
| Proforma | `/v1/organisations/{orgId}/proformas` | Não fiscal — usa `.../{id}/convert` para virar factura real |
| Guia de Remessa | `/v1/organisations/{orgId}/invoices/{invoiceId}/delivery-note` | |

Todos devolvem PDF em `.../{id}/pdf`, e há uma versão pública sem autenticação em `/v1/public/{tipo}/{id}/pdf` (para enviar por email/link directo).

## Webhooks

```bash
curl -X POST https://api.vero.ao/v1/organisations/{orgId}/webhooks \
  -H "Authorization: Bearer sk_..." -H "Content-Type: application/json" \
  -d '{"url":"https://teu-site.com/webhook","events":["invoice.issued","invoice.cancelled"]}'
```

Eventos disponíveis: `invoice.issued`, `invoice.cancelled`, `proforma.converted`, `proforma.cancelled`.

Cada entrega vem com `X-Vero-Signature: sha256=...` (HMAC do body com o `secret` devolvido na criação — guarda-o, só é mostrado uma vez). Verifica sempre a assinatura antes de confiar no payload. Reentrega automática com backoff (1m, 5m, 30m, 2h, 8h) até 6 tentativas se o teu endpoint não responder `2xx`.

## Erros comuns

| Erro | Significado |
|---|---|
| `401 unauthorized` | Chave inválida, revogada, ou em falta no header `Authorization` |
| `403 forbidden` | Chave `pk_` a tentar aceder a rota que não é `/v1/widget/*` (anular facturas/recibos/notas de débito exige chave secreta, no servidor) |
| `422 agt_keys_not_configured` | Organização sem chave AGT própria — não emite fora do sandbox |
| `400 insufficient_stock` | Produto com `trackStock: true` sem quantidade suficiente |
| `409 receipt_already_exists` | Já existe um recibo activo para esta factura (anule-o primeiro se foi emitido por engano) |
| `400 invalid_reason` | Anulação de recibo ou nota de débito com motivo diferente de `N`/`I` |
| `409 already_cancelled` | O recibo ou a nota de débito já estava anulado |
| `409 agt_pending` | A AGT ainda está a validar o documento - tente anular daqui a alguns minutos |
| `422 tax_rate_incompatible_with_regime` | Linha com `taxRate` ≠ 0 numa organização em regime `isento`/`simplificado` |
| `422 tax_exemption_code_required` | `taxRate` resolveu para `0` mas falta `taxExemptionCode` (e não é Consumidor Final) |
| `422 invalid_tax_rate` | `taxRate` enviado não é `0`, `5` nem `14` |

## Onde ver mais

Documentação completa e interactiva: `https://vero.ao/docs`. Se precisares de um endpoint que não está aqui resumido, confirma lá antes de adivinhar o formato do payload. Mudanças que afectam código já escrito (novos campos, correcções de comportamento) ficam registadas em `https://vero.ao/docs/changelog` — os SDKs seguem semver, mas vale sempre confirmar aí antes de assumir um formato de payload que já não é actual.
