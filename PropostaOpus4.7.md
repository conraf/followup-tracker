# Proposta: hybrid (localStorage + Sheets sync), non Sheets puro

Il memo propone Sheets come storage primario (sostituisce localStorage). Sconsiglio. **Tieni localStorage come storage primario, aggiungi Sheets come layer di sync write-through asincrono.**

## Perché:

- L'app oggi è istantanea (~0ms). Sheets puro = 200-800ms su ogni operazione → UX percepibilmente più lenta su tutto: aggiungi, archivia, elimina.
- Offline continua a funzionare (oggi sì, con Sheets puro no).
- Token scaduto a metà sessione = app rotta con Sheets puro; con hybrid = save differito, recovery trasparente al re-auth.
- Refactor incrementale: puoi spedire prima la sola sync, senza toccare la logica esistente. Riduce rischio regressioni.
- Async pervasivo (15+ funzioni) = il cambiamento più costoso e rischioso del refactor monolite. Hybrid lo evita: le funzioni restano sincrone, lo `scheduleSync()` parte dopo il setItem.

Costo aggiuntivo dell'hybrid: logica di conflict resolution (timestamp `lastModified` per riga, last-write-wins) + retry queue per le scritture fallite.

---

## Tempi (effort effettivo, non calendario)

| Fase | Sheets puro | Hybrid |
|---|---|---|
| OAuth scope + Sheets API setup | 2h | 2h |
| Schema + foglio per venditore + auto-create | 4h | 3h |
| CRUD mapping + async refactor | 12-16h | 4h (write-through only) |
| Conflict/sync/retry queue | n/a | 6-8h |
| Migrazione localStorage→Sheets one-shot | 3h | 3h |
| Token expiry handling | 4h | 2h (meno critico) |
| Test multi-venditore + edge cases | 6h | 5h |
| **Totale** | **31-39h** | **25-29h** |

---

## Decisioni del memo: la mia preferenza

- **(A) Struttura Sheet:** un foglio per venditore + un foglio "Archivio_<nome>". Più semplice da scrivere, manager apre i tab. Colonna "Venditore" su foglio unico complica i filtri lato manager senza vantaggi reali per 5-15 utenti.
- **(B) Cache locale:** vedi sopra, hybrid.
- **(C) Token:** timer su `expires_in` con re-auth silenzioso a 55min. Critico anche in hybrid.
- **(D) Spreadsheet ID:** hardcoded nel codice. È un team singolo, un file solo. Drive auto-discovery è over-engineering.

---

## Rischi che il memo non evidenzia abbastanza

1. **Permessi Drive del file Sheet:** chi è il proprietario? Se è il manager, ogni venditore scrive su file altrui (serve "editor"). Se uno se ne va dall'azienda, l'account può bloccare l'accesso. Mitigazione: file in Shared Drive aziendale, non in My Drive di una persona.

2. **Race condition stesso utente, due tab aperti:** Sheets non garantisce atomicità. Hybrid mitiga (localStorage è la verità locale, sync ricompone).

3. **GitHub Pages + dominio Client ID:** lo scope Sheets va aggiunto in Cloud Console + il dominio `conraf.github.io` deve restare l'unico autorizzato. Se in futuro ognuno fa il proprio fork, esplode.

4. **Costi nulli ma quota:** 60 req/min/utente è morbido per uso normale; un import JSON da ~100 chiamate con sync immediato lo satura. Serve batching.

---

## Cosa farei concretamente, in ordine

1. Aggiungo scope Sheets all'OAuth (zero rischio, ortogonale al resto).
2. Creo helper `sheetsClient.js` con `appendRow`, `updateRow`, `deleteRow`, `readAll` — testabili in isolamento.
3. Aggiungo `scheduleSync()` chiamato dopo ogni `setItem` esistente. localStorage resta master.
4. Migrazione one-shot al primo avvio post-update.
5. Manager: nessun lavoro app-side, apre il Google Sheet direttamente.

Se vuoi procedo con uno spike (solo punto 1+2, nessuna modifica alla logica) per validare l'approccio prima di committare al refactor pieno.
