---
layout: center
class: section
---

# Invecchiamento e stato

---

<SectionLabel label="Invecchiamento" />

<div class="eyebrow">Quando un converter va in pensione</div>

# Dopo <em>15 mesi</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div>
  <ol class="tight" style="padding-left:1.2em">
    <li>La CI di main avvisa: tag ancora supportato e più vecchio di 15 mesi</li>
    <li>Una PR alza <code>oldest-supported</code> (a mano)</li>
    <li>Il build disattiva i converter con <code>since ≤ oldest-supported</code> e salta i loro test</li>
    <li>I client sotto la finestra ricevono <code>400 UNSUPPORTED_API_VERSION</code></li>
    <li>Quando vuoi, cancelli converter, tipo legacy e test, e aggiorni la loro riga nel registry</li>
  </ol>
  <Src repo="ce" path=".ci/api-guard.sh" lines="298-334" label="api-guard.sh window_warning" />
</div>
<div>
  <div class="head-teal">Release simulata 4.0.0, oldest-supported = 4.0.0</div>
  <div class="term-amber code-sm">

```text
[WARNING] converter ...UploadDestinationIdConverter:
  disabled in this build, since 4.0.0 <= oldest supported 4.0.0
[WARNING] converter ...UploadResponseConverter:
  disabled in this build, since 4.0.0 <= oldest supported 4.0.0
```

  </div>
  <Src repo="ext" path="api-versioning/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/deployment/ConverterScanner.java" lines="549-557" label="ConverterScanner: disabled" />
  <div class="card-red xs mt-2">Cancellarli <strong>prima</strong>, con la finestra ancora aperta: build e guard verdi lo stesso (provato). Lo ferma solo la review.</div>
</div>
</div>

---

<SectionLabel label="Stato" />

<div class="eyebrow">A che punto siamo</div>

# Stato <em>attuale</em>

<div class="grid grid-cols-2 gap-6 mt-2">
  <div class="card-teal">
    <div class="head-teal">Fatto</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>PoC su 4 repo: estensioni, jenkins-lib-common, files-ce, files Advanced. CI verde, mai mergiato</li>
      <li>Compatibilità con main verificata: gli IT di main passano sul PoC senza header (fallisce solo la route preview rimossa di proposito)</li>
      <li>Nessun overhead misurabile sui tempi dei test di Files</li>
      <li>Branch demo: 3 cambi reali, 657 IT verdi in locale</li>
    </ul>
  </div>
  <div class="card-red">
    <div class="head-red">Costi e limiti</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>≈7k righe di produzione nelle estensioni, più i test</li>
      <li>L'integrazione GraphQL usa tre dettagli interni di SmallRye: un upgrade può romperla, se ne accorgono solo i test</li>
      <li>Il body JSON delle richieste REST e gli status code non si convertono</li>
      <li>Payload delle subscription GraphQL mai convertiti</li>
    </ul>
  </div>
</div>

---

<SectionLabel label="Prossimi passi" />

<div class="eyebrow">Prima del merge</div>

# Prossimi <em>passi</em>

<div class="grid grid-cols-2 gap-4 mt-3">
  <div class="card card-accent"><div class="card-num">01 · Client</div><div class="small">files-ui e files-mobile mandano l'header, con la mappa <code>{ce, advanced}</code> generata al build. Prerequisito del primo converter.</div></div>
  <div class="card card-accent"><div class="card-num">02 · CI</div><div class="small">Ripinnare i Jenkinsfile di files-ce e Advanced su un tag di jenkins-lib-common.</div></div>
  <div class="card card-accent"><div class="card-num">03 · Release branch</div><div class="small">Rischio aperto: un cambio incompatibile scritto a mano su una linea vecchia arriva non convertito ai client di una linea più nuova.</div></div>
  <div class="card card-accent"><div class="card-num">04 · Guard</div><div class="small">Da decidere: una riga per ogni cambio invece di una per branch? Oggi la regola conta le righe, non i cambi.</div></div>
</div>

---
layout: center
class: section
---

# Domande?

<div class="text-center mt-6 small">
  <Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" label="branch versioning-poc-test-impl" />
  <Src repo="poc" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" label="branch poc-files-versioning" />
</div>
