# Starting5

Build a "Starting 5+1" lineup from any category — NBA players, sitcom characters, politicians, inanimate objects, whatever — and put it up against other people's lineups. The community votes on who wins. Free, open source, ad-supported.

## Status

Pre-MVP. See [`docs/superpowers/specs/`](docs/superpowers/specs/) for the current design spec and roadmap.

## Stack

- [Next.js](https://nextjs.org/) (App Router) + TypeScript
- [Tailwind CSS](https://tailwindcss.com/) + [shadcn/ui](https://ui.shadcn.com/)
- [Supabase](https://supabase.com/) (Postgres, Auth, Storage)
- [Vercel](https://vercel.com/) (hosting)
- [Vitest](https://vitest.dev/) + [React Testing Library](https://testing-library.com/react) for unit tests
- [Playwright](https://playwright.dev/) for end-to-end tests

## Getting started

```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

## Development

This project follows test-driven development — see [CONTRIBUTING.md](CONTRIBUTING.md).

```bash
npm run lint        # ESLint
npm run typecheck   # TypeScript
npm run test         # Vitest unit tests
npm run test:e2e     # Playwright end-to-end tests
npm run build        # Production build
```

## License

[GPL-3.0](LICENSE)
