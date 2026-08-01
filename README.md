# FantaList — gestione del minigioco

Sito in un solo file (`index.html`) che raccoglie i pronostici dei partecipanti,
li fa approvare all'organizzatore e tiene aggiornato il foglio Excel della lega.
Si appoggia alla stessa identità visiva del sito "La Lega" già nella repository.

---

## Cosa fa

**Partecipante** — accede con nome utente e password, vede la sua schedina della
giornata, sceglie una squadra tra quelle che non ha ancora usato e la invia.
Finché non hanno inviato tutti, le scelte degli altri restano coperte.

**Organizzatore** — vede il tabellone delle richieste (chi ha inviato, chi manca,
chi ha scelto cosa), approva o respinge con una nota, può fissare un termine di
invio, scegliere al posto di chi scrive su WhatsApp, segnare l'esito di ogni
squadra e chiudere la giornata. Da lì eliminazioni, storico e giornata successiva
sono automatici.

**Excel** — il file della lega si carica all'inizio e si riscarica aggiornato
quando serve: X sulle squadre usate, `V (Squadra)` / `NV (Squadra)` sulle
colonne Giornata. Gli altri fogli restano intatti.

---

## Passo 1 — Archivio condiviso (Supabase, gratuito)

Il sito è statico: da solo non ha un posto dove tenere account e pronostici.
Senza archivio funziona, ma i dati restano sul singolo dispositivo — buono per
provarlo, inutile per farci giocare la lega.

1. Crea un account su supabase.com e un nuovo progetto.
2. Apri **SQL Editor** e lancia:

```sql
create table if not exists fantalist_kv (
  k          text primary key,
  v          text not null,
  updated_at timestamptz not null default now()
);

alter table fantalist_kv enable row level security;

create policy "accesso pubblico al minigioco"
  on fantalist_kv for all
  to anon
  using (true) with check (true);
```

3. Vai in **Project settings → API** e copia *Project URL* e la chiave
   *anon public*.
4. Aprili in `index.html`, riga ~310, nel blocco `CONFIG`:

```js
const CONFIG = {
  supabaseUrl:     "https://xxxxxxxx.supabase.co",
  supabaseAnonKey: "eyJhbGciOi…",
  tableName:       "fantalist_kv"
};
```

In alternativa puoi incollarli dal sito stesso (banner rosso → *collega
Supabase*), ma valgono solo su quel dispositivo: per il sito pubblico servono
dentro al file.

## Passo 2 — Pubblicazione

Carica `index.html` nella repository `FantaList` e attiva **Settings → Pages →
Deploy from a branch → main / (root)**. Il sito esce su
`https://lubes98.github.io/FantaList/`.

Se vuoi tenere anche il sito attuale della lega, rinomina questo in
`minigioco.html` e mettici un link dalla voce "Minigioco".

## Passo 3 — Primo avvio

1. Apri il sito: la prima schermata chiede di creare l'account
   dell'organizzatore. È l'unico che nasce con i poteri di approvazione.
2. Carica `FantaList_2627.xlsx`. Con le due spunte attive vengono presi i 9
   partecipanti reali (i "partecipante 12" e chi non ha la x su *Quota
   MiniGiochi* restano fuori).
3. Manda il link agli altri: alla voce **Primo accesso** scelgono il proprio
   nome e si creano la password. Ogni nome si collega una volta sola.

---

## Come viene letto il foglio Excel

Nel foglio il cui nome contiene "Minigioco":

| Colonna | Significato |
|---|---|
| `Partecipanti` | i nomi in gioco |
| una colonna per squadra | `X` = squadra già usata da quel partecipante |
| `Giornata N` | `V (Inter)` se la squadra ha vinto, `NV (Inter)` se no |

Il sito riparte dalla prima giornata senza esiti. Una `X` su una squadra che non
compare in nessuna colonna Giornata viene letta come pronostico già dato per la
giornata in corso, e importata come richiesta approvata.

Chi ha una `NV` risulta eliminato da quella giornata. Il foglio *Partecipanti e
Premi* serve solo a capire chi ha pagato la quota minigiochi: non viene mai
modificato.

---

## Sicurezza — cosa aspettarsi

Le password non sono salvate in chiaro: di ognuna resta un'impronta PBKDF2
(SHA-256, 150.000 giri, sale casuale per utente). Basta a evitare che un
partecipante mandi il pronostico spacciandosi per un altro.

Non è però una fortezza: la chiave `anon` di Supabase è pubblica per definizione,
quindi chi si mette d'impegno può leggere la tabella e vedere impronte e
pronostici. Per una lega tra amici va benissimo — l'importante è **non riusare
qui una password che usi altrove**.

Un consiglio pratico: attiva in Supabase i backup automatici o esporta ogni
tanto il file Excel aggiornato. È la copia di sicurezza più leggibile che hai.

---

## Note pratiche

- La pagina si aggiorna da sola ogni 8 secondi: durante la giornata puoi
  lasciarla aperta e vedere le richieste arrivare.
- *Approva da solo i pronostici validi* (sezione Richieste) salta il passaggio
  di approvazione quando non hai voglia di controllare uno per uno.
- *Riapri l'ultima giornata* (sezione File Excel) annulla una chiusura
  sbagliata: esiti ed eliminazioni tornano modificabili.
- Password dimenticata: la azzeri tu da **Partecipanti → Nuova password**.
- Il file gira anche offline da un doppio clic, ma in quel caso i dati restano
  su quel computer.
