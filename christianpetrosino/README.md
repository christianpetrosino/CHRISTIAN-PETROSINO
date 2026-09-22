# Olist BI Report — Esercizio

Progettazione di un report di Business Intelligence sul dataset pubblico [Olist Store](https://olist.com/pt-br/), e-commerce brasiliano.

## Obiettivo

Analizzare, per il periodo 2016–2018:
1. Andamento degli ordini nel tempo per stato ordine
2. Andamento dei ricavi nel tempo per stato ordine
3. Distribuzione del rating (review score)

con confronto anno su anno, variazione % mese su mese e filtro sullo status dell'ordine.

## Contenuto del repository

- [`docs/design_report_olist.md`](docs/design_report_olist.md) — documento di progettazione: riduzione del dataset, star schema, dimensione calendario, misure DAX, layout e UX del report.
- [`mockup/mockup.html`](mockup/mockup.html) — mockup interattivo (HTML/SVG, nessuna dipendenza esterna) che simula la navigazione tra le 3 pagine del report con filtri, KPI e grafici di confronto anno su anno.

## Modello dati — sintesi

Star schema con due fact table a grana diversa (item e ordine/review), unite da una dimensione ponte `DIM_ORDER` per garantire coerenza del filtro sullo status ordine su entrambe:

- `FACT_SALES` (grana `order_item_id`): prezzo, spedizione, ricavo
- `FACT_REVIEWS` (grana ordine/review): review score
- `DIM_ORDER` (bridge): order_id, order_status, order_date, customer_id
- `DIM_CALENDAR`, `DIM_CUSTOMER` (customer_state), `DIM_PRODUCT` (categoria)

Dettagli completi e codice DAX nel documento in `docs/`.

## Come usare il mockup

Apri `mockup/mockup.html` in un browser: puoi passare tra le pagine **Ordini / Ricavi / Rating** dal menu in alto e provare i filtri (anno, status ordine, stato geografico).

## Autore

Christian Petrosino
