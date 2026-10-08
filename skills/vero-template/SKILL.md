---
name: vero-template
description: Create, check and publish Angolan invoice templates (FT, FR, NC, ND, RC, pro-forma) in React and Tailwind CSS with @veroao/invoice, then use them in Vero or render the PDF in your own project. Use this whenever the user asks to "design/create an invoice template", "change how invoices look in Vero", "generate an invoice PDF", "publish a template to Vero Template" (or the Portuguese "criar um template de factura"), or works with @veroao/invoice.
---

# Vero Template - invoice templates in React + Tailwind

`@veroao/invoice` turns a React component styled with Tailwind classes into an Angolan invoice PDF, with the AGT (tax authority) rules built in. A template only describes the **look**: everything fiscal (number, date, ATCUD, signature, QR code, tax IDs, totals, legal notes) always comes from the document data.

- Website and docs (Portuguese): https://template.vero.ao/
- Code and gallery: https://github.com/Eterim/vero-template (`templates/` folder)
- Package: https://www.npmjs.com/package/@veroao/invoice (0.x - the API may change until 1.0)

Text printed on the PDF (labels, `Text` content) should be in Portuguese - these are Angolan fiscal documents. Library error messages are in Portuguese too; the meaning of each is in the table at the end.

## Getting started

```bash
npx @veroao/invoice init my-invoices   # new project with templates/invoice.tsx
cd my-invoices && npm install
npx @veroao/invoice dev                 # live preview at http://localhost:3200
npx @veroao/invoice check               # the same checks Vero and the gallery run (exit 1 on failure)
```

In an existing project: `npm install @veroao/invoice react`.

`dev` looks for templates in `templates/` (otherwise the current folder): every `.tsx` with a default export, or `<folder>/<name>/template.tsx`. It renders all five types (FT, FR, NC, ND, RC), shows errors with the component and the reason, and has a **JSON para o Vero** button.

## A template

```tsx
import { Tailwind, Document, Row, Logo, DocumentTitle, DocumentNumber,
  DocumentDate, Customer, Issuer, Items, Totals, LegalNotes } from "@veroao/invoice"

export default function MyInvoice() {
  return (
    <Tailwind config={{ theme: { extend: { colors: { brand: "#0E4C63" } } } }}>
      <Document className="bg-white px-12 pt-10 text-[11px]">
        <Row className="items-start justify-between">
          <Logo className="h-12" />
          <DocumentTitle className="text-3xl font-bold uppercase text-brand" />
        </Row>
        <Row className="mt-8 gap-6">
          <Customer className="flex-1 border border-zinc-200 p-3" labelClassName="font-bold text-brand" />
          <Issuer className="flex-1" />
        </Row>
        <Row className="mt-6 justify-between">
          <DocumentNumber className="font-bold" />
          <DocumentDate />
        </Row>
        <Items className="mt-3" rowClassName="border-b border-zinc-200 even:bg-zinc-50" />
        <Totals className="ml-auto mt-4 w-1/2" totalClassName="text-xl font-bold text-brand" />
        <LegalNotes className="mt-6 text-[9px] text-zinc-500" />
      </Document>
    </Tailwind>
  )
}
```

Component rules:
- It must return a `<Document>` (optionally inside `<Tailwind>`).
- Your own components, `.map()` and conditionals work. **Hooks don't** (`useState`, `useEffect`…): a template has no state.
- Import only from `react` and `@veroao/invoice`. No `fetch`, `process`, `eval`, `require`, timers or `import()`.

## Components

| Component | Purpose | Useful props |
|---|---|---|
| `Tailwind` | Template colours and fonts (like `tailwind.config`) | `config={{ theme: { extend: { colors, fontFamily } } }}` |
| `Document` | The A4 page (root) | `bg-*` background, `px-*` side margins, `pt-*` top margin |
| `Header` | Full-width band at the top of page 1 | |
| `Footer` | Band at the bottom of every page | `h-*` height |
| `CornerDecoration` | Decorative stripes in the top-right corner | `color` |
| `Row` / `Column` | Side by side / stacked | `gap-*`, `items-*`, `justify-*` |
| `Text` | Free text (with variables, see below) | |
| `Spacer` / `Hr` | Vertical space / rule | `h-*` / `border-t-*` |
| `Logo` | Issuer's logo (a template never ships its own) | `fallback="name" \| "none"` |
| `DocumentTitle` | "Factura", "Nota de Crédito"… | `withCode` |
| `DocumentNumber` / `DocumentDate` / `Atcud` | Number, date, ATCUD | `prefix`, `withTime` |
| `StatusBadge` | PAGO / POR PAGAR / ANULADO | |
| `Issuer` / `Customer` | Issuer and customer (name, NIF, address) | `label`, `taxIdLabel`, `labelClassName`, `nameClassName` |
| `Payment` | Payment method | `label` |
| `Items` | Line items table | `columns`, `headerClassName`, `rowClassName` (supports `even:`), `gridClassName`, `showCurrency` |
| `Totals` | Totals and VAT by rate | `byRate`, `totalLabel`, `totalClassName`, `totalRowClassName`, `rowClassName` |
| `Notes` / `AmountInWords` | Notes / total in words | `label`, `inline` |
| `BankAccounts` | Issuer's bank accounts | `layout="list" \| "table"` |
| `LegalNotes` | ATCUD, exemptions, legal text | `parts` |
| `PageNumber` | "PÁGINA 1 / 2" | |

`Items` columns: `description`, `details`, `quantity`, `unitPrice`, `lineDiscount`, `taxRate`, `taxAmount`, `lineTotal`. `width` is a relative proportion - e.g. `columns={[{ field: "description", label: "Artigo", width: 3 }, { field: "quantity", label: "Qt", width: 0.8 }, { field: "lineTotal", label: "Total", width: 1 }]}`.

## Supported Tailwind

Sizes as on the web (1 px = 0.75 pt; A4 is 794 px wide).

- Spacing: `p-* px-* py-* pt-* pb-* pl-* pr-* m*-*`, `gap-*`, `ml-auto`
- Text: `text-xs…6xl`, `text-[11px]`, `font-normal/semibold/bold`, `font-display`, `uppercase`, `tracking-*`, `leading-*`, `text-left/center/right`
- Colour: Tailwind palette, `config` colours, `text-[#hex]`, `bg-[#hex]`, `border-[#hex]`
- Borders: `border`, `border-2`, `border-[0.5px]`, `border-t/b/l/r-*`, `rounded-*`
- Layout: `flex-1`, `flex-[2]`, `items-*`, `justify-*`, `w-1/2`, `w-full`, `w-[200px]`, `h-*`
- Variant: only `even:` (alternating table rows)
- Also `style={{ fontSize: 9 }}` (in pt), like react-pdf

**Not available in a PDF** and rejected with an error: `hover:`, `md:`/`lg:`, `dark:`, shadows, gradients, transforms, `grid`, absolute positioning. Don't use them.

## What a template must NEVER contain

The library, Vero and the gallery CI reject (`checkTemplate`):
- Hand-written IBANs, account numbers, phone numbers, NIFs or other long numbers → use `<BankAccounts />`, `<Issuer />`.
- Hand-written links and e-mails → use the variables.
- Phrases imitating fiscal notes ("Processado por programa válido", ATCUD, AGT, exemption codes, "Original", "Pago") → they come from `<LegalNotes />`, `<Atcud />`, `<StatusBadge />`.
- Invisible characters.

Variables allowed in `Text`, `prefix` and `label`: `{{org.name}}`, `{{org.website}}`, `{{org.email}}`, `{{org.phone}}`, `{{customer.name}}`, `{{document.reference}}`, `{{document.number}}`, `{{document.title}}`, `{{document.type}}`.

```tsx
<Text className="text-[9px] text-zinc-500">{"{{org.name}} · {{org.website}} · {{org.email}}"}</Text>
```

## What the library guarantees on its own (don't reimplement)

- AGT QR code in the bottom-right corner of the last page (pro-formas don't get one).
- AGT footer ("XXXX-Processado por programa válido nº …" + document number) on every page.
- Withholding tax and "Valor líquido a pagar", "IVA - Regime Simplificado", "Não sujeito" on M02 lines, watermark on cancelled documents, pro-forma notice.
- Missing mandatory elements (title, number, date, parties, lines, totals, legal notes) are added and reported in `warnings` - fix the template so there are none.
- Minimum contrast of 4.5 and never below 7 pt for fiscal text.

## Rendering the PDF in your own project (without Vero)

```ts
import { render, sampleDocument } from "@veroao/invoice"
import MyInvoice from "./templates/invoice"

const { pdf, warnings } = await render(<MyInvoice />, sampleDocument("FT"))  // pdf: Uint8Array
```

The second argument is a `DocumentData` with the real data: `documentType` (`FT|FR|NC|ND|RC|PF`), `number`, `issuedAt`, `atcud`, `hashChars` (4 signature characters), `certificationNumber` (**your** certified software number), `qrUrl`, `org`, `customer`, `lines`, `totals` (amounts in cents), `currency`, and optionally `status`, `payment`, `withholding`, `notes`, `amountInWords`. To experiment, use `sampleDocument(type)`.

## Using it in Vero

1. Get the JSON: the **JSON para o Vero** button in `dev`, or `compile(<MyInvoice />)` (returns the template; the import JSON is `{ format: "vero-template", schemaVersion: 2, id, version, name, author, template }`).
2. In Vero (Pro plan): Definições → Template de facturas → **Importar template**, paste the JSON, review the preview with the company's data, **Importar e usar**.
3. Gallery templates: the **Abrir no Vero** button on the template page opens the import already filled in.

Vero only receives JSON and never runs code. Each document is pinned to the template version it was issued with; documents already issued never change.

## Publishing to the gallery (pull request)

1. Fork `Eterim/vero-template` and create `templates/<name>/` (lowercase letters, digits, hyphens) with `meta.json` and `template.tsx` - copy an existing template as a starting point.
2. `meta.json`: `slug`, `name`, `description`, `author: { name: "<github-user>" }`, `collection: "comunidade"`, `version: 1` (bump on every change), `license: "MIT"`, `docTypes`, `tags`.
3. Generate `template.json` and the previews (never edit them by hand) and check:
   ```bash
   npm install
   npm run templates -w packages/invoice -- <name>
   npm run check:templates -w packages/invoice -- <name>
   ```
4. Open the PR. An external PR may only touch its own `templates/<name>/` folder. CI repeats the checks and a maintainer reviews before merging.

## Common errors (messages are in Portuguese)

| Message | Cause | Fix |
|---|---|---|
| `hooks não são suportados` | `useState`/`useEffect` in the template | Remove the hook; compute from props/constants |
| `o template tem de devolver um <Document>` | Wrong root | Wrap everything in `<Document>` (inside `<Tailwind>` if there is a config) |
| `variável desconhecida {{…}}` | Variable not in the list | Use only the allowed variables |
| `números de conta, telefone, NIF…` | Hand-written number | `<BankAccounts />` / `<Issuer />` |
| `ligações não podem estar no template` | URL in text | `{{org.website}}` |
| `elemento obrigatório que não estava no template` | A fiscal component is missing | Add the component named in the message |
| `template.json não corresponde ao template.tsx` | JSON edited by hand or stale | `npm run templates -w packages/invoice -- <name>` |
| Class with `hover:`/`md:`/`shadow`/`grid` rejected | Doesn't exist in a PDF | Use `flex`/`Row`/`Column` and solid colours |

To integrate Vero's invoicing API (issuing invoices, customers, webhooks), use the `vero` skill.
