---
layout: center
class: section
---

# Demo: tre cambi reali

---

<SectionLabel label="Demo" />

<div class="eyebrow">Branch versioning-poc-test-impl su files-ce, partito da poc-files-versioning</div>

# Modifica, aggiunta, <em>rimozione</em>

| | Cambio | Tipo | Registry | Commit |
|---|---|---|---|---|
| **A** | `POST /upload`: `ParentId` opzionale → `DestinationId` obbligatorio; risposta `{nodeId}` → `{node_id, version}` | <span class="pill pill-red">incompatibile</span> | 2 converter | <Src repo="commit" path="1b97267a637980e40e2f7e82985f4850af2775ab" /> |
| **B** | `POST /upload/check`: pre-flight di un upload | <span class="pill pill-teal">additivo</span> | 1 riga description | <Src repo="commit" path="3f9c2535c13afc46c8fd52c4d85700c9007c7703" /> |
| **C1** | query GraphQL `getAccountByEmail` deprecata | <span class="pill pill-amber">deprecazione</span> | 1 riga description | <Src repo="commit" path="530b96daae9571c8e21d7e48a2bcb013dcb8a6af" /> |
| **C2** | `getAccountByEmail` rimossa | <span class="pill pill-red">breaking</span> | 1 riga description | <Src repo="commit" path="9d8a0da342905ca36cb7acd9bf1bab0968f9b99c" /> |
| **D** | `POST /upload-version`: `version` sempre presente, anche quando vale 1 | <span class="pill pill-red">incompatibile</span> | 1 converter di valore | <Src repo="commit" path="a1da2eff79db4866e679d45f6cfd43ec235cac1b" /> |

<div class="grid grid-cols-3 gap-3 mt-4">
  <div class="stat"><div class="stat-num">659</div><div class="stat-label">IT verdi in locale (<code>mvn clean verify</code>)</div></div>
  <div class="stat"><div class="stat-num">11</div><div class="stat-label">blocchi reali incontrati e catturati</div></div>
  <div class="stat"><div class="stat-num">0</div><div class="stat-label">release fatte: branch di test, mai mergiato</div></div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow">Il cambio incompatibile, firma in ingresso e in uscita</div>

# A · POST /upload prima e <em>dopo</em>

<div class="grid grid-cols-2 gap-6 mt-1">
<div>
  <div class="head-red">Prima</div>

```http
POST /upload
Filename: cmVwb3J0LnBkZg==
ParentId: 7d1f…        # opzionale: se manca, Home

200 {"nodeId":"9a2c…"} # "version" solo se > 1
```

  <Src repo="poc" path="app/src/main/java/com/zextras/carbonio/files/rest/resources/BlobResource.java" lines="258" label="default silenzioso a LOCAL_ROOT" />
  <Src repo="poc" path="app/src/main/java/com/zextras/carbonio/files/rest/types/UploadVersionResponse.java" lines="23-29" label="TODO: serve un breaking change" />
</div>
<div>
  <div class="head-teal">Dopo</div>

```http
POST /upload
Filename: cmVwb3J0LnBkZg==
DestinationId: 7d1f…   # obbligatorio, 400 se manca

200 {"node_id":"9a2c…","version":1}
```

  <ul class="tight xs" style="padding-left:1.1em">
    <li>Il TODO c'è davvero nel codice: "we need to make a breaking change if we want to return always the version"</li>
    <li>files-ui 3.2.3 manda <code>ParentId</code> e legge <code>nodeId</code>: senza converter si rompe</li>
  </ul>
  <Src repo="ui" path="src/carbonio-files-ui-common/utils/uploadUtils.ts" lines="461" />
  <Src repo="ui" path="src/carbonio-files-ui-common/utils/uploadUtils.ts" lines="392" />
</div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">1</span>Cambio la resource, il build rigenera i docs</div>

# Il dominio parla <em>solo</em> il contratto nuovo

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="code-sm">

```java
public Uni<Response> upload(
    @HeaderParam(Headers.UPLOAD_FILENAME)
        @Parameter(required = true) String encodedFilename,
    @HeaderParam(Headers.UPLOAD_DESTINATION_ID)
        @Parameter(required = true) String destinationId,
    ...)

if (destinationId == null || destinationId.isBlank()) {
  throw new IllegalArgumentException("Missing DestinationId header");
}
...
return Response.ok(new UploadResponse(nodeId.get(), 1))
    .type(MediaType.APPLICATION_JSON)
    .build();
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/rest/resources/BlobResource.java" lines="148-156" />
<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/rest/resources/BlobResource.java" lines="295-331" />

</div>
<div class="code-sm">

```diff
# app/docs/openapi.yaml (generato, committato)
-      - required: false
-        name: ParentId
+      - required: true
+        name: DestinationId
-                $ref: "#/components/schemas/UploadVersionResponse"
+                $ref: "#/components/schemas/UploadResponse"
+    UploadResponse:
+      required: [node_id, version]
```

<Src repo="ce" path="app/docs/openapi.yaml" lines="1566-1632" />
<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/rest/types/UploadResponse.java" lines="10-14" />

  <div class="note mt-2">I docs li rigenera il build (hook pre-commit <code>maven</code>). Se li dimentichi, in CI fallisce "Verify Build Outputs".</div>
</div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">2</span>Provo a committare senza riga nel registry</div>

# Il guard se ne <em>accorge</em>

<div class="term-red">

```text
$ git commit            # hook pre-commit "api-guard" (.ci/api-guard.sh --staged)
[api-guard] Comparing contract files against 2a8f2c74ccce (merge-base with origin/poc-files-versioning)
[api-guard] FATAL: contract files changed but the registry has no new @ApiChange entry with since = CURRENT for this version:
[api-guard]   the base already holds 1 unstamped CURRENT entries of other changes: add your own
[api-guard]   changed contract file: app/docs/openapi.json
[api-guard] Fix: add one line to the registry, for example
[api-guard]   @ApiChange(since = ApiVersion.CURRENT, description = "what changed in the API")
[api-guard]   (or converter = <YourConverter>.class when old clients need their values converted)
```

</div>

<div class="grid grid-cols-2 gap-4 mt-2 xs">
  <div class="card"><strong>Regola:</strong> se <code>schema.graphql</code> o <code>openapi.json</code> cambiano rispetto alla base, il registry deve avere <strong>più</strong> righe <code>since = CURRENT</code> della base. Stessa regola in Jenkins (stage "API Guard").</div>
  <div class="card-amber">Branch impilato sul PoC: in automatico la base è <code>main</code> e la riga del PoC basta a passare. Per la demo ho passato <code>--base-branch poc-files-versioning</code>.</div>
</div>

<Src repo="ce" path=".ci/api-guard.sh" lines="200-210" label="api-guard.sh count_current" />
<Src repo="ce" path=".pre-commit-config.yaml" lines="106-114" />
<Src repo="jlc" path="vars/dt3_pipeline.groovy" lines="286" label="dt3_pipeline stage API Guard" />

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">3</span>Converter di richiesta</div>

# Il client vecchio manda <em>ParentId</em>

<div class="grid grid-cols-2 gap-6 mt-1">
<div>

```java
@ApiConverter(operation =
    "com.zextras.carbonio.files.rest.resources.BlobResource#upload")
public class UploadDestinationIdConverter
    implements RestRequestConverter {

  @Override
  public void convert(RestRequest request) {
    String parentId = request.header(Headers.UPLOAD_PARENT_ID)
        .orElse(RootId.LOCAL_ROOT);
    request.removeHeader(Headers.UPLOAD_PARENT_ID);
    request.setHeader(Headers.UPLOAD_DESTINATION_ID, parentId);
  }
}
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/UploadDestinationIdConverter.java" lines="17-25" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Gira <strong>prima</strong> del metodo, solo per client con versione &lt; <code>since</code></li>
    <li>Riproduce anche il vecchio default: niente <code>ParentId</code> → <code>LOCAL_ROOT</code></li>
    <li><code>operation</code> è verificata al build sul modello di Quarkus REST: un refuso fa fallire il build</li>
    <li>Vede solo header e query param: un cambio del <strong>body JSON</strong> di una richiesta REST non è convertibile</li>
  </ul>
</div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">4</span>Converter di risposta</div>

# Il client vecchio legge <em>nodeId</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="code-sm">

```java
@ApiConverter
public class UploadResponseConverter
    implements ValueConverter<UploadResult> {

  @Override
  public UploadResult convert(UploadResult value) {
    return value instanceof UploadResponse current
        ? LegacyUploadResponse.of(current)
        : value;
  }
}

@JsonInclude(Include.NON_NULL)
public record LegacyUploadResponse(String nodeId, Integer version)
    implements UploadResult {
  static LegacyUploadResponse of(UploadResponse current) {
    return new LegacyUploadResponse(
        current.nodeId(), current.version() > 1 ? current.version() : null);
  }
}
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/UploadResponseConverter.java" lines="13-20" />
<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/LegacyUploadResponse.java" lines="15-22" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li><code>ValueConverter&lt;T&gt;</code> restituisce un <code>T</code>: per cambiare <strong>forma</strong> serve un supertipo comune (<code>UploadResult</code>) e un tipo legacy</li>
    <li>Il serializer REST usa il serializer della classe restituita, quindi esce il JSON vecchio</li>
    <li>Il tipo legacy vive accanto al converter: a fine finestra si cancellano insieme</li>
    <li>In GraphQL la forma la sceglie il client con la selezione: lì il value converter cambia valori, non forme</li>
  </ul>
  <Src repo="ext" path="api-versioning/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/integration/rest/ConvertingSerializer.java" lines="106-117" label="ConvertingSerializer.serialize" />
</div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">5</span>Dimentico di registrare i converter</div>

# Il build <em>non parte</em>

<div class="term-red code-sm">

```text
$ mvn package
[error]: Build step ...ApiVersioningProcessor#model threw an exception: API versioning: 2 problem(s):
  - converter com.zextras.carbonio.files.versioning.UploadDestinationIdConverter is not bound by the registry of the
    application: add to com.zextras.carbonio.files.versioning.FilesApiChanges the line
    @ApiChange(since = ApiVersion.CURRENT, converter = com.zextras.carbonio.files.versioning.UploadDestinationIdConverter.class)
  - converter com.zextras.carbonio.files.versioning.UploadResponseConverter is not bound by the registry of the application: ...
```

</div>

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="term-green">

```java
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadDestinationIdConverter.class)
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadResponseConverter.class)
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" lines="15-16" />
</div>
<div class="xs">
  <div class="head-teal">Altre regole dello stesso build step</div>
  <ul class="tight" style="margin:0;padding-left:1.1em">
    <li>ogni converter legato da <strong>esattamente</strong> una riga</li>
    <li><code>operation</code> deve esistere</li>
    <li><code>since</code> stampato sopra la versione del build: errore</li>
    <li>build di release con <code>since = current</code>: errore</li>
  </ul>
  <Src repo="ext" path="api-versioning/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/deployment/ConverterScanner.java" lines="443-450" label="ConverterScanner: not bound" />
</div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">6</span>Test versionati</div>

# Il test <em>promette</em> che il converter gira

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="code-sm">

```java
@Test
@ApiVersionBefore(UploadResponseConverter.class)
void anOldClientReadsNodeIdWithoutTheVersion() {
  uploadAs(ApiVersionTestContext.version(), "old-shape.txt")
      .header("ParentId", LOCAL_ROOT)
      .body(CONTENT)
      .post("/upload")
      .then()
      .statusCode(200)
      .body("nodeId", notNullValue())
      .body("$", not(hasKey("node_id")))
      .body("$", not(hasKey("version")));
}
```

<Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/UploadApiVersioningIT.java" lines="117-130" />

  <ul class="tight xs" style="padding-left:1.1em">
    <li>Il kit manda la <strong>oldest-supported</strong> (3.0.14), mai versioni scritte a mano</li>
    <li>Il test fallisce se il converter non ha cambiato niente</li>
  </ul>
</div>
<div>
  <div class="head-red">Trappola vera: risposta serializzata a mano</div>
  <div class="xs muted">Files scrive spesso il JSON con un <code>ObjectMapper</code> statico. Così il value converter non gira mai:</div>
  <div class="term-red code-sm">

```text
@ApiVersionBefore(...UploadResponseConverter) promises that this
test exercises the converter, but ...UploadResponseConverter was
not applied during it (X-FILES-API-VERSION sent: [3.0.14];
converters that did change something:
[...UploadDestinationIdConverter])
```

  </div>
  <div class="xs">Soluzione: restituire l'<strong>entity</strong> e lasciare serializzare Quarkus REST. Dopo la demo l'abbiamo fatto nel PoC per <code>/upload</code>, <code>/upload-version</code> e <code>/upload-to</code>: il JSON resta identico.</div>
  <Src repo="commit" path="653fa5fb34fcbd4933fbbcf1f3766f29a3baa829" label="refactor(rest): entity" />
  <Src repo="ext" path="api-versioning/test/src/main/java/com/zextras/carbonio/quarkus/extensions/apiversioning/test/ApiVersionExtension.java" lines="66-100" label="kit: afterEach" />
</div>
</div>

---

<SectionLabel label="Demo · A" />

<div class="eyebrow"><span class="step">7</span>Copertura e suite esistente</div>

# I test vecchi sono <em>i client di oggi</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div>
  <div class="head-red">Converter senza test</div>
  <div class="term-red code-sm">

```text
ApiVersioningCoverageIT: API versioning: active converters named
by no @ApiVersionBefore in .../app/target/test-classes (add a
test annotated @ApiVersionBefore(<converter>.class) for each):
[com.zextras.carbonio.files.versioning.UploadResponseConverter]
```

  </div>
  <Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/ApiVersioningCoverageIT.java" lines="9" />
</div>
<div>
  <div class="head-red">Suite esistente dopo il cambio</div>
  <ul class="tight" style="margin:0;padding-left:1.1em">
    <li><strong>9 IT rossi</strong> (<code>UploadFileApiIT</code> ×5, <code>StreamedUploadApiIT</code> ×4) e 1 unit test</li>
    <li><strong>3 test verdi per il motivo sbagliato</strong>: i test sul <code>Filename</code> davano 400… perché mancava <code>DestinationId</code></li>
    <li>Aggiornati e dichiarati nel commit come <strong>cambio di specifica</strong>, mai indeboliti</li>
  </ul>
  <Src repo="commit" path="1b97267a637980e40e2f7e82985f4850af2775ab" label="messaggio del commit A" />
</div>
</div>

---

<SectionLabel label="Demo · B" />

<div class="eyebrow">Cambio additivo</div>

# B · nuovo endpoint <em>POST /upload/check</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="code-sm">

```java
@POST
@Path("/upload/check")
@Blocking
@APIResponses({
  @APIResponse(responseCode = "204", description = "..."),
  @APIResponse(responseCode = "400", description = "..."),
  @APIResponse(responseCode = "404", description = "..."),
  @APIResponse(responseCode = "413", description = "...")
})
public Response checkUpload(
    ...
    @HeaderParam(Headers.UPLOAD_DESTINATION_ID)
        @Parameter(required = true) String destinationId,
    @HeaderParam(Headers.UPLOAD_FILE_SIZE)
        @Parameter(required = true) Long fileSize) {
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/rest/resources/BlobResource.java" lines="250-293" />
<Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/UploadCheckApiIT.java" lines="47-92" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Chiede prima dei byte: permesso sulla cartella, dimensione, nome</li>
    <li>Additivo: nessun client vecchio lo conosce, niente da convertire</li>
    <li>I docs cambiano, quindi serve comunque una riga:</li>
  </ul>
  <div class="term-green">

```java
@ApiChange(
    since = ApiVersion.CURRENT,
    description = "POST /upload/check: pre-flight of an upload ...")
```

  </div>
  <Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" lines="17-19" />
  <div class="note">Test normali (versione corrente): una riga senza converter non chiede <code>@ApiVersionBefore</code>.</div>
</div>
</div>

---

<SectionLabel label="Demo · B" />

<div class="eyebrow">Il contratto deve essere completo</div>

# Quello che ho <em>dimenticato</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div>
  <div class="head-red">Al build: niente dichiarazioni</div>
  <div class="term-red code-sm">

```text
quarkus.carbonio-rest.require-declared-responses is on and 1
application resource method(s) declare no method-level 1xx-3xx
status (class-level statuses do not count):
...BlobResource#checkUpload

quarkus.carbonio-rest.require-explicit-parameters is on and 1
header/query/cookie/form parameter(s) state neither
@Parameter(required = true|false), nor @NotNull/@NotBlank/
@NotEmpty, nor @DefaultValue: ...BlobResource#checkUpload FileSize
```

  </div>
  <Src repo="ext" path="rest/deployment/src/main/java/com/zextras/carbonio/quarkus/extensions/deployment/ResponsePlanner.java" lines="38" label="ResponsePlanner" />
</div>
<div>
  <div class="head-red">Negli IT: il 413 non dichiarato</div>
  <div class="term-red code-sm">

```text
ERROR [DeclaredResponseFilter] Undeclared response status 413
written by operation ...BlobResource#checkUpload
(declared: [400, 401, 500, 204, 403, 503, 404]);
replaced with a 500

AssertionFailedError: undeclared REST response for
POST .../upload/check: X-Undeclared-Response: status=413
```

  </div>
  <div class="xs">Strict mode accesa solo negli IT. Perché conta: <strong>docs cambiati = contratto cambiato</strong> regge solo se i docs dicono tutto.</div>
  <Src repo="ext" path="rest/runtime/src/main/java/com/zextras/carbonio/quarkus/extensions/rest/DeclaredResponseFilter.java" lines="114" label="DeclaredResponseFilter" />
</div>
</div>

---

<SectionLabel label="Demo · C" />

<div class="eyebrow">Togliere un'operazione: prima si depreca</div>

# C1 · deprecare <em>getAccountByEmail</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="code-sm">

```java
@Deprecated
@Description("Deprecated: use getAccountsByEmail(emails: [email])")
@Query("getAccountByEmail")
@GraphqlErrors({UNAUTHENTICATED, ACCOUNT_NOT_FOUND})
public Account getAccountByEmail(@Name("email") @NonNull String email)
```

```diff
# app/docs/schema.graphql
-  getAccountByEmail(email: String!): Account @errors(...)
+  "Deprecated: use getAccountsByEmail(emails: [email])"
+  getAccountByEmail(email: String!): Account @deprecated @errors(...)
```

<Src repo="commit" path="530b96daae9571c8e21d7e48a2bcb013dcb8a6af" label="commit C1 (diff completo)" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li><code>getAccountsByEmail(emails: [email])</code> fa la stessa cosa</li>
    <li>La query continua a funzionare: cambia solo lo schema</li>
    <li>Una deprecazione cambia i docs, quindi serve una riga description</li>
    <li>codegen e linter dei client segnalano gli usi <code>@deprecated</code></li>
  </ul>
  <div class="term-green">

```java
@ApiChange(
    since = ApiVersion.CURRENT,
    description =
        "Deprecate the getAccountByEmail query:"
            + " use getAccountsByEmail")
```

  </div>
  <Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" lines="20-22" />
</div>
</div>

---

<SectionLabel label="Demo · C" />

<div class="eyebrow">La rimozione non si converte</div>

# C2 · rimuovere <em>getAccountByEmail</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div>
  <div class="head-red">Cosa vede un client che la usa ancora</div>
  <div class="term-red code-sm">

```text
Validation error (FieldUndefined@[getAccountByEmail]) :
Field 'getAccountByEmail' in type 'Query' is undefined
```

  </div>
  <ul class="tight xs" style="padding-left:1.1em">
    <li>Con qualunque <code>X-FILES-API-VERSION</code>: la validazione GraphQL viene prima dei converter</li>
    <li>files-ui 3.2.3 la usa ancora in <code>AddSharing.tsx</code></li>
    <li>Sono gli IT vecchi di <code>UserResolversApiIT</code>, prima di aggiornarli</li>
  </ul>
  <Src repo="ui" path="src/carbonio-files-ui-common/views/components/sharing/collaborators/AddSharing.tsx" lines="154" />
</div>
<div>
  <div class="term-green code-sm">

```java
@ApiChange(
    since = ApiVersion.CURRENT,
    description =
        "Remove the getAccountByEmail query (deprecated):"
            + " clients that still call it get a GraphQL"
            + " validation error, no converter can bring"
            + " an operation back")
```

  </div>
  <Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/FilesApiChanges.java" lines="23-27" />

```text
feat(graphql)!: remove the getAccountByEmail query
...
BREAKING CHANGE: the getAccountByEmail GraphQL query no longer
exists; use getAccountsByEmail(emails: [email]).
```

  <div class="card-amber xs">Processo giusto: depreca → i client migrano → la finestra supera la release della deprecazione → rimuovi. <strong>Nessun check sa se i client hanno smesso di usarla.</strong></div>
</div>
</div>

---

<SectionLabel label="Demo · D" />

<div class="eyebrow">Il TODO di /upload-version: stesso tipo, cambia solo un valore</div>

# D · un converter <em>di valore</em>

<div class="grid grid-cols-2 gap-5 mt-1">
<div class="code-sm">

```diff
 public void setVersion(int version) {
-  // TODO: ... we need to make a breaking change
-  //  if we want to return always the version
-  if (version > 1) {
-    this.version = version;
-  }
+  this.version = version;
 }
```

```java
@ApiConverter
public class UploadVersionNumberConverter
    implements ValueConverter<UploadVersionResponse> {

  @Override
  public UploadVersionResponse convert(UploadVersionResponse value) {
    if (!Integer.valueOf(1).equals(value.getVersion())) {
      return value;
    }
    UploadVersionResponse legacy = new UploadVersionResponse();
    legacy.setNodeId(value.getNodeId());
    return legacy;
  }
}
```

<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/versioning/UploadVersionNumberConverter.java" lines="12-24" />
<Src repo="ce" path="app/src/main/java/com/zextras/carbonio/files/rest/types/UploadVersionResponse.java" lines="17-27" />

</div>
<div>
  <ul class="tight" style="padding-left:1.1em">
    <li>Overwrite della versione 1: prima la risposta non aveva <code>version</code>, adesso <code>{"nodeId":…,"version":1}</code></li>
    <li>La forma non cambia, quindi <strong>niente tipo legacy</strong>: si toglie solo il valore (con <code>NON_NULL</code> il campo sparisce)</li>
    <li>Funziona solo perché nel PoC <code>/upload-version</code> ora restituisce l'entity</li>
    <li>Il tipo è condiviso con l'upload interno, serializzato a mano: lì non si converte, ma i backend sono sempre current</li>
    <li>Docs: <code>version</code> diventa <code>required</code>, quindi serve la riga nel registry</li>
  </ul>
  <div class="term-green code-sm">

```java
@ApiChange(since = ApiVersion.CURRENT,
    converter = UploadVersionNumberConverter.class)
```

  </div>
  <Src repo="ce" path="app/src/test/java/com/zextras/carbonio/files/it/UploadVersionApiVersioningIT.java" lines="51-74" />
</div>
</div>
