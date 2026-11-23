<SectionLabel label="Agenda" />

<div class="eyebrow">Di cosa parliamo</div>

# Agenda

<div class="grid grid-cols-4 gap-3 mt-6">
  <div class="card"><div class="card-num">01</div><div class="font-bold" style="color:var(--ink)">Il problema</div><div class="xs muted">backward e forward compatibility</div></div>
  <div class="card"><div class="card-num">02</div><div class="font-bold" style="color:var(--ink)">Il modello</div><div class="xs muted">numero, header, current, registry</div></div>
  <div class="card"><div class="card-num">03</div><div class="font-bold" style="color:var(--ink)">Alternative</div><div class="xs muted">cosa abbiamo scartato e perché</div></div>
  <div class="card-teal card-accent"><div class="card-num">04</div><div class="font-bold" style="color:var(--ink)">Demo reale</div><div class="xs muted">3 cambi su files-ce, passo per passo</div></div>
  <div class="card"><div class="card-num">05</div><div class="font-bold" style="color:var(--ink)">Guardrail</div><div class="xs muted">cosa ti ferma, cosa no</div></div>
  <div class="card"><div class="card-num">06</div><div class="font-bold" style="color:var(--ink)">Release</div><div class="xs muted">semantic-release e stamping</div></div>
  <div class="card"><div class="card-num">07</div><div class="font-bold" style="color:var(--ink)">Client</div><div class="xs muted">con e senza header</div></div>
  <div class="card"><div class="card-num">08</div><div class="font-bold" style="color:var(--ink)">Stato</div><div class="xs muted">invecchiamento, costi, prossimi passi</div></div>
</div>

---
layout: center
class: section
---

# Il problema

---

<SectionLabel label="Il problema" />

<div class="eyebrow">Files oggi</div>

# Ogni cambio di contratto <em>rompe qualcuno</em>

<div class="grid grid-cols-2 gap-6 mt-4">
  <div class="card-red">
    <div class="head-red">Oggi</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Nessun header di versione, nessuna negoziazione</li>
      <li>Il server serve <strong>un solo</strong> contratto: l'ultimo</li>
      <li>Client in giro: files-ui, files-mobile, public-folder-ui, link pubblici</li>
      <li>Ottobre 2026: il passaggio a GraphQL code-first rompe files-mobile, che reagisce con un retry "a tentativi"</li>
      <li>Il refactor delle preview è fermo in attesa del versioning</li>
    </ul>
  </div>
  <div class="card-teal card-accent">
    <div class="head-teal">Obiettivo</div>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Il dominio implementa <strong>solo il comportamento corrente</strong></li>
      <li>Un piccolo <strong>converter</strong> per ogni cambio incompatibile traduce per i client vecchi</li>
      <li>Finestra di supporto: <strong>15 mesi</strong></li>
      <li>Il build e la CI ti ricordano cosa manca</li>
    </ul>
  </div>
</div>

---

<SectionLabel label="Il problema" />

<div class="eyebrow">Due direzioni, due responsabili</div>

# Backward e <em>forward</em> compatibility

<div class="grid grid-cols-2 gap-6 mt-3">
  <div class="card-teal card-accent">
    <div class="card-num">Backward · client vecchio → server nuovo</div>
    <h3>Lo fa il server</h3>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Il client dichiara la versione con cui è stato costruito</li>
      <li>Il server applica i converter con <code>since</code> maggiore di quella versione</li>
      <li>Upgrade della richiesta, downgrade della risposta</li>
    </ul>
  </div>
  <div class="card">
    <div class="card-num">Forward · client nuovo → server vecchio</div>
    <h3>Lo fa il client</h3>
    <ul class="tight" style="margin:0;padding-left:1.1em">
      <li>Un server vecchio non può conoscere il futuro</li>
      <li>Versione sopra la sua: risponde col <strong>suo</strong> contratto e lo dichiara nell'header di risposta</li>
      <li>Il client legge l'echo (o <code>GET /versions</code>) e si adatta</li>
    </ul>
  </div>
</div>

<div class="card text-center mt-4 small">
  Default per Files: <strong>nessun header = current</strong>. Chi non manda l'header vede il comportamento nuovo, esattamente come oggi.
</div>
