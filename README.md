# folkefakta-anker

Det offentlige ankerdepotet for Folkefaktas hvelv (vault). Ingen kode, intet
redaksjonelt innhold, ingen persondata — kun hasher, metadata og
bevisformater.

(English summary at the bottom of this file.)

## Hva er dette depotet?

Folkefakta er en norsk plattform for politisk etterrettelighet. Deler av det
den finner, forsegles i et internt hvelv i stedet for å publiseres
umiddelbart — funn som er systemisk viktige, men der bevisgrunnlaget ennå
ikke er modent nok til å stå for seg selv. Det forseglede innholdet lagres
kryptert, privat, og er ikke offentlig.

Det som derimot er offentlig, er beviset for at et gitt funn fantes, i en
gitt form, på et gitt tidspunkt — uten at innholdet avsløres. Det er det
dette depotet er: en append-only historikk av kryptografiske hasher og
tidsstempler, uavhengig av Folkefaktas egen database.

Depotet inneholder tre typer poster:

- `vault/<finding-id>.txt` — forseglingsposter: SHA-256-hash av det
  kanoniske funnet, sektor og tidspunkt for forsegling.
- `vault/<finding-id>.unseal.txt` — avforseglingsposter: tidspunkt, modus
  (full/redigert/fortsatt forseglet) og hva som utløste ny behandling, når
  et forseglet funn siden blir tatt opp igjen.
- `ledger/<YYYY-MM-DD>.txt` — daglige hendelseslogg-røtter: én samlet hash
  per dag over hele den operative hendelsesloggen (standing-hendelser,
  tribunalmeninger, hvelvhendelser, intervensjoner og saks-overganger).
  Dette utvider samme prinsipp fra enkeltfunn til hele driftshistorikken —
  også dager uten hendelser får en rotpost, fordi stillhet også er en
  påstand som kan etterprøves.

Postfiler kan ha en `.ots`-nabofil — et OpenTimestamps-bevis som forankrer
filen til Bitcoin-blokkkjeden, uavhengig av både GitHub og Folkefakta.

## Tillitsmodell — les dette ærlig

Git-historikk under depoteierens kontroll er i prinsippet forfalskbar av
depoteieren selv: den som har skriverettigheter til dette depotet, kan i
teorien slette og skrive om historikken. Det er nettopp derfor
OpenTimestamps-bevisene finnes — en `.ots`-fil forankrer en post til
Bitcoin-blokkjeden på et tidspunkt en hvilken som helst utenforstående kan
etterprøve uavhengig, uten å måtte stole på GitHub eller på Folkefakta. Uten
et slikt bevis er alt du sitter igjen med, en commit-datostemplet
git-historikk du må stole på depoteieren for.

Den operative databasen bak Folkefakta er mutable (rader kan endres, rettes
og slettes — blant annet fordi persondata skal kunne slettes på ekte, jamfør
personvernforordningen). Dette ankerdepotet beviser derfor ikke at databasen
til enhver tid er "riktig". Det beviser bare om historikken siden forrige
ankring har endret seg. Hvis en hash i dette depotet ikke lenger stemmer med
det tilhørende innholdet, er det bevis for at noe er endret — ikke bevis for
hva som opprinnelig var sant.

**Signerte commits — ærlig status per 2026-08-07.** Depotet er *konfigurert*
for SSH-commit-signering (`gpg.format ssh`, `user.signingkey` pekt på
`~/.ssh/id_ed25519.pub`, samme nøkkel som er publisert i `allowed_signers`
nedenfor). I praksis er signering **ikke slått på for de automatiske
commitene** (ankringsjobben som kjører ubevoktet, uten menneske til stede):
den private nøkkelen er passordbeskyttet, og en ubevoktet prosess kan verken
skaffe eller skal ha tilgang til det passordet. Inntil depoteieren enten
laster nøkkelen inn i en vedvarende ssh-agent for interaktive commits, eller
setter opp en egen passordløs nøkkel viet automatiserte commits (og legger
den til i `allowed_signers`), er commitene i dette depotet **usignerte**.
Datostempling hviler i mellomtiden på git-historikken alene (se avsnittet
over) og på OpenTimestamps-bevisene, som er uavhengige av signering i det
hele tatt. Sjekk alltid `git log --show-signature -1` på en aktuell commit
for å se om signering faktisk er aktiv akkurat nå — ikke stol på dette
avsnittet alene, det oppdateres ikke nødvendigvis i sanntid.

## Nøyaktige postformater

**Forseglingspost** (`vault/<finding-id>.txt`):

```
entry: seal
finding: 7d9e1c2a-0000-0000-0000-000000000000
sector: helse
sealed_at: 2026-08-07T12:00:00.000Z
sha256: bbb8e7667558a86d3622dafcc40ea84605d1835a87b06ee47ea8adac48be8d7a
```

**Avforseglingspost** (`vault/<finding-id>.unseal.txt`):

```
finding: 7d9e1c2a-0000-0000-0000-000000000000
unsealed_at: 2026-09-01T14:00:00.000Z
mode: full
trigger: riksrevisjonen:dok3:2026:2
```

**Daglig ledger-rot** (`ledger/<YYYY-MM-DD>.txt`):

```
day: 2026-08-07
events: 42
root: 8279ba93e454bdfe6fe5aa7fbe76823b34c7206fa3f5ab2c7c4c5ec8a3510ebd
```

Alle postfiler bruker `\n` linjeskift og avsluttes med en linjeskift. Tider
er alltid normalisert til ISO 8601 UTC (`new Date(x).toISOString()`).

## Kanonisering

`sha256`-verdien i en forseglingspost, og `root`-verdien i en ledger-post,
er begge SHA-256 over en **kanonisk JSON-serialisering** av det underliggende
artefaktet: nøkler sortert alfabetisk på hvert objektnivå, arrayer bevarer
sin opprinnelige rekkefølge, ingen mellomrom eller linjeskift i
serialiseringen, og hashen tas over UTF-8-bytene av resultatet. Regelen
gjelder rekursivt gjennom hele objektet.

Dette gjør at hvem som helst kan regne ut hashen på nytt fra et publisert
eller avforseglet artefakt, uten noe av Folkefaktas egen kode — kun dette
selvstendige uttrekket:

```js
// node verify.js < artifact.json
function canonical(v) {
  if (v === null || typeof v !== 'object') return JSON.stringify(v);
  if (Array.isArray(v)) return '[' + v.map(canonical).join(',') + ']';
  return '{' + Object.keys(v).sort().map((k) => JSON.stringify(k) + ':' + canonical(v[k])).join(',') + '}';
}
const crypto = require('crypto');
let raw = '';
process.stdin.on('data', (c) => { raw += c; });
process.stdin.on('end', () => {
  console.log(crypto.createHash('sha256').update(canonical(JSON.parse(raw)), 'utf8').digest('hex'));
});
```

## Slik verifiserer du

1. **Hash av innholdet.** Kjør `node verify.js < artifact.json` (snippet
   over) på et publisert eller avforseglet artefakt. Sammenlign resultatet
   mot `sha256:`-linjen i finnetes fil under `vault/`.
2. **Tidspunkt.** Se når posten faktisk ble skrevet til historikken:
   `git log --follow -- vault/<finding-id>.txt`. Commit-datoen er
   depoteierens påstand om tidspunkt; se punkt 4 for hvorfor du ikke trenger
   å stole blindt på den.
3. **Signatur, hvis til stede.** Sjekk om commiten er signert av nøkkelen
   som er publisert i dette depotet:
   ```
   git -c gpg.ssh.allowedSignersFile=allowed_signers verify-commit HEAD
   ```
   (bytt `HEAD` med aktuell commit-hash for eldre poster). Per 2026-08-07 er
   ikke alle commits i dette depotet signert (se avsnittet "Signerte
   commits" over for hvorfor) — en manglende signatur er ikke i seg selv et
   avvik. Er den til stede, beviser den at posten kommer fra depoteieren —
   ikke at depoteieren ikke kunne ha skrevet om historikken i ettertid (se
   tillitsmodellen over).
4. **OpenTimestamps-bevis.** Dette er beviset som er uavhengig av
   depoteieren. To måter å sjekke en `.ots`-fil på:
   - **Uten installasjon:** dra postfilen (f.eks. `vault/<id>.txt`) og dens
     `.ots`-nabofil inn på https://opentimestamps.org — tjenesten viser om
     og når filen ble forankret i Bitcoin-blokkjeden.
   - **Kommandolinje:** `ots verify vault/<finding-id>.txt.ots` (krever en
     lokal Bitcoin-node, eller en ekstern kalender-tjeneste for
     tidligfase-bevis som ennå ikke er fullt forankret).

   Ferske `.ots`-bevis kan i en periode vise "pending" — det betyr at
   beviset er sendt inn til en kalendertjeneste, men ennå ikke bekreftet i
   en Bitcoin-blokk. Det blir gyldig og verifiserbart automatisk når det
   skjer, uten at noen i Folkefakta må gjøre noe.

   Hvis en postfil ikke (ennå) har noen `.ots`-nabofil: per i dag ankres
   hasher kun i git-historikken. Det betyr at tidsstemplene er sporbare, men
   i prinsippet forfalskbare av repo-eieren. OpenTimestamps-bevis ettersendes
   fortløpende for eldre poster; til de finnes for en gitt post, bør du lese
   datoene for den posten med den forutsetningen.

## allowed_signers

Filen `allowed_signers` i roten av dette depotet inneholder den offentlige
SSH-nøkkelen som brukes til å signere commits her, på formatet git
`verify-commit` forventer (ett prinsipal-navn etterfulgt av nøkkelen). Den
er selv en committet, historikkført fil — endres den, vises det i
`git log -- allowed_signers`.

---

## English summary

This is the public anchor repository for Folkefakta's vault: an
append-only history of SHA-256 hashes, sector tags, and dates proving a
finding existed, in a given form, at a given time — without revealing its
content. `vault/<finding-id>.txt` holds seal records,
`vault/<finding-id>.unseal.txt` holds unseal records, and
`ledger/<YYYY-MM-DD>.txt` holds a daily root hash over the full operational
event log (including zero-event days — silence is itself provable). Record
files may carry a `.ots` OpenTimestamps proof anchoring them to the Bitcoin
blockchain, independent of GitHub and of Folkefakta.

**Trust model, stated honestly:** git history under the repo owner's
control is, in principle, forgeable by the repo owner. That is exactly why
OpenTimestamps proofs exist — they anchor a record to Bitcoin at a point in
time any outside party can verify independently, without trusting GitHub or
Folkefakta. Without one, all you have is a committer-dated git history you
must trust the repo owner for. The operational database behind Folkefakta
is mutable; these anchors prove only whether history has changed since the
last anchor — not that the database is currently "correct."

**Signing status, honestly, as of 2026-08-07:** this repo is configured for
SSH commit signing (`gpg.format ssh`, `user.signingkey` set to the key
published in `allowed_signers`), but signing is not actually active for the
automated commits made by the unattended anchoring job — the private key is
passphrase-protected, and an unattended process neither has nor should have
that passphrase. Until the owner either loads the key into a persistent
ssh-agent for interactive commits, or provisions a dedicated
passphrase-less key for automated commits (added to `allowed_signers`),
commits in this repo are unsigned. Check `git -c
gpg.ssh.allowedSignersFile=allowed_signers verify-commit HEAD` on any given
commit to see whether a signature is actually present — do not assume it
from this paragraph alone. GitHub's "Verified" badge additionally requires
the owner to separately upload the key as a Signing Key in GitHub account
settings (not done as of 2026-08-07); that step is not required for
independent verification, which needs nothing from GitHub — only the
published public key.

**To verify:** hash a published/unsealed artifact with the `verify.js`
snippet above and compare it to the `sha256:` line in the finding's vault
file; check the commit date with `git log --follow -- vault/<id>.txt`;
verify the commit signature with
`git -c gpg.ssh.allowedSignersFile=allowed_signers verify-commit HEAD`; and
verify the `.ots` proof either by dragging the record file and its `.ots`
neighbor onto https://opentimestamps.org, or with `ots verify` on the
command line. Per i dag: not every record necessarily has an `.ots` proof
yet (they are stamped and upgraded on an ongoing basis) — where one is
absent, treat that record's date as git-history-only until it lands.
