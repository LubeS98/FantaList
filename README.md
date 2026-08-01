[README (4).md](https://github.com/user-attachments/files/30614776/README.4.md)
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
3. Vai in **Partecipanti e accessi** e premi *Crea gli accessi mancanti*: per
   ognuno viene generato un codice, e l'elenco completo finisce negli appunti,
   pronto da incollare in chat. In alternativa lascia che si registrino da soli
   dalla voce **Primo accesso**: scelgono il proprio nome e si scelgono il
   codice.

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

## Nomi utente e codici di accesso

**Il nome utente non si sceglie.** È il nome del partecipante come sta nella
colonna *Partecipanti* del file Excel, normalizzato:

| Nel file | Nome utente |
|---|---|
| Salvo | `salvo` |
| Peppe Verbaro | `peppe-verbaro` |
| Giacomo Rustico | `giacomo-rustico` |

In fase di accesso si può scrivere anche "Peppe Verbaro" con maiuscole e spazi:
il sito normalizza da solo. Se cambi un nome nel file Excel e reimporti, cambia
anche il nome utente: l'accesso vecchio va ricreato.

**I codici li vedi tutti.** Nella sezione *Partecipanti e accessi* c'è una
colonna Codice, coperta finché non premi *Mostra i codici* (utile se capita di
condividere lo schermo). Da lì puoi copiare la riga di un singolo partecipante,
copiare l'elenco completo, cambiare un codice o generarne di nuovi in blocco.
I codici generati sono di sette caratteri senza lettere ambigue (niente `l`,
`o`, `0`, `1`), fatti per essere dettati al telefono.

## Sicurezza — cosa aspettarsi

Perché tu possa leggerli, **i codici sono salvati in chiaro** nell'archivio,
accanto all'impronta PBKDF2 che serve alla verifica dell'accesso. È una scelta
consapevole, ma ha un prezzo da conoscere: la chiave `anon` di Supabase è
pubblica per definizione, quindi chi ha il link al sito e un po' di pazienza può
leggere la tabella — e adesso quello che ci trova sono codici utilizzabili, non
impronte.

Per una lega tra amici è un compromesso ragionevole: quello che si può fare con
quei codici è mandare il pronostico al posto di qualcun altro. Diventa un
problema solo se una persona riusa lì una password vera. Perciò:

- fai generare i codici al sito invece di lasciarli scegliere;
- di' esplicitamente ai partecipanti che tu li vedi;
- non riusare mai qui la password della mail o della banca.

Se un domani preferisci il contrario — nessuno legge i codici, nemmeno tu, e
chi dimentica se lo fa rigenerare — si torna indietro togliendo il campo
`password` da `createUser` e `changePassword`.

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
- Codice dimenticato: lo leggi tu da **Partecipanti e accessi**, oppure ne
  imposti uno nuovo con *Cambia codice*.
- Il file gira anche offline da un doppio clic, ma in quel caso i dati restano
  su quel computer.
