# Report BI Olist — Progettazione del modello e del report

## 1. Riduzione del volume dei dati

Delle tabelle originali si tengono solo le colonne necessarie alle analisi richieste. Tutto il resto (descrizioni prodotto, foto, timestamp intermedi di logistica non usati, testo libero delle review, ecc.) viene scartato in fase di query/ETL (Power Query).

| Tabella sorgente | Colonne da mantenere |
|---|---|
| `olist_orders_dataset` | `order_id`, `customer_id`, `order_status`, `order_purchase_timestamp` |
| `olist_order_items_dataset` | `order_id`, `order_item_id`, `product_id`, `price`, `freight_value` |
| `olist_products_dataset` | `product_id`, `product_category_name` |
| `olist_order_reviews_dataset` | `review_id`, `order_id`, `review_score` |
| `olist_customers_dataset` | `customer_id`, `customer_state` |

Riduzioni chiave:
- niente colonne di testo libero (titoli/commenti review, nomi prodotto lunghi);
- `customer_unique_id` non serve (l'analisi è geografica, non sul singolo cliente);
- dimensioni tempo derivate una sola volta nella Calendar, non ripetute nelle tabelle fatto;
- i prezzi/valori restano a grana di riga (item), l'aggregazione avviene solo a runtime nelle misure.

## 2. Star Schema

Si usano **due fact table** a grana diversa (best practice: non forzare tutto in un'unica fact, altrimenti la review — che è a livello ordine — verrebbe duplicata per ogni item e falserebbe le medie).

```
                         ┌───────────────┐
                         │  DIM_CALENDAR │
                         └───────┬───────┘
                                 │ (Date)
        ┌────────────────────────┼────────────────────────┐
        │                        │                         │
┌───────┴────────┐      ┌────────┴────────┐        ┌───────┴────────┐
│  DIM_CUSTOMER   │      │   FACT_SALES     │        │  FACT_REVIEWS  │
│ (customer_state)│──────┤ (grana: order_   │        │ (grana: order_ │
└───────┬────────┘      │  item)           │        │  id / review)  │
        │               │ order_id (FK)    │────────┤ order_id (FK)  │
        │               │ order_item_id    │  1:1   │ review_id      │
        │               │ product_id (FK)  │        │ review_score   │
        │               │ customer_id (FK) │        │ order_date(FK) │
        │               │ order_date (FK)  │        └────────────────┘
        │               │ order_status(FK) │
        │               │ price            │
        │               │ freight_value    │
        │               └────────┬─────────┘
        │                        │
┌───────┴────────┐      ┌────────┴────────┐
│  DIM_PRODUCT    │      │ DIM_ORDER_STATUS│
│ (category)      │      │ (order_status)  │
└─────────────────┘      └─────────────────┘
```

**FACT_SALES** (una riga = un `order_item_id`)
- Chiavi: `order_id`, `order_item_id`, `product_id`, `customer_id`, `order_date`, `order_status`
- Misure base: `price`, `freight_value`
- Colonna calcolata: `revenue = price + freight_value` (per riga, poi si somma)

**FACT_REVIEWS** (una riga = una review, grana ordine)
- Chiavi: `order_id`, `order_date`
- Attributo: `review_score`
- Relazione 1:1 (o 1:molti se un ordine ha più review) con `order_id`

**Dimensioni**
- `DIM_CALENDAR`: dimensione calendario standard, generata da `CALENDAR(MIN(data ordini), MAX(data ordini))`, marcata come **Date Table**. Relazione attiva verso `order_date` di entrambe le fact.
- `DIM_CUSTOMER`: solo `customer_id` → `customer_state` (+ eventualmente nome/regione mappata dallo stato per la mappa).
- `DIM_PRODUCT`: `product_id` → `product_category_name`.
- `DIM_ORDER_STATUS`: elenco distinto degli `order_status` (delivered, shipped, canceled, ecc.), usata come slicer trasversale sulle due fact tramite relazione da `FACT_SALES`. Per applicare lo stesso filtro anche a `FACT_REVIEWS`, si porta lo status anche su `FACT_REVIEWS` (via merge con `orders`) o si usa una tabella ponte `DIM_ORDER` (order_id → order_status → order_date) collegata a entrambe le fact: **scelta consigliata**, perché evita di duplicare lo status nelle due fact e garantisce coerenza del filtro.

Versione finale consigliata: aggiungere **DIM_ORDER** (bridge, grana `order_id`) con `order_id`, `order_status`, `order_date`, `customer_id`. `FACT_SALES` e `FACT_REVIEWS` si relazionano a `DIM_ORDER` (molti-a-uno), e `DIM_ORDER` si relaziona a `DIM_CALENDAR` e `DIM_CUSTOMER`. Così lo slicer sullo status filtra automaticamente entrambe le fact, e il conteggio ordini/ricavi/rating restano sempre coerenti sullo stesso perimetro di ordini.

## 3. Dimensione Calendario

Colonne minime:
`Date, Anno, Mese (numero), NomeMese, MeseAnno (es. "gen 2017"), Trimestre, GiornoSettimana`

```DAX
DIM_CALENDAR = 
ADDCOLUMNS(
    CALENDAR(DATE(2016,1,1), DATE(2018,12,31)),
    "Anno", YEAR([Date]),
    "NumMese", MONTH([Date]),
    "NomeMese", FORMAT([Date],"MMM"),
    "MeseAnno", FORMAT([Date],"MMM YYYY"),
    "Trimestre", "Q" & FORMAT([Date],"Q")
)
```
Marcare come **Mark as Date Table** su `Date`. Questo abilita le funzioni di time intelligence native (`SAMEPERIODLASTYEAR`, `DATEADD`) necessarie per i confronti anno su anno richiesti.

## 4. Misure DAX

### Ordini
```DAX
Conteggio Ordini = DISTINCTCOUNT(FACT_SALES[order_item_id])

Conteggio Ordini PY = 
CALCULATE([Conteggio Ordini], SAMEPERIODLASTYEAR(DIM_CALENDAR[Date]))

Var% Ordini MoM vs PY = 
DIVIDE([Conteggio Ordini] - [Conteggio Ordini PY], [Conteggio Ordini PY])
```

### Ricavi
```DAX
Ricavi = SUMX(FACT_SALES, FACT_SALES[price] + FACT_SALES[freight_value])

Ricavi PY = CALCULATE([Ricavi], SAMEPERIODLASTYEAR(DIM_CALENDAR[Date]))

Var% Ricavi MoM vs PY = DIVIDE([Ricavi] - [Ricavi PY], [Ricavi PY])
```

### Rating
```DAX
Rating Medio = AVERAGE(FACT_REVIEWS[review_score])

Conteggio Review = COUNTROWS(FACT_REVIEWS)

% Review per Voto = DIVIDE(COUNTROWS(FACT_REVIEWS), CALCULATE(COUNTROWS(FACT_REVIEWS), ALL(FACT_REVIEWS[review_score])))
```

Tutte le misure ereditano automaticamente il filtro su `order_status` grazie al bridge `DIM_ORDER`.

## 5. Layout del report (3 pagine + drillthrough)

Impostazione grafica comune a tutte le pagine: header fisso in alto con logo/titolo pagina e i 3 pulsanti di navigazione (Ordini · Ricavi · Rating); sotto, una barra filtri sempre visibile con: slicer **Anno**, slicer **Stato Ordine** (multi-selezione), slicer **Stato geografico (UF)**. Palette: blu petrolio per i KPI principali, grigio neutro per lo storico/PY, arancio solo per evidenziare variazioni negative.

**Pagina 1 — Andamento Ordini**
- riga KPI: card "Ordini anno selezionato", card "Ordini anno precedente", card "Var% totale"
- grafico combinato: colonne = ordini mese per mese anno corrente, linea = anno precedente
- grafico a barre orizzontali sotto: variazione % mese su mese (colore verde/rosso)
- mappa Brasile a destra: ordini per `customer_state`
- drillthrough su un mese/stato → pagina dettaglio con elenco ordini, prodotto, cliente

**Pagina 2 — Andamento Ricavi**
- stessa struttura della pagina 1, ma su `Ricavi`; aggiunta di uno scomposizione (decomposition tree o tooltip) prezzo vs freight per capire quanto pesa la spedizione
- drillthrough identico, con tabella ordini ordinata per ricavo decrescente

**Pagina 3 — Distribuzione Rating**
- istogramma voti 1–5 (barre) con % sopra ogni barra
- KPI "Rating medio" e trend del rating medio mese per mese
- due grafici di dettaglio affiancati: rating medio per categoria prodotto (top/bottom 10) e rating medio per stato (mappa o barre)
- drillthrough su categoria/stato → elenco ordini con voto basso, per analisi delle cause

**UX trasversale**
- pulsanti di navigazione con stato "attivo" evidenziato (bookmark)
- tooltip personalizzato su ogni grafico con mini scheda (ordini, ricavi, rating dello specifico mese/stato)
- pulsante "Reset filtri" in alto a destra
- drill-through disponibile da qualsiasi visual con click destro su mese, stato o categoria, per non appesantire le pagine di sintesi con troppo dettaglio

## 6. Analisi aggiuntive incluse

- **Rating per prodotto/categoria**: individua le categorie con voto medio basso e volumi alti → priorità di intervento qualità.
- **Rating per area geografica**: incrocia `customer_state` con `review_score` per capire se ci sono zone con più insoddisfazione (spesso legato ai tempi di consegna, essendo il Brasile molto esteso).
- **Peso spedizione sul ricavo**: rapporto `freight_value / (price+freight_value)` per stato, utile a capire dove la logistica incide di più sul prezzo finale.
