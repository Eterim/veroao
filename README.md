# Eterim Skills

Skills for AI coding agents (Claude Code, Cursor, Codex, …) to work with [Vero](https://vero.ao) - certified electronic invoicing for Angola (AGT).

| Skill | What it does |
|---|---|
| [`vero`](skills/vero/SKILL.md) | Integrate with the Vero API: customers, products, invoices, credit/debit notes, receipts, pro-formas, webhooks, `@veroao/*` SDKs. |
| [`vero-template`](skills/vero-template/SKILL.md) | Design invoice templates in React + Tailwind with [`@veroao/invoice`](https://github.com/Eterim/vero-template), check them against AGT rules, render PDFs, import them into Vero or publish them to the gallery. |

## Install

```bash
npx skills add Eterim/veroao                        # both
npx skills add Eterim/veroao --skill vero-template  # just one
```

Listed on [skills.sh](https://skills.sh/eterim/veroao).

## License

MIT
