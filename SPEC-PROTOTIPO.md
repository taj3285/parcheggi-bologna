# SPEC PROTOTIPO — Trova parcheggio Bologna (locale, solo frontend, esecuzione one-shot)

> Specifica **operativa** di un prototipo da guardare, non del prodotto finale.
> Va eseguita **tutta d'un fiato, senza fermarsi a chiedere conferme**. Vedi la sezione 11.

---

## 1. Obiettivo

Una web app che gira su `localhost` e risponde a questa domanda:

> *"Arrivo in Piazza Maggiore tra 15 minuti, resto 3 ore, ho un'auto elettrica, non voglio camminare più di 10 minuti, budget massimo 12 €. Dove parcheggio?"*

La risposta è una lista ordinata di parcheggi, ciascuno con la **probabilità di trovare posto all'orario di arrivo** (non adesso), il costo stimato, il tempo a piedi, e l'indicazione onesta di quanto quel dato sia affidabile.

## 2. Vincoli — cosa NON costruire

Servono alla velocità. Rispettali anche se sembrano limitanti.

- **Nessun backend.** Niente server Node, niente Express/Fastify, niente API proprietarie.
- **Nessun database.** Niente Postgres, niente Docker, niente migrazioni.
- **Nessun modello ML.** La previsione è una curva oraria in un file di configurazione.
- **Nessun servizio a pagamento e nessuna API key.** Se una soluzione richiede una chiave, scegline un'altra.
- **Nessun deploy.** Il prototipo vive solo su `localhost`.
- **Nessuna autenticazione, nessun account, nessun pagamento, nessuna prenotazione.**
- **Nessuna suite di test completa.** Solo i test della sezione 10.

## 3. Stack tecnico esatto

- **React + TypeScript**, scaffolding con **Vite**
- **npm** come gestore pacchetti (non pnpm, non yarn: riduce i punti di rottura)
- **MapLibre GL JS** per la mappa
- Stato in React (`useState` / `useReducer`), preferenze in `localStorage`. Niente Redux.
- Stile: CSS moduli o Tailwind, a tua scelta. Non è il punto del prototipo.

**Prima cosa:** esegui `node -v` e scegli una versione di Vite compatibile. Se Node manca o è più vecchio di 20, **fermati e segnalalo**: è l'unico blocco che non puoi aggirare da solo.

### Mappa base — usa questa, è senza chiave

```
https://tiles.openfreemap.org/styles/liberty
```

OpenFreeMap è keyless e non richiede account. **Non usare i basemap CARTO** (`basemaps.cartocdn.com`): richiedono una API key e servono tile con watermark alle richieste anonime. Se OpenFreeMap non risponde, ripiega su tile raster standard di OpenStreetMap con la dovuta attribuzione.

## 4. Struttura file

```
src/
  App.tsx
  types.ts
  config/
    scoring.ts              # pesi del ranking, commentati
    occupancy-curves.ts     # curve orarie di occupazione tipica
    destinations.ts         # destinazioni predefinite di Bologna
  providers/
    types.ts                # interfaccia ParkingDataProvider
    MockProvider.ts
    BolognaOpenDataProvider.ts
  core/
    scoring.ts              # motore di ranking (logica pura, testabile)
    pricing.ts              # calcolo del costo della sosta
    geo.ts                  # distanze e tempi stimati
  components/
    Map.tsx
    NeedsPanel.tsx
    ResultList.tsx
    ResultCard.tsx
    FacilityDetail.tsx
```

La logica in `core/` deve essere **pura**: nessuna chiamata di rete, nessun riferimento a React.

## 5. Modello dati

```ts
type DataLevel = 'MEASURED' | 'ESTIMATED';

interface ParkingFacility {
  id: string;
  name: string;
  type: 'struttura' | 'raso' | 'strada' | 'scambiatore';
  lat: number;
  lng: number;
  totalSpaces: number;
  hasEvCharging: boolean;
  accessibleSpaces: number;
  maxHeightMeters: number | null;   // null = nessun limite
  covered: boolean;
  guarded: boolean;
  pricePerHour: number;             // semplificazione: tariffa unica
  dailyCap: number | null;
  freeForEv: boolean;
  freeForDisabled: boolean;
  insideZtl: boolean;
}

interface Availability {
  facilityId: string;
  freeSpaces: number;
  timestamp: string;                // ISO
  dataLevel: DataLevel;
  confidence: number;               // 0–1
}

interface UserNeeds {
  destination: { lat: number; lng: number; label: string };
  arrivalInMinutes: number;         // default 15
  durationHours: number;            // default 2
  vehicle: 'auto' | 'moto' | 'van';
  vehicleHeightMeters: number | null;
  isElectric: boolean;
  needsCharging: boolean;
  hasDisabledPermit: boolean;
  maxBudgetEur: number | null;
  maxWalkMinutes: number;
  preferCovered: boolean;
}
```

## 6. Dati

### 6.1 Interfaccia comune

```ts
interface ParkingDataProvider {
  readonly id: string;
  getFacilities(): Promise<ParkingFacility[]>;
  getAvailability(): Promise<Availability[]>;
}
```

Tutta l'app parla solo con questa interfaccia. Cambiare fonte dati deve significare cambiare una riga.

### 6.2 MockProvider — costruiscilo per primo

Genera dati sintetici ma **plausibili** per Bologna: 25–35 parcheggi con coordinate reali distribuite tra centro storico, zona stazione, Fiera e cintura esterna. Mescola le tipologie: strutture multipiano, parcheggi a raso, un paio di scambiatori periferici, diverse aree di sosta su strada.

La disponibilità varia con l'ora seguendo le curve in `config/occupancy-curves.ts`, con un po' di rumore casuale. I parcheggi di tipo `strada` hanno sempre `dataLevel: 'ESTIMATED'` e confidenza più bassa: nella realtà la sosta su strada non è sensorizzata.

Con il solo MockProvider l'app deve essere già completamente utilizzabile.

### 6.3 BolognaOpenDataProvider

Portale: `opendata.comune.bologna.it`, piattaforma Opendatasoft.

```
https://opendata.comune.bologna.it/api/explore/v2.1/catalog/datasets/{dataset_id}/records?limit=100
```

Dataset da cui partire: `disponibilita-parcheggi-vigente` (posti liberi, ultima rilevazione), `parcheggi` (anagrafica delle strutture).

**IMPORTANTE.** Interroga davvero l'endpoint, guarda lo schema che torna, e scrivi il parser sui **nomi di campo reali**. Non inventare nomi di campo. Non dare per scontato che gli ID dei dataset siano ancora questi.

**Se questa parte fallisce, non fermarti**: lascia il provider implementato al meglio possibile, fai partire l'app sul MockProvider, e scrivi nel README esattamente cosa non ha funzionato e cosa hai osservato. Un prototipo che gira su dati finti vale infinitamente più di un prototipo che non parte.

Se il browser blocca la chiamata per CORS, configura il proxy di sviluppo di Vite in `vite.config.ts`.

### 6.4 Selettore visibile in interfaccia

In alto un interruttore **Mock / Dati reali**. Se il provider reale fallisce, l'app ricade sul mock mostrando un avviso, senza rompersi.

## 7. Motore di raccomandazione

### 7.1 Esclusioni (prima del punteggio)

Un candidato viene **escluso**, con motivo consultabile, se: l'altezza del veicolo supera `maxHeightMeters`; serve la ricarica e non c'è; serve uno stallo accessibile e `accessibleSpaces` è zero; il parcheggio è dentro la ZTL e il veicolo non ha titolo per entrare. Le esclusioni non sono penalità: sono squalifiche.

### 7.2 Punteggio

```
score = w_p · probabilitàPosto
      + w_c · utilitàCosto
      + w_w · utilitàCamminata
      + w_f · fiduciaDato
```

I pesi stanno in `config/scoring.ts`, commentati, e si **adattano alle esigenze dichiarate**: se l'utente ha indicato un budget, `w_c` cresce; se ha indicato una camminata massima stretta, cresce `w_w`. Non pesi fissi.

### 7.3 Probabilità di trovare posto all'arrivo

Non usare il valore attuale. Calcola:

```
occupazione_prevista = occupazione_attuale
                     + (curva[ora_arrivo] − curva[ora_attuale])
```

dove `curva` è la curva oraria tipica per quel tipo di parcheggio, in `config/occupancy-curves.ts`, con un commento che dichiara che è una stima grezza e non un modello addestrato. Converti in probabilità con una funzione monotona semplice, e restituisci sempre anche la confidenza.

### 7.4 Costo

`pricePerHour × durationHours`, con tetto giornaliero se presente, e gratuità per veicolo elettrico e contrassegno disabili. Nel prototipo la tariffa è un numero unico: fasce orarie e festivi sono una semplificazione ammessa, da annotare nel README.

### 7.5 Spiegazione

Ogni risultato espone una riga in italiano che spiega perché è lì:
*"6 minuti a piedi, storicamente libero al 70% a quest'ora, e la tua auto elettrica non paga."*

## 8. Semplificazioni ammesse — non cercare soluzioni migliori

| Cosa | Come farla |
|---|---|
| Distanza a piedi | Linea d'aria × 1,35, a 4,8 km/h |
| Tempo di guida | Linea d'aria × 1,5, a 25 km/h urbani |
| Isocroni pedonali | Cerchi sulla mappa, non poligoni reali |
| Ricerca destinazione | Lista fissa in `config/destinations.ts` con ~12 luoghi: Piazza Maggiore, Stazione Centrale, Fiera, Ospedale Sant'Orsola, via Zamboni, Aeroporto Marconi, Ospedale Maggiore, Piazza VIII Agosto, Autostazione, Stadio Dall'Ara, Villa Ghigi, MAST. La ricerca libera solo se avanza tempo, con Photon (`photon.komoot.io`, senza chiave) |
| Confini ZTL | Un poligono approssimato del centro storico, hardcoded, con un commento che dichiara l'approssimazione |
| Meteo, eventi, traffico | Ignorali |

## 9. Interfaccia

Schermata unica, mobile-first, che sta in un solo scroll:

1. **Barra in alto** — destinazione (menu a tendina), interruttore Mock/Reale, ora dell'ultimo aggiornamento.
2. **Mappa** (circa metà schermo) — marker colorati per disponibilità. Bordo **pieno** = dato misurato, **tratteggiato** = stimato. Cerchio dell'area raggiungibile a piedi attorno alla destinazione.
3. **Pannello esigenze** — compatto, richiudibile: ora di arrivo, durata, tipo veicolo, elettrico, contrassegno disabili, budget, camminata massima.
4. **Lista risultati** — le 5 migliori opzioni. Ogni scheda mostra, in quest'ordine di dominanza visiva: **probabilità**, **minuti a piedi**, poi costo, tipo di parcheggio, badge (elettrico / accessibile / coperto / ZTL), e infine la riga di spiegazione e la freschezza del dato.
5. **Dettaglio** — pannello che si apre al clic: tutti i campi, grafico della curva oraria, e un pulsante "Apri in Google Maps" che è solo un link.

### Regole di linguaggio, non negoziabili

- Mai scrivere "posto disponibile" o "posto garantito". Sempre probabilistico: "78% di probabilità".
- Ogni numero porta con sé provenienza ed età. Un dato di 40 minuti fa si deve **vedere** che è vecchio.
- Non affidarti solo a rosso/verde: usa anche forma e testo.
- Se nessun risultato rispetta i vincoli, non mostrare una lista vuota: proponi il rilassamento del vincolo più stringente — *"nessun posto entro 10 minuti a piedi sotto 12 €; allargando a 15 minuti ne trovi 3."*
- In fondo alla pagina: attribuzione dei dati (Comune di Bologna, OpenStreetMap) e disclaimer che i dati sono indicativi e non garantiscono la disponibilità.

## 10. Test — solo questi

Test unitari (Vitest) sulla sola logica in `core/`:

- costo con e senza tetto giornaliero, con e senza gratuità
- esclusioni: un van di 2,4 m non riceve mai un parcheggio con altezza massima 2,0 m
- il punteggio cambia coerentemente al variare dei pesi
- la probabilità all'arrivo differisce dalla disponibilità attuale quando le curve lo prevedono

Niente test end-to-end, niente test di accessibilità automatizzati, niente obiettivi di copertura.

## 11. Esecuzione one-shot

**Costruisci tutto in una sola sessione, senza chiedermi conferme intermedie.** Questo è l'ordine tecnico da seguire, non una serie di punti di controllo:

1. Scaffolding Vite + mappa MapLibre centrata su Bologna
2. Tipi, interfaccia provider, MockProvider, marker sulla mappa
3. Pannello esigenze + motore di scoring + lista risultati
4. Scheda di dettaglio, grafico, spiegazioni, stati vuoti
5. `BolognaOpenDataProvider` con dati reali e interruttore Mock/Reale
6. Test della sezione 10, README, `npm run dev` avviato

**Regole di autonomia:**

- **Fai un commit alla fine di ogni punto**, con messaggio convenzionale (`feat: mappa base`, `feat: motore di scoring`, …). Sono i miei punti di ritorno: se qualcosa non mi piace, torno lì con `git reset`.
- **Non chiedermi di scegliere.** Se incontri un'ambiguità, prendi la decisione più ragionevole, vai avanti, e annotala nel README sotto "Assunzioni".
- **Non fermarti per un fallimento recuperabile.** API irraggiungibile, CORS, un dataset cambiato: degrada sul mock, prosegui, documenta.
- **Fermati solo per un blocco reale:** Node assente o troppo vecchio, spazio disco esaurito, permessi negati. In quel caso dimmi esattamente cosa serve.
- **Alla fine**, avvia il server di sviluppo e dammi in tre righe: l'URL da aprire, cosa funziona, cosa è rimasto approssimato.

## 12. Fatto quando

- [ ] `npm install && npm run dev` funziona da repo pulita e stampa l'indirizzo locale.
- [ ] La mappa carica e mostra i parcheggi.
- [ ] Lo scenario della sezione 1 produce cinque risultati sensati e ordinati.
- [ ] Ogni risultato mostra probabilità, camminata, costo, provenienza e freschezza del dato.
- [ ] Un van di 2,4 metri non riceve mai un parcheggio troppo basso.
- [ ] L'interruttore Mock/Reale funziona, e con i dati reali irraggiungibili l'app avvisa invece di rompersi.
- [ ] Da nessuna parte è scritto che un posto è garantito.
- [ ] Il README spiega come avviare l'app, quali semplificazioni sono state fatte e quali assunzioni hai preso.
