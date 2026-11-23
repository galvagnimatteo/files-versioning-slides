---
layout: center
class: section
---

# I client

---

<SectionLabel label="Client" />

<div class="eyebrow">Lo stesso upload, tre client</div>

# Con e <em>senza</em> versionamento

<div class="grid grid-cols-3 gap-3 mt-2">
<div class="card-teal card-accent">
  <div class="card-num">Client 3.0.14 con header</div>

```http
POST /upload
X-FILES-API-VERSION: 3.0.14
ParentId: 7d1f…

200 {"nodeId":"9a2c…"}
X-FILES-API-VERSION: 3.0.14
```

  <div class="xs">Converter applicati: richiesta e risposta. Funziona come prima.</div>
</div>
<div class="card">
  <div class="card-num">Client nuovo</div>

```http
POST /upload
X-FILES-API-VERSION: 4.0.0
DestinationId: 7d1f…

200 {"node_id":"9a2c…","version":1}
X-FILES-API-VERSION: 4.0.0
```

  <div class="xs">Servito come current, nessun converter.</div>
</div>
<div class="card-red">
  <div class="card-num" style="color:var(--red)">files-ui 3.2.3, senza header</div>

```http
POST /upload
ParentId: 7d1f…

400 Bad Request
X-FILES-API-VERSION: 4.0.0
```

  <div class="xs">Senza header = current: <strong>si rompe</strong>. E leggerebbe comunque <code>nodeId</code>.</div>
</div>
</div>

<Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/UploadApiVersioningIT.java" lines="74-81" label="test: client senza header → 400" />
<Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/UploadApiVersioningIT.java" lines="84-100" label="test: client vecchio → converter" />

<div class="card-amber text-center mt-2 small">Regola: <strong>l'header va nei client prima del primo cambio che richiede un converter.</strong> Oggi nessun client Files lo manda.</div>

---

<SectionLabel label="Client" />

<div class="eyebrow">Cosa deve fare un client</div>

# Il client <em>versionato</em>

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```ts
// a build time, dagli snapshot contro cui il client è validato
const FILES_API = { ce: '3.0.14', advanced: '4.0.14' } as const

const edition = isCarbonioCE ? 'ce' : 'advanced'
headers['X-FILES-API-VERSION'] = FILES_API[edition]

// risposta
const served = res.headers.get('X-FILES-API-VERSION')
if (res.status === 400 && body.code === 'UNSUPPORTED_API_VERSION') {
  // client troppo vecchio: chiedere l'aggiornamento
}
```

<div class="note">Pseudocodice: i client non hanno ancora l'header.</div>
</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li><strong>Backward</strong>: manda la sua versione, al resto pensa il server</li>
    <li><strong>Forward</strong>: con un server più vecchio riceve indietro la versione del server; è lui a scegliere documento o feature (GraphQL valida l'intera operazione: un campo sconosciuto la fa fallire)</li>
    <li>L'edizione la rilevano già: files-ui con <code>useIsCarbonioCE</code>, files-mobile da <code>systemType</code></li>
    <li>Release: valore concreto, mai <code>current</code></li>
  </ul>
</div>
</div>

---

<SectionLabel label="Client" />

<div class="eyebrow">Dove l'header non arriva</div>

# Route da <em>browser</em> e backend

<div class="grid grid-cols-2 gap-6 mt-3">
  <div class="card">
    <div class="head-teal">Non possono mandare header</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li><code>&lt;img src&gt;</code>, <code>a.href</code>: <code>/preview/**</code>, <code>/download/{id}</code></li>
      <li>Link pubblici aperti da utenti esterni: <code>/public/**</code>, <code>/link/**</code>, <code>/invite/**</code></li>
      <li>Sono sempre current: <strong>solo cambi additivi</strong> (nuovo path, query param opzionale)</li>
      <li>Se servisse un converter, la chiamata passa a fetch con header</li>
    </ul>
  </div>
  <div class="card">
    <div class="head-teal">Backend verso backend</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li><code>/internal/**</code>: i backend si aggiornano insieme, sempre current</li>
      <li>Fuori dallo scope del versioning</li>
      <li>Infatti l'upload interno tiene <code>ParentId</code>: il branch demo non lo tocca</li>
    </ul>
  </div>
</div>
