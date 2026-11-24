---
theme: default
title: "Files: API versionate"
colorSchema: light
fonts:
  sans: Plus Jakarta Sans
  mono: JetBrains Mono
  weights: '300,400,500,600,700,800'
transition: slide-left
mdc: true
class: cover
---

<div class="cover-wrap">
  <div class="badge"><span class="badge-dot"></span>Zextras · Carbonio Files</div>
  <h1 class="cover-hero">Files:<br><em>API versionate</em></h1>
  <p class="cover-sub">Cambiare il contratto senza rompere i client</p>
  <div class="cover-rule"></div>
  <p class="cover-by"><strong>Dev team 3</strong> · Novembre 2026 · PoC <code>poc-files-versioning</code></p>
</div>

---

<SectionLabel label="Il problema" />

<div class="eyebrow">Files oggi</div>

# Ogni cambio di contratto <em>rompe qualcuno</em>

<div class="grid grid-cols-2 gap-6 mt-4">
  <div class="card-red">
    <div class="head-red">Oggi</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Il server serve <strong>un solo</strong> contratto, l'ultimo, e nessun client dichiara la sua versione</li>
      <li>Client in giro: files-ui, files-mobile, public-folder-ui, link pubblici</li>
      <li>Ottobre 2026: il passaggio a GraphQL code-first rompe files-mobile</li>
      <li>Il refactor delle preview è fermo in attesa del versioning</li>
    </ul>
  </div>
  <div class="card-teal card-accent">
    <div class="head-teal">Obiettivo</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Il dominio implementa <strong>solo il comportamento corrente</strong></li>
      <li>Ogni cambio incompatibile ha un piccolo <strong>converter</strong> che traduce per i client vecchi</li>
      <li>Supporto per <strong>15 mesi</strong>, poi si alza la versione minima e il converter va cancellato: il guard lo impone</li>
      <li>Se dimentichi qualcosa, te lo dicono build e CI</li>
    </ul>
  </div>
</div>

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Un header con la versione del pacchetto</div>

# Il client dice <em>chi è</em>, il server si adatta

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```http
POST /graphql
X-FILES-API-VERSION: 3.0.14

HTTP/1.1 200 OK
X-FILES-API-VERSION: 3.0.14
```

<div class="note mt-2">Sotto la finestra: <code>400 UNSUPPORTED_API_VERSION</code>. Più nuovo del server: servito con il contratto del server, che lo dichiara nella risposta.</div>

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Il numero è la <strong>versione del pacchetto così com'è</strong>, decisa da semantic-release (CE <code>3.x</code>, Advanced <code>4.x</code>): ogni release la muove, anche quando non tocca le API</li>
    <li><strong>Nessun header = current</strong>: chi non lo manda vede il comportamento nuovo, come oggi</li>
    <li><strong>Backward</strong> (client vecchio, server nuovo): converte il server</li>
    <li><strong>Forward</strong> (client nuovo, server vecchio): il server dichiara la sua versione, si adatta il client</li>
  </ul>
</div>
</div>

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Un file per servizio: FilesApiChanges</div>

# Il <em>registry</em>: una riga per ogni cambio

<div class="grid grid-cols-2 gap-6 mt-2">
<div class="code-sm">

```java
// incompatibile: i client vecchi passano dal converter
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadDestinationIdConverter.class)

// additivo, deprecazione, rimozione: solo descrizione
@ApiChange(since = ApiVersion.CURRENT,
    description = "POST /upload/check: pre-flight of an upload")

final class FilesApiChanges {}
```

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>È il <strong>changelog del contratto</strong> e l'unico posto dove compare una versione</li>
    <li>In PR si scrive sempre <code>since = ApiVersion.CURRENT</code>: il numero vero lo scrive la release</li>
    <li>Sui build di sviluppo una riga <code>CURRENT</code> vale per ogni client che manda una versione, quindi i test la esercitano subito</li>
    <li>Advanced ha il suo registry: quando alza il pin di CE lega i converter ereditati con <code>inherits</code> o <code>replaces</code></li>
  </ul>
</div>
</div>

---

<SectionLabel label="Cosa fa il dev" />

<div class="eyebrow">Esempio reale: POST /upload, da ParentId opzionale a DestinationId obbligatorio</div>

# Un cambio incompatibile in <em>quattro passi</em>

<div class="grid gap-6 mt-1" style="grid-template-columns: 2fr 3fr">
<div>
  <div class="mb-3 small" style="display:flex;align-items:baseline"><span class="step" style="flex-shrink:0">1</span><span><strong>Cambia la resource</strong>: il dominio parla solo il contratto nuovo, il build rigenera i docs</span></div>
  <div class="mb-3 small" style="display:flex;align-items:baseline"><span class="step" style="flex-shrink:0">2</span><span><strong>Scrivi il converter</strong>: riporta la richiesta vecchia alla forma nuova, o la risposta nuova alla vecchia</span></div>
  <div class="mb-3 small" style="display:flex;align-items:baseline"><span class="step" style="flex-shrink:0">3</span><span><strong>Aggiungi la riga</strong> nel registry</span></div>
  <div class="mb-3 small" style="display:flex;align-items:baseline"><span class="step" style="flex-shrink:0">4</span><span><strong>Scrivi il test</strong> da client vecchio: il kit manda la versione più vecchia supportata</span></div>
  <div class="note">Il resto del dominio e i test esistenti non sanno niente delle versioni.</div>
</div>
<div class="code-sm">

```java
// 2 · converter
@ApiConverter(operation = "...BlobResource#upload")
public class UploadDestinationIdConverter implements RestRequestConverter {
  public void convert(RestRequest request) {
    String parentId = request.header(PARENT_ID).orElse(LOCAL_ROOT);
    request.removeHeader(PARENT_ID);
    request.setHeader(DESTINATION_ID, parentId);
  }
}

// 3 · registry
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadDestinationIdConverter.class)

// 4 · test
@Test
@ApiVersionBefore(UploadDestinationIdConverter.class)
void anOldClientUploadsIntoItsParentId() { ... }
```

</div>
</div>

---

<SectionLabel label="Cosa fa il dev" />

<div class="eyebrow">Il tipo di cambio decide cosa scrivere</div>

# Non tutti i cambi <em>vogliono un converter</em>

| Cambio | Esempio | Nel registry |
|---|---|---|
| <span class="pill pill-teal">additivo</span> | nuovo endpoint, campo o parametro opzionale | riga `description` |
| <span class="pill pill-red">incompatibile</span> | campo rinominato, parametro diventato obbligatorio | riga `converter`, più converter e test |
| <span class="pill pill-amber">deprecazione</span> | `@Deprecated` su una query | riga `description` |
| <span class="pill pill-red">rimozione</span> | query o endpoint tolto | riga `description`: nessun converter può riportarlo |

<div class="grid grid-cols-2 gap-4 mt-4 small">
  <div class="card-amber">Rimuovere è un processo: <strong>depreca → i client migrano → la finestra supera la deprecazione → rimuovi</strong>. Nessun controllo sa se i client hanno smesso di usarla.</div>
  <div class="card">Il numero non dipende dal tipo di cambio API: è la <strong>versione del pacchetto</strong>, che semantic-release alza dai commit come ha sempre fatto. Le API non hanno regole di commit proprie.</div>
</div>

---

<SectionLabel label="Cosa fa il dev" />

<div class="eyebrow">Da sapere prima di progettare il cambio</div>

# Cosa si può <em>convertire</em>

| Converter | Interfaccia | Converte |
|---|---|---|
| Richiesta REST | `RestRequestConverter` | header e query param |
| Argomenti GraphQL | `GraphqlArgumentConverter` | valori degli argomenti |
| Valore | `ValueConverter<T>` | valori in risposta, REST e GraphQL; in REST anche la forma, con un tipo legacy |

<div class="grid grid-cols-2 gap-4 mt-4">
  <div class="card-red small">
    <div class="head-red">Non si converte</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>il body JSON delle richieste REST</li>
      <li>status code e codici di errore</li>
      <li>un'operazione o un campo GraphQL rimossi</li>
      <li>i payload delle subscription GraphQL</li>
    </ul>
  </div>
  <div class="card-teal small">
    <div class="head-teal">Regole pratiche</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Se il cambio non è convertibile, rendilo <strong>additivo</strong></li>
      <li>Le risposte REST vanno restituite come <strong>entity</strong>: sul JSON scritto a mano il converter non gira</li>
      <li>Il tipo legacy vive accanto al converter e si cancella con lui</li>
    </ul>
  </div>
</div>

---

<SectionLabel label="Rete di sicurezza" />

<div class="eyebrow">Ogni passo saltato ha un controllo che lo prende, tutti provati sul branch demo</div>

# Se sbagli, <em>chi ti ferma</em>

<div class="grid grid-cols-2 gap-3 mt-1 xs">
  <div class="card">
    <div class="head-teal">Al commit · pre-commit</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Contratto cambiato senza una riga nuova nel registry → <strong>commit bloccato</strong></li>
      <li>Converter ancora in finestra cancellato → <strong>commit bloccato</strong></li>
      <li>Converter fuori finestra lasciato nel registry → <strong>commit bloccato</strong></li>
      <li>Docs non rigenerati → li rigenera l'hook</li>
    </ul>
  </div>
  <div class="card">
    <div class="head-teal">Al build · locale e CI</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Converter senza riga nel registry → <strong>il build non parte</strong> e indica la riga da aggiungere</li>
      <li><code>operation</code> inesistente → <strong>build rosso</strong></li>
      <li>Status, parametri o errori non dichiarati nei docs → <strong>build rosso</strong></li>
    </ul>
  </div>
  <div class="card">
    <div class="head-teal">Nei test · IT</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Converter senza test → <strong>rosso</strong> il test di copertura</li>
      <li>Converter che nel suo test non cambia niente → <strong>test rosso</strong></li>
      <li>Status non dichiarato restituito a runtime → <strong>test rosso</strong></li>
      <li>I test esistenti sono i client di oggi: un cambio non voluto li rompe</li>
    </ul>
  </div>
  <div class="card">
    <div class="head-teal">In PR e alla release · Jenkins</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Stesse regole del guard sulla PR, contro il branch target</li>
      <li>Docs committati diversi da quelli generati → <strong>build rosso</strong></li>
      <li>Release senza stamping → <strong>la release si ferma</strong>, il tag build rifiuta <code>current</code></li>
      <li>Converter portato su una linea più vecchia → <strong>build rosso</strong></li>
    </ul>
  </div>
</div>

---

<SectionLabel label="Rete di sicurezza" />

<div class="eyebrow">Provati anche questi: passano senza errori</div>

# Dove la rete <em>non arriva</em>

| Cosa passa | Perché | Chi lo prende |
|---|---|---|
| Un secondo cambio senza la sua riga, o una `description` dove serviva un converter | il guard chiede almeno una riga per PR, non una per ogni operazione toccata, e non ne giudica il tipo | review |
| Stessa forma, significato diverso (unità, semantica) | i docs non cambiano | review |
| Client che non mandano l'header | sono serviti come current | header nei client prima del primo converter |

<div class="card text-center mt-4 small">I controlli prendono le <strong>dimenticanze in buona fede</strong>. Le scelte sbagliate le prende la review.</div>

---

<SectionLabel label="Release e finestra" />

<div class="eyebrow">Dalla PR alla cancellazione: il since lo scrive la release, non il dev</div>

# La vita di <em>un converter</em>

<div class="timeline" style="margin-top:.6rem">
  <div class="tl-line"></div>
  <div class="tl-item"><div class="tl-dot done"></div><div class="tl-bubble"><div class="tl-bubble-date">1 · PR</div><div class="tl-bubble-title">since = CURRENT</div><div class="tl-bubble-note">vale per ogni client con versione: i test lo esercitano subito</div></div></div>
  <div class="tl-item"><div class="tl-dot done"></div><div class="tl-bubble"><div class="tl-bubble-date">2 · merge</div><div class="tl-bubble-title">semantic-release</div><div class="tl-bubble-note">ogni merge su main è una release, il tipo di commit decide il numero</div></div></div>
  <div class="tl-item"><div class="tl-dot today"></div><div class="tl-bubble today-bubble"><div class="tl-bubble-date">3 · stamping</div><div class="tl-bubble-title">CURRENT → "3.1.0"</div><div class="tl-bubble-note">scritto nel commit di release, per sempre</div></div></div>
  <div class="tl-item"><div class="tl-dot"></div><div class="tl-bubble"><div class="tl-bubble-date">4 · 15 mesi</div><div class="tl-bubble-title">il build di main avvisa</div><div class="tl-bubble-note">una riga nel log elenca le versioni fuori finestra ancora supportate</div></div></div>
  <div class="tl-item"><div class="tl-dot"></div><div class="tl-bubble"><div class="tl-bubble-date">5 · una PR</div><div class="tl-bubble-title">alza oldest-supported e cancella</div><div class="tl-bubble-note">converter, tipo legacy e test insieme: il guard blocca chi ne lascia uno, e chi lo cancella prima</div></div></div>
</div>

<div class="grid grid-cols-2 gap-6 mt-4">
<div class="code-sm">

```diff
 # chore(release): 3.1.0
-@ApiChange(since = ApiVersion.CURRENT,
+@ApiChange(since = "3.1.0",
     converter = UploadDestinationIdConverter.class)
```

</div>
<div class="small">
  Da qui i client sotto <code>3.1.0</code> passano dal converter; quelli da <code>3.1.0</code> in su, o senza header, no.
  <div class="note mt-2">Provato con semantic-release vero su una release simulata, senza push né tag.</div>
</div>
</div>

---

<SectionLabel label="I client" />

<div class="eyebrow">Prerequisito del primo converter</div>

# Cosa deve fare <em>un client</em>

<div class="grid grid-cols-2 gap-6 mt-2">
<div class="code-sm">

```ts
// generata al build, dalle versioni contro cui il client è validato
const FILES_API = { ce: '3.0.14', advanced: '4.0.14' } as const

headers['X-FILES-API-VERSION'] = FILES_API[isCarbonioCE ? 'ce' : 'advanced']

if (res.status === 400 && body.code === 'UNSUPPORTED_API_VERSION') {
  // client troppo vecchio: chiedere l'aggiornamento
}
```

</div>
<div>
  <div class="card-amber small"><strong>L'header va nei client prima del primo cambio con converter.</strong> Oggi nessun client Files lo manda: un client senza header riceve il contratto nuovo.</div>
  <ul class="tight mt-3" style="padding-left:1.1em">
    <li>Sempre un valore concreto della propria edizione, mai <code>current</code></li>
    <li>Link e route da browser (<code>&lt;img src&gt;</code>, download, <code>/public/**</code>) non mandano header: restano current, lì solo cambi additivi</li>
    <li><code>/internal/**</code>: i backend si aggiornano insieme, fuori scope</li>
  </ul>
</div>
</div>

---

<SectionLabel label="Stato" />

<div class="eyebrow">PoC, non mergiato</div>

# Dove <em>siamo</em>

<div class="card-teal mt-4">
  <ul class="tight" style="margin:0;padding-left:1.1em">
    <li>PoC su 4 repo: estensioni Quarkus, jenkins-lib-common, files-ce, files Advanced</li>
    <li>Gli IT di main passano sul PoC senza header: il contratto di oggi non cambia (tranne una route preview tolta di proposito)</li>
    <li>Branch demo con cambi reali (modifica, aggiunta, deprecazione, rimozione), IT tutti verdi</li>
    <li>Nessun overhead misurabile sui tempi dei test</li>
  </ul>
</div>

---
layout: center
class: section
---

# Domande?

<div class="text-center mt-6 small">
  <Src repo="pochead" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" label="branch poc-files-versioning" />
  <Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" label="branch demo versioning-poc-test-impl" />
</div>
