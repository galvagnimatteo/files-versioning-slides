---
layout: center
class: section
---

# Release

---

<SectionLabel label="Release" />

<div class="eyebrow">Ogni merge su main è una release</div>

# Da <em>current</em> a un numero vero

<div class="timeline" style="margin-top:.6rem">
  <div class="tl-line"></div>
  <div class="tl-item"><div class="tl-dot done"></div><div class="tl-bubble"><div class="tl-bubble-date">1 · merge</div><div class="tl-bubble-title">Squash su main</div><div class="tl-bubble-note">una PR = un commit = una release</div></div></div>
  <div class="tl-item"><div class="tl-dot done"></div><div class="tl-bubble"><div class="tl-bubble-date">2 · analyze</div><div class="tl-bubble-title">Tipo di commit</div><div class="tl-bubble-note">feat → minor, feat! → major, il resto → patch</div></div></div>
  <div class="tl-item"><div class="tl-dot today"></div><div class="tl-bubble today-bubble"><div class="tl-bubble-date">3 · prepare</div><div class="tl-bubble-title">Stamping</div><div class="tl-bubble-note">pom, PKGBUILD, registry</div></div></div>
  <div class="tl-item"><div class="tl-dot"></div><div class="tl-bubble"><div class="tl-bubble-date">4 · git</div><div class="tl-bubble-title">chore(release)</div><div class="tl-bubble-note">commit [skip ci] + tag vX.Y.Z</div></div></div>
  <div class="tl-item"><div class="tl-dot"></div><div class="tl-bubble"><div class="tl-bubble-date">5 · tag build</div><div class="tl-bubble-title">Artefatti</div><div class="tl-bubble-note">qui niente guard, ma il build rifiuta current</div></div></div>
</div>

<div class="grid grid-cols-2 gap-5 mt-3 code-sm">
<div>

```bash
# prepareCmd di @semantic-release/exec
sed -i 's|<revision>.*</revision>|<revision>$V</revision>|' pom.xml
sed -i 's/^pkgver=.*/pkgver="$V"/' package/PKGBUILD
bash .ci/stamp-api-changes.sh $V .../versioning/FilesApiChanges.java
```

</div>
<div>

```json
// assets di @semantic-release/git: finiscono nel commit di release
["pom.xml", "CHANGELOG.md", "package/PKGBUILD",
 "app/src/main/java/.../versioning/FilesApiChanges.java"]
```

</div>
</div>

<Src repo="ce" path=".releaserc.json" lines="44-56" />
<Src repo="ce" path=".ci/stamp-api-changes.sh" lines="21-34" />
<Src repo="jlc" path="vars/dt3_pipeline.groovy" lines="719-731" label="stage Release: solo main e release/*" />

---

<SectionLabel label="Release" />

<div class="eyebrow">semantic-release vero, in locale, con un remote finto: nessun push, nessun tag su GitHub</div>

# Lo stamping sul <em>branch demo</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div>
  <div class="term-green code-sm">

```text
[semantic-release] Found git tag v3.0.14 associated with version 3.0.14
[semantic-release] Found 43 commits since last release
[commit-analyzer] Analyzing commit: feat(graphql)!: remove the getAcc…
[commit-analyzer] The release type for the commit is major
[semantic-release] The next release version is 4.0.0
[exec] Call script sed -i ... && bash .ci/stamp-api-changes.sh 4.0.0 ...
[git] Found 4 file(s) to commit
```

  </div>
  <div class="xs muted">43 commit: il PoC non è mergiato, quindi la release simulata li prende tutti insieme.</div>
</div>
<div class="code-sm">

```diff
# chore(release): 4.0.0, FilesApiChanges.java
-@ApiChange(since = ApiVersion.CURRENT,
+@ApiChange(since = "4.0.0",
     converter = UploadDestinationIdConverter.class)
-@ApiChange(since = ApiVersion.CURRENT,
+@ApiChange(since = "4.0.0",
     converter = UploadResponseConverter.class)
 @ApiChange(
-    since = ApiVersion.CURRENT,
+    since = "4.0.0",
     description = "POST /upload/check: ...")
# pom.xml          revision 3.0.14 → 4.0.0
# package/PKGBUILD pkgver   3.0.14 → 4.0.0
```

  <div class="xs">Da qui in poi i client <code>&lt; 4.0.0</code> passano dai converter, quelli <code>≥ 4.0.0</code> o senza header no.</div>
  <div class="note">L'import <code>ApiVersion</code> resta inutilizzato fino alla prossima riga <code>CURRENT</code> (noto, innocuo).</div>
</div>
</div>

---

<SectionLabel label="Release" />

<div class="eyebrow">Le reti di sicurezza della release</div>

# Se lo stamping <em>non avviene</em>

<div class="term-red code-sm">

```text
$ mvn -Pprod -Drevision=4.0.0 package     # come una tag build, sul commit NON stampato
API versioning: 6 problem(s):
  - the build version 4.0.0 is a release (no pre-release suffix) but the registry still has since = current:
    @ApiChange(since = ApiVersion.CURRENT, converter = ...UploadDestinationIdConverter.class) #2 of ...FilesApiChanges;
    the release stamping must rewrite it into the real version
  - ...
```

</div>

<div class="grid grid-cols-3 gap-3 mt-2 xs">
  <div class="card"><div class="head-teal">Stamper</div>Esce con 1 se resta un <code>since</code> che non è <code>"X.Y.Z"</code>: la release si ferma prima del commit.<br><Src repo="ce" path=".ci/stamp-api-changes.sh" lines="27-33" /></div>
  <div class="card"><div class="head-teal">Tag build</div>Versione senza suffisso + <code>since = current</code> = errore (sopra).<br><Src repo="ext" path="api-versioning/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/deployment/ConverterScanner.java" lines="500-515" label="checkOwnCurrent" /></div>
  <div class="card"><div class="head-teal">Tetto di versione</div><code>since</code> stampato sopra la X.Y.Z del build = errore. Copre il cherry-pick di un converter su una linea più vecchia.<br><Src repo="ext" path="api-versioning/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/deployment/ConverterScanner.java" lines="314-329" label="since sopra il build" /></div>
</div>

<div class="note mt-2">Commit stampato, stesso comando: build verde, <code>converters.json</code> con <code>"current": "4.0.0"</code> e <code>"since": "4.0.0"</code>.</div>

---

<SectionLabel label="Release" />

<div class="eyebrow">Il tipo di commit decide il numero</div>

# Che versione <em>esce</em>

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

| PR (squash) | Tipo | Da 3.0.14 |
|---|---|---|
| A · `/upload` + converter | `feat` | 3.1.0 |
| B · `/upload/check` | `feat` | 3.2.0 |
| C1 · deprecazione | `feat` | 3.3.0 |
| C2 · rimozione | `feat!` | **4.0.0** |

  <div class="xs muted mt-1">Branch unico come nella simulazione: tutto stampato 4.0.0.</div>
</div>
<div>
  <div class="card-teal card-accent xs">
    <div class="head-teal">Proposta: quando usare <code>!</code></div>
    Un cambio coperto da converter non rompe i client in finestra che mandano l'header: <code>feat</code>. <code>!</code> solo per ciò che nessun converter copre (rimozioni).
  </div>
  <div class="card-amber xs mt-3">
    <div class="head-teal" style="color:var(--amber)">Attenzione ai numeri</div>
    CE 4.0.0 ha lo stesso numero della linea Advanced 4.x. L'header non porta l'edizione: il client deve mandare il valore giusto della sua mappa <code>{ce, advanced}</code>.
  </div>
</div>
</div>

---

<SectionLabel label="Release" />

<div class="eyebrow">Advanced incorpora il jar di CE</div>

# E <em>Advanced</em>?

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```java
// AdvancedApiChanges, nella PR che alza il pin di CE
@ApiChange(since = ApiVersion.CURRENT,
    inherits = UploadDestinationIdConverter.class)
@ApiChange(since = ApiVersion.CURRENT,
    inherits = UploadResponseConverter.class)
// oppure lo ridefinisce
@ApiChange(since = ApiVersion.CURRENT,
    converter = AdvUploadResponseConverter.class,
    replaces = UploadResponseConverter.class)
final class AdvancedApiChanges {}
```

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Registry proprio, versioni proprie (4.x)</li>
    <li>Ogni converter del jar CE va legato, altrimenti il build fallisce indicando la riga mancante</li>
    <li>A runtime si confronta la versione Advanced del client con il <code>since</code> Advanced: niente tabelle di traduzione</li>
    <li>Il branch demo tocca solo CE: con il bump del pin in Advanced servirebbero le due righe <code>inherits</code></li>
  </ul>
  <div class="note">Esempio illustrativo: le righe Advanced non sono sul branch demo.</div>
</div>
</div>
