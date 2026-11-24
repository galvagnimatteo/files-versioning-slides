---
layout: center
class: section
---

# Guardrail

---

<SectionLabel label="Guardrail" />

<div class="eyebrow">Tutti provati davvero, sul branch demo o sulla release simulata</div>

# Cosa ti <em>ferma</em>

| Controllo | Dove | Cosa intercetta | Sorgente |
|---|---|---|---|
| hook `maven` (`mvn package`) | pre-commit | rigenera `app/docs`, esegue le regole di build delle estensioni | <Src repo="ce" path=".pre-commit-config.yaml" lines="74-78" label="pre-commit" /> |
| hook `api-guard --staged` | pre-commit | docs cambiati senza una riga `CURRENT` nuova | <Src repo="ce" path=".ci/api-guard.sh" lines="240-309" label="api-guard.sh" /> |
| hook `api-guard`, regola converter | pre-commit + Jenkins | riga di un converter con `since` stampato sopra `oldest-supported` che sparisce o cambia `since` | <Src repo="ce" path=".ci/api-guard.sh" lines="398-441" label="converter_check" /> |
| build `api-versioning` | locale + CI | converter non legato, `operation` inesistente, `since` sopra la versione del build, `current` in un build di release | <Src repo="ext" path="api-versioning/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/deployment/ConverterScanner.java" lines="314-329" label="ConverterScanner" /> |
| build `rest` / `graphql` | locale + CI | status, parametri e codici di errore non dichiarati | <Src repo="ext" path="rest/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/deployment/ResponsePlanner.java" lines="178" label="ResponsePlanner" /> |
| `@ApiVersionBefore` | IT | converter che non cambia niente nel suo test | <Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/UploadApiVersioningIT.java" lines="84-131" label="UploadApiVersioningIT" /> |
| `ApiVersioningCoverageIT` | IT | converter attivo senza test | <Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/ApiVersioningCoverageIT.java" lines="9" label="CoverageIT" /> |
| strict mode | IT | status o codice GraphQL non dichiarato | <Src repo="ext" path="api-versioning/test/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/test/UndeclaredContractFilter.java" lines="34" label="UndeclaredContractFilter" /> |
| suite IT esistente | IT | il contratto che usano i client di oggi | — |
| stage "API Guard" + drift | Jenkins | stessa regola sulle PR (base = target); script vendorizzati diversi dalla libreria | <Src repo="jlc" path="vars/dt3_apiGuard.groovy" lines="61-71" label="dt3_apiGuard" /> |
| stage "Verify Build Outputs" | Jenkins | `app/docs` committati ≠ generati | <Src repo="jlc" path="vars/dt3_pipeline.groovy" lines="511" label="dt3_pipeline" /> |
| tag build | Jenkins | `since = current` non stampato | <Src repo="ext" path="api-versioning/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/deployment/ConverterScanner.java" lines="500-515" label="checkOwnCurrent" /> |

---

<SectionLabel label="Guardrail" />

<div class="eyebrow">Onestà: provati sul branch, passano tutti</div>

# Cosa <em>non</em> ti ferma

<div class="grid grid-cols-3 gap-3 mt-2">
  <div class="card-red xs">
    <div class="head-red">Il guard conta per branch</div>
    La rimozione C2, senza la sua riga, passa: il branch ha già più righe <code>CURRENT</code> della base. Output reale: <em>"Contract changed … and the registry has a new @ApiChange entry with since = CURRENT"</em>, exit 0.
  </div>
  <div class="card-red xs">
    <div class="head-red">Additivo o breaking, per il guard è uguale</div>
    Una riga <code>description</code> soddisfa la regola anche per una rimozione. Il guard ti obbliga a <strong>dichiarare</strong>, il giudizio resta tuo e del reviewer.
  </div>
  <div class="card-red xs">
    <div class="head-red">Tipo condiviso tra endpoint</div>
    Un tipo serializzato come entity su un endpoint e a mano su un altro: il converter gira solo sul primo, e il test vede solo l'endpoint che chiama.
  </div>
  <div class="card-red xs">
    <div class="head-red">Stessa forma, significato diverso</div>
    Un'unità di misura o una semantica che cambia senza toccare lo schema: docs uguali, nessun guard.
  </div>
  <div class="card-red xs">
    <div class="head-red">Branch impilati</div>
    Base automatica = <code>main</code>: su un branch figlio del PoC il guard passa con la riga del PoC. In Jenkins le PR usano la base giusta (il target).
  </div>
  <div class="card-red xs">
    <div class="head-red">Client senza header</div>
    Sono serviti come current: nessun converter li protegge. Vale per files-ui oggi.
  </div>
</div>

<div class="card text-center mt-3 small">I controlli prendono le <strong>dimenticanze in buona fede</strong>. Le scelte sbagliate le prende la review.</div>
