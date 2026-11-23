---
layout: center
class: section
---

# Le alternative

---

<SectionLabel label="Alternative" />

<div class="eyebrow">Come si identifica il contratto</div>

# Cosa abbiamo <em>scartato</em> · il modello

| Alternativa | Perché no |
|---|---|
| Nessuna versione: cambi con nome dichiarati dal client | portano la stessa informazione di un numero: "versioning numerico con passaggi in più" |
| Capability / feature flag per i cambi | con una linea per edizione e un changelog, il nome della feature non aggiunge niente. Licenza e switch admin restano runtime settings |
| Contatore di contratto separato (come WSC `1.6.x` ≠ pacchetto) | due numeri da tenere allineati a mano: la versione la decide già semantic-release |
| Valore con edizione (`advanced/4.3.0`) o dizionario RFC 9651 | i client rilevano già l'edizione da anni |
| Versione nel media type | non va con i media type fissi di GraphQL né con gli endpoint binari |
| Header assente = versione più vecchia (stile WSC) | per Files assente = current: per i client senza header non cambia niente rispetto a oggi |
| Advanced traduce la sua versione nella versione CE pinnata | sostituito: ogni converter CE va legato esplicitamente nel registry Advanced |

<div class="note mt-2">Ispirazione: Stripe e Cadwyn (versione + catena di downgrade), <code>MCP-Protocol-Version</code>, API versioning di Spring Framework 7, WSC <code>X-WSC-API-VERSION</code>.</div>

---

<SectionLabel label="Alternative" />

<div class="eyebrow">Come si garantisce che nessuno se lo dimentichi</div>

# Cosa abbiamo <em>scartato</em> · i controlli

| Alternativa | Perché no |
|---|---|
| Gate CI su diff del contratto (graphql-inspector, oasdiff, asyncapi diff) | tre tassonomie diverse, guardano solo la forma |
| Tabelle di supporto generate per i client, `supports(operation)`, manifest dei documenti persistiti | è il backend a dire cosa supporta, con i suoi test |
| Un converter obbligatorio (anche vuoto) per ogni PR o versione | ogni merge è una release, anche Renovate: una checkbox scritta in codice |
| Rieseguire gli IT della release precedente contro il build nuovo | si testa solo ciò che cambia: ogni converter ha i suoi test con `@ApiVersionBefore` |
| `oldest-supported` calcolato dai tag e rimozione automatica | la rimozione resta una PR voluta e revisionata, la CI avvisa e basta |
| Marcare ogni test con una versione | nessuna informazione tra un converter e l'altro |

<div class="card-teal card-accent mt-3 small">
  Scelto: <strong>docs del contratto cambiati ⇒ serve una riga nuova nel registry</strong>. Una regola semplice che copre il caso comune, con i buchi documentati.
</div>

---

<SectionLabel label="Alternative" />

<div class="eyebrow">Il precedente in casa</div>

# Cosa abbiamo imparato da <em>WSC</em>

<div class="grid grid-cols-2 gap-6 mt-3">
  <div class="card">
    <div class="head-teal">Cosa c'è</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Header <code>X-WSC-API-VERSION</code>, 17 versioni API in 17 mesi</li>
      <li>Un solo <code>ChangeSet</code> registrato a mano (1.6.2)</li>
      <li>Downgrade delle risposte e degli eventi WebSocket</li>
    </ul>
  </div>
  <div class="card-red">
    <div class="head-red">Cosa non rifacciamo</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Liste di versioni scritte a mano: nei client non coincidono, una cita una 1.7.0 mai esistita</li>
      <li>15 controlli di versione sparsi negli schermi della UI</li>
      <li>422 al client più nuovo invece di servirlo alla versione del server</li>
      <li>Nessuna finestra, nessun controllo che il cambio abbia un converter</li>
    </ul>
  </div>
</div>

<div class="card text-center mt-4 small">Il numero non era il problema: lo erano le liste a mano e l'assenza di controlli.</div>
