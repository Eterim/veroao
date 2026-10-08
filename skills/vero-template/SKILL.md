---
name: vero-template
description: Criar, verificar e publicar templates de facturas angolanas (FT, FR, NC, ND, RC, pró-forma) em React e Tailwind CSS com @veroao/invoice, e usá-los no Vero ou gerar o PDF no próprio projecto. Usa isto sempre que o utilizador pedir para "desenhar/criar um template de factura", "mudar o aspecto das facturas no Vero", "gerar o PDF de uma factura", "publicar um template no Vero Template" ou trabalhar com @veroao/invoice.
---

# Vero Template - templates de facturas em React + Tailwind

`@veroao/invoice` transforma um componente React com classes Tailwind num PDF de factura angolana, com as regras da AGT incluídas. Um template só descreve o **aspecto**: tudo o que é fiscal (número, data, ATCUD, assinatura, QR, NIF, totais, menções legais) vem sempre dos dados do documento.

- Site e documentação: https://template.vero.ao/
- Código e galeria: https://github.com/Eterim/vero-template (pasta `templates/`)
- Pacote: https://www.npmjs.com/package/@veroao/invoice (0.x - a API pode mudar até à 1.0)

## Começar

```bash
npx @veroao/invoice init minhas-facturas   # projecto novo com templates/invoice.tsx
cd minhas-facturas && npm install
npx @veroao/invoice dev                     # pré-visualização ao vivo em http://localhost:3200
npx @veroao/invoice check                   # o que o Vero e a galeria verificam (exit 1 se falhar)
```

Num projecto que já existe: `npm install @veroao/invoice react`.

O `dev` procura os templates em `templates/` (senão, na pasta actual): cada `.tsx` com `export default`, ou `<pasta>/<nome>/template.tsx`. Mostra os cinco tipos (FT, FR, NC, ND, RC), os erros com o componente e o motivo, e o botão **JSON para o Vero**.

## Um template

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

Regras do componente:
- Tem de devolver um `<Document>` (pode estar dentro de `<Tailwind>`).
- Componentes próprios, `.map()` e condições funcionam. **Hooks não** (`useState`, `useEffect`…): um template não tem estado.
- Só importa de `react` e `@veroao/invoice`. Nada de `fetch`, `process`, `eval`, `require`, timers ou `import()`.

## Componentes

| Componente | Para quê | Props úteis |
|---|---|---|
| `Tailwind` | Cores e letras do template (como o `tailwind.config`) | `config={{ theme: { extend: { colors, fontFamily } } }}` |
| `Document` | A página A4 (raiz) | `bg-*` fundo, `px-*` margens, `pt-*` margem de cima |
| `Header` | Faixa a toda a largura no topo da 1.ª página | |
| `Footer` | Faixa no fundo de todas as páginas | `h-*` altura |
| `CornerDecoration` | Faixas decorativas no canto superior direito | `color` |
| `Row` / `Column` | Lado a lado / empilhados | `gap-*`, `items-*`, `justify-*` |
| `Text` | Texto livre (com variáveis, ver abaixo) | |
| `Spacer` / `Hr` | Espaço vertical / linha | `h-*` / `border-t-*` |
| `Logo` | Logótipo da empresa que emite (o template nunca traz um) | `fallback="name" \| "none"` |
| `DocumentTitle` | "Factura", "Nota de Crédito"… | `withCode` |
| `DocumentNumber` / `DocumentDate` / `Atcud` | Número, data, ATCUD | `prefix`, `withTime` |
| `StatusBadge` | PAGO / POR PAGAR / ANULADO | |
| `Issuer` / `Customer` | Empresa e cliente (nome, NIF, morada) | `label`, `taxIdLabel`, `labelClassName`, `nameClassName` |
| `Payment` | Forma de pagamento | `label` |
| `Items` | Tabela das linhas | `columns`, `headerClassName`, `rowClassName` (aceita `even:`), `gridClassName`, `showCurrency` |
| `Totals` | Totais e IVA por taxa | `byRate`, `totalLabel`, `totalClassName`, `totalRowClassName`, `rowClassName` |
| `Notes` / `AmountInWords` | Observações / total por extenso | `label`, `inline` |
| `BankAccounts` | Contas da empresa | `layout="list" \| "table"` |
| `LegalNotes` | ATCUD, isenções, texto legal | `parts` |
| `PageNumber` | "PÁGINA 1 / 2" | |

Colunas de `Items`: `description`, `details`, `quantity`, `unitPrice`, `lineDiscount`, `taxRate`, `taxAmount`, `lineTotal` - `width` é a proporção da coluna - ex.: `columns={[{ field: "description", label: "Artigo", width: 3 }, { field: "quantity", label: "Qt", width: 0.8 }, { field: "lineTotal", label: "Total", width: 1 }]}`.

## Tailwind suportado

Medidas como na web (1 px = 0,75 pt; a A4 tem 794 px de largura).

- Espaço: `p-* px-* py-* pt-* pb-* pl-* pr-* m*-*`, `gap-*`, `ml-auto`
- Texto: `text-xs…6xl`, `text-[11px]`, `font-normal/semibold/bold`, `font-display`, `uppercase`, `tracking-*`, `leading-*`, `text-left/center/right`
- Cor: paleta do Tailwind, cores do `config`, `text-[#hex]`, `bg-[#hex]`, `border-[#hex]`
- Bordas: `border`, `border-2`, `border-[0.5px]`, `border-t/b/l/r-*`, `rounded-*`
- Disposição: `flex-1`, `flex-[2]`, `items-*`, `justify-*`, `w-1/2`, `w-full`, `w-[200px]`, `h-*`
- Variante: só `even:` (linhas alternadas da tabela)
- Também `style={{ fontSize: 9 }}` (em pt)

**Não existem num PDF** e dão erro: `hover:`, `md:`/`lg:`, `dark:`, sombras, degradês, transformações, `grid`, posicionamento absoluto. Não os uses.

## O que um template NUNCA pode ter

A biblioteca, o Vero e a CI da galeria recusam (`checkTemplate`):
- IBAN, números de conta, telefones, NIF ou outros números longos escritos à mão → usa `<BankAccounts />`, `<Issuer />`.
- Ligações e e-mails escritos à mão → usa as variáveis.
- Frases que imitam menções fiscais ("Processado por programa válido", ATCUD, AGT, códigos de isenção, "Original", "Pago") → vêm de `<LegalNotes />`, `<Atcud />`, `<StatusBadge />`.
- Caracteres invisíveis.

Variáveis permitidas em `Text`, `prefix` e `label`: `{{org.name}}`, `{{org.website}}`, `{{org.email}}`, `{{org.phone}}`, `{{customer.name}}`, `{{document.reference}}`, `{{document.number}}`, `{{document.title}}`, `{{document.type}}`.

```tsx
<Text className="text-[9px] text-zinc-500">{"{{org.name}} · {{org.website}} · {{org.email}}"}</Text>
```

## O que a biblioteca garante sozinha (não reimplementes)

- QR AGT no canto inferior direito da última página (a pró-forma não leva).
- Rodapé AGT ("XXXX-Processado por programa válido nº …" + número do documento) em todas as páginas.
- Retenção na fonte e "Valor líquido a pagar", "IVA - Regime Simplificado", "Não sujeito" nas linhas M02, marca de água nos anulados, aviso da pró-forma.
- Elementos obrigatórios em falta (título, número, data, partes, linhas, totais, menções legais) são acrescentados e aparecem nos `warnings` - corrige o template para não haver avisos.
- Contraste mínimo 4,5 e nunca menos de 7 pt no texto fiscal.

## Gerar o PDF no teu projecto (sem o Vero)

```ts
import { render, sampleDocument } from "@veroao/invoice"
import MyInvoice from "./templates/invoice"

const { pdf, warnings } = await render(<MyInvoice />, sampleDocument("FT"))  // pdf: Uint8Array
```

O segundo argumento é um `DocumentData` com os dados reais: `documentType` (`FT|FR|NC|ND|RC|PF`), `number`, `issuedAt`, `atcud`, `hashChars` (4 caracteres da assinatura), `certificationNumber` (o número do **teu** programa certificado), `qrUrl`, `org`, `customer`, `lines`, `totals` (valores em cêntimos), `currency`, e opcionalmente `status`, `payment`, `withholding`, `notes`, `amountInWords`. Para experimentar usa `sampleDocument(tipo)`.

## Usar no Vero

1. Obter o JSON: botão **JSON para o Vero** do `dev`, ou `compile(<MyInvoice />)` (devolve o template; o JSON de importação é `{ format: "vero-template", schemaVersion: 2, id, version, name, author, template }`).
2. No Vero (plano Pro): Definições → Template de facturas → **Importar template**, colar o JSON, ver a pré-visualização com os dados da empresa, **Importar e usar**.
3. Templates da galeria: o botão **Abrir no Vero** na página do template abre a importação já preenchida.

O Vero só recebe JSON, nunca executa código. Cada documento fica preso à versão do template com que foi emitido; os já emitidos não mudam.

## Publicar na galeria (pull request)

1. Fork de `Eterim/vero-template`, criar `templates/<nome>/` (minúsculas, números, hífens) com `meta.json` e `template.tsx` - copiar um template existente como ponto de partida.
2. `meta.json`: `slug`, `name`, `description`, `author: { name: "<github>" }`, `collection: "comunidade"`, `version: 1` (subir sempre que o template mudar), `license: "MIT"`, `docTypes`, `tags`.
3. Gerar `template.json` e as pré-visualizações (nunca editar à mão) e verificar:
   ```bash
   npm install
   npm run templates -w packages/invoice -- <nome>
   npm run check:templates -w packages/invoice -- <nome>
   ```
4. Abrir o PR. Um PR de fora só pode mexer numa pasta `templates/<nome>/` própria. A CI repete as verificações e um maintainer revê antes do merge.

## Erros comuns

| Mensagem | Causa | Correcção |
|---|---|---|
| `hooks não são suportados` | `useState`/`useEffect` no template | Tirar o hook; calcular com props/constantes |
| `o template tem de devolver um <Document>` | Raiz errada | Envolver tudo em `<Document>` (dentro de `<Tailwind>` se houver config) |
| `variável desconhecida {{…}}` | Variável fora da lista | Usar só as variáveis permitidas |
| `números de conta, telefone, NIF…` | Número escrito à mão | `<BankAccounts />` / `<Issuer />` |
| `ligações não podem estar no template` | URL no texto | `{{org.website}}` |
| `elemento obrigatório que não estava no template` | Falta um componente fiscal | Acrescentar o componente indicado |
| `template.json não corresponde ao template.tsx` | JSON editado à mão ou desactualizado | `npm run templates -w packages/invoice -- <nome>` |
| Classe com `hover:`/`md:`/`shadow`/`grid` recusada | Não existe num PDF | Usar `flex`/`Row`/`Column` e cores sólidas |

Para integrar a API de facturação do Vero (emitir facturas, clientes, webhooks), usa a skill `vero`.
