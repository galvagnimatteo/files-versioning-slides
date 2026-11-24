---
layout: center
class: section
---

# Il modello

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Che numero mandiamo</div>

# La versione API è la <em>versione del pacchetto</em>

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```http
POST /graphql
X-FILES-API-VERSION: 3.0.14

HTTP/1.1 200 OK
X-FILES-API-VERSION: 3.0.14
Vary: X-FILES-API-VERSION
```

```http
GET /versions          # pubblico
{"name":"carbonio-files","edition":"ce",
 "current":"3.0.15-187.abc1234-SNAPSHOT",
 "oldestSupported":"3.0.14",
 "header":"X-FILES-API-VERSION"}
```

<Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/ApiVersioningApiIT.java" lines="68-170" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Una sola semver, quella che <strong>semantic-release</strong> decide per l'edizione: CE <code>3.x</code>, Advanced <code>4.x</code></li>
    <li>Niente contatore separato, niente prefisso di edizione nel valore</li>
    <li>Il client tiene una mappa <code>{ce, advanced}</code> e manda il valore dell'edizione che rileva (lo fanno già files-ui e files-mobile)</li>
    <li>Il numero non ha un significato forte: si muove a ogni merge, anche per un bump di Renovate (accettato)</li>
    <li>La risposta ripete la versione servita</li>
  </ul>
</div>
</div>

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Negoziazione, prima dell'autenticazione</div>

# Cosa succede a <em>ogni richiesta</em>

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```text
negotiate(X-FILES-API-VERSION):
  assente | "" | "current"           → current
  non semver | > 128 char | 2 valori → 400 INVALID_API_VERSION
  == versione del build              → current
  X.Y.Z > X.Y.Z del build            → current (client più nuovo)
  < oldest-supported                 → 400 UNSUPPORTED_API_VERSION
  altrimenti                         → servito a quella versione

converter applicato ⇔ attivo
                    ∧ client ≠ current
                    ∧ since > versione del client
```

<Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/internal/VersionPolicy.java" lines="94-109" label="VersionPolicy.negotiateToken" />
<Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/internal/ApiVersioningModel.java" lines="107-115" label="ApiVersioningModel.applies" />

</div>
<div>
  <div class="card small">
    <div class="head-teal">Il 400</div>

```json
{"code":"UNSUPPORTED_API_VERSION",
 "received":"3.0.10",
 "oldestSupported":"3.0.14",
 "current":"3.0.15-187.abc1234-SNAPSHOT"}
```

  <div class="xs muted">Anche su <code>/graphql</code> è un HTTP 400, non un <code>errors</code> GraphQL: i client Apollo lo vedono come errore di rete.</div>
  </div>
  <div class="note mt-2">Pseudocodice: il repo dell'estensione è privato, il link porta alla riga vera.</div>
</div>
</div>

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Il valore speciale</div>

# Come funziona <em>current</em>

<div class="grid grid-cols-2 gap-6 mt-2">
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li><code>ApiVersion.CURRENT = "current"</code>, ordinato <strong>sopra ogni semver</strong></li>
    <li>In una PR si scrive <code>since = ApiVersion.CURRENT</code>: la versione vera esiste solo alla release</li>
    <li>Sui build di sviluppo un converter <code>current</code> si applica a <strong>ogni</strong> client che manda una versione concreta: i test lo vedono subito</li>
    <li>Alla release lo stamper lo riscrive in <code>"X.Y.Z"</code>, per sempre</li>
  </ul>
  <Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/ApiVersion.java" lines="20-23" label="ApiVersion.CURRENT" />
  <Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/ApiVersion.java" lines="184-202" label="ApiVersion.compareTo" />
</div>
<div>

| Build | Versione | Per la negoziazione |
|---|---|---|
| locale | `3.0.14-SNAPSHOT` | X.Y.Z = 3.0.14 |
| CI branch | `3.0.15-187.abc1234-SNAPSHOT` | X.Y.Z = 3.0.15 |
| tag | `3.1.0` | release |

  <div class="card-amber mt-3 xs">
    Perché non semver puro: <code>3.0.14-SNAPSHOT &lt; 3.0.14</code>, quindi un client 3.0.14 risulterebbe "più nuovo" del build locale e salterebbe i converter. Regola: è "sopra" solo chi supera la X.Y.Z del build.
  </div>
</div>
</div>

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Un file per servizio</div>

# Il <em>registry</em> dei cambi

<div class="grid grid-cols-2 gap-6 mt-1">
<div class="code-sm">

```java
@ApiChange(
    since = ApiVersion.CURRENT,
    description = "Introduce API versioning: ...")
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadDestinationIdConverter.class)
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadResponseConverter.class)
@ApiChange(
    since = ApiVersion.CURRENT,
    description = "POST /upload/check: pre-flight ...")
@ApiChange(
    since = ApiVersion.CURRENT,
    description = "Deprecate the getAccountByEmail query ...")
@ApiChange(
    since = ApiVersion.CURRENT,
    description = "Remove the getAccountByEmail query ...")
final class FilesApiChanges {}
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" lines="10-29" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Una riga per cambio di API: è il <strong>changelog del contratto</strong></li>
    <li><code>since</code> vive solo qui, i converter non hanno versione</li>
    <li>Tre forme:
      <ul>
        <li><code>converter = X.class</code>: cambio incompatibile</li>
        <li><code>description = "..."</code>: additivo, deprecazione, rimozione</li>
        <li><code>inherits</code> / <code>replaces</code>: Advanced che adotta o ridefinisce un converter CE</li>
      </ul>
    </li>
    <li>Queste sono le righe del branch demo: le vediamo nascere una per una</li>
  </ul>
</div>
</div>

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Cosa può convertire un converter</div>

# Tre tipi di <em>converter</em>

| Tipo | Interfaccia | Quando gira | Può | Non può |
|---|---|---|---|---|
| Richiesta REST | `RestRequestConverter` | prima del metodo, `operation = "Classe#metodo"` | header, query param | **body** della richiesta |
| Argomenti GraphQL | `GraphqlArgumentConverter` | prima del data fetcher, `operation = root field` | valori degli argomenti | argomenti assenti dallo schema corrente (la validazione viene prima) |
| Valore | `ValueConverter<T>` | quando un `T` viene serializzato (REST Jackson e campi GraphQL) | cambiare valori, restituire un sottotipo con un'altra forma (REST) | status code, codici di errore |
| Vuoto | riga con `description` | mai | documentare | — |

<div class="grid grid-cols-3 gap-3 mt-3 xs">
  <div class="card">Il converter diventa un bean CDI da solo: basta <code>@ApiConverter</code>.</div>
  <div class="card">Catena per un client V: tutti i converter attivi con <code>since &gt; V</code>, in ordine di <code>since</code>.</div>
  <div class="card-amber">Una rimozione non si converte: un'operazione sparita non torna.</div>
</div>

<Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/ValueConverter.java" lines="17-21" label="ValueConverter" />
<Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/RestRequestConverter.java" lines="13-16" label="RestRequestConverter" />

---

<SectionLabel label="Il modello" />

<div class="eyebrow">Configurazione e finestra</div>

# Quanto <em>indietro</em> supportiamo

<div class="grid grid-cols-2 gap-6 mt-2">
<div>

```properties
quarkus.carbonio-api-versioning.header=X-FILES-API-VERSION
quarkus.carbonio-api-versioning.oldest-supported=3.0.14
quarkus.carbonio-api-versioning.name=carbonio-files
quarkus.carbonio-api-versioning.edition=ce
quarkus.carbonio-api-versioning.versions-path=/versions
```

<Src repo="ce" path="app/src/main/resources/application.properties" lines="193-200" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li><strong>15 mesi</strong>, misurati sui tag di main</li>
    <li><code>oldest-supported</code> si cambia <strong>a mano</strong>, in una PR revisionata</li>
    <li>La CI di main avvisa quando un tag supportato supera i 15 mesi: solo warning</li>
    <li>Converter con <code>since ≤ oldest-supported</code>: disattivati, il codice resta, il build lo segnala. Cancellarne uno ancora attivo lo blocca il guard</li>
    <li>La finestra parte dalla prima release con il modulo: i client di prima non mandavano l'header</li>
  </ul>
</div>
</div>
