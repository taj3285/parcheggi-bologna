# Progetto: app trova-parcheggio real-time (città pilota: Bologna)

App che stima **dove ci sarà posto all'orario di arrivo**, non dove c'è posto adesso.
Specifica completa: `docs/SPEC.md` — leggila prima di lavorare su scoring, provider dati o modello dati.

> Nota: `docs/SPEC.md` è volutamente **non** importato con la sintassi `@`. È lungo e
> caricarlo a ogni sessione sprecherebbe contesto. Leggilo su richiesta.

## Stack

- Frontend: React + TypeScript + Vite, PWA, MapLibre GL
- Backend: Node.js + Fastify, PostgreSQL + PostGIS, Redis
- Monorepo: `apps/web`, `apps/api`, `packages/core`, `packages/providers`

## Comandi

<!-- Aggiornali appena lo scaffolding esiste; ora sono segnaposto -->

```bash
pnpm install
docker compose up -d      # postgres + redis
pnpm dev                  # web + api
pnpm test                 # unit
pnpm test:e2e             # playwright
pnpm lint && pnpm typecheck
```

## Regole vincolanti

- **IMPORTANTE: non inventare mai nomi di campo, ID di dataset o endpoint delle API open data.**
  Interroga l'endpoint reale, ispeziona lo schema, poi scrivi il parser. Se non hai rete,
  lavora contro `MockProvider` e lascia un TODO esplicito.
- Mai committare `.env`, dump di dati o file in `data/raw/`.
- Ogni dato mostrato all'utente porta con sé livello (`MEASURED` / `ESTIMATED` / `REPORTED`)
  e timestamp. Mai presentare una stima come una misurazione.
- Nessun linguaggio che garantisce la disponibilità di un posto: sempre probabilistico.
- La logica di dominio non dipende dallo schema di una specifica API comunale:
  passa sempre dall'interfaccia `ParkingDataProvider`.
- Niente `any` in TypeScript.

## Ordine di costruzione

Modello dati e tipi condivisi → provider e ingestion → motore di scoring con i suoi test → API → UI.
**Non partire dalla UI.**

## Convenzioni

- Commit convenzionali (`feat:`, `fix:`, `chore:`, `docs:`)
- Un branch per feature, merge su `main` via PR
- I commenti spiegano il *perché*, non il *cosa*
