[← Ondřej Maléř · all projects](https://ondmal.github.io)

# Document Template Rendering Service

A stateless backend service that fills **Word templates** with request data and
renders them as **PDF/A-2u** documents — with loops over nested data,
conditional sections, template variables, **barcodes**, and empty **signature
fields** ready for digital signing. It succeeds a legacy templating engine: the
same templates, the same storage, the same request contract — and output that
matches the legacy engine's, template by template. In production, used by
three consuming services for four document types, with more to follow.

**Role:** solution architect — designed the service, ran the library selection
and the proof of concept, and reviewed the implementation.

**Built:** 2026.

**System of record for nothing — it is a stateless processor.** Template
maintainers own the **templates**, which they publish to shared template
storage; the consumer that asked for a document owns the **document**. The
service owns only the rendering in between, and keeps neither.

---

## The problem

Several business flows need contracts and statements as PDFs: built from Word
templates that business people maintain, filled with customer data, and — for
contracts — signed digitally afterwards. A legacy templating engine produced
them, and a whole estate of templates was built against it.

So what had to be built is a service that:

1. **Is a drop-in successor to the legacy engine** — the same template estate in
   the same storage, the same request contract, the same template language —
   richer than tag replacement: loops, conditional sections, variables,
   barcodes — and output a template maintainer cannot tell apart, so a consumer
   moves over by changing an endpoint and a template moves over unchanged.
2. **Produces documents another service can sign** — archival PDF/A with empty
   signature fields wherever the template marks a signatory.
3. **Answers online requests in seconds, in parallel, holding no state.** The
   requirement is modest — a document in under three seconds at the 90th
   percentile — but the load grows as documents move over, so the service
   scales by adding replicas.

The hard part is fidelity. Rendering Word is a commercial engine's job, but no
two engines lay out a document identically: fonts, bullets, tables and page
numbers come out differently, and a contract whose clause moves to another page
is a defect. Parity with an existing template estate had to be engineered,
quirk by quirk, without letting compatibility code leak into the rendering
core.

---

## System context

Consuming services call the service through the internal API gateway, which
authenticates them. A request names a template, which the service reads from
shared template storage, where template maintainers publish it — or carries the
template itself; either way the service streams the PDF back. Documents
that need signatures then go from the consumer to the signing service — the two
services never call each other.

![System context: consuming services call the internal API gateway, which authenticates them and forwards to the template rendering service; the service reads templates from shared template storage, where template maintainers publish them, and sends telemetry to Honeycomb; consumers pass the PDFs to the signing service](images/document-template-rendering-system-context.svg)

---

## Technology

| Area | Choices |
|---|---|
| Language / platform | C# / .NET, ASP.NET Core minimal APIs |
| Style | Vertical slices over a thin layered core |
| Rendering | GemBox.Document (Word → PDF/A-2u, PDF 2.0); signature fields with PdfSharp and PdfPig |
| Persistence | None. Templates read from Azure Files through the storage SDK, cached in memory per pod |
| Hosting / runtime | Azure; Kubernetes (one stateless, autoscaled workload via Helm); chiseled container image with the font set built in |
| Security | OAuth enforced at the internal API gateway; workload identity to Azure storage; no personal data stored or logged |
| Observability | OpenTelemetry (traces, metrics, logs) exported to Honeycomb; dashboard and alerts as code |
| Testing | Unit, architecture (NetArchTest), in-process integration, container end-to-end on the shipped image; BenchmarkDotNet |

---

## Highlights

- **A rendering library chosen on evidence.** Commercial engines assessed, the
  strongest compared in a proof of concept on a real production contract — the
  output signed end to end, performance measured. → *Design decisions*
- **Parity with a legacy engine, engineered.** Same fonts, same layout rules,
  and the legacy engine's quirks isolated in a removable shim. → *Legacy
  compatibility*
- **An extensible document-feature engine.** Signature fields, barcodes, loops
  and compatibility fixes plug in as one folder each, without touching the
  engine or the rendering library. → *Design decisions*, *The generation
  pipeline*
- **Stateless and horizontally scalable.** No database, nothing kept between
  requests; capacity is a replica count. → *Design decisions*, *Runtime &
  infrastructure*
- **The shipped image is tested, not only the code.** A container end-to-end
  suite gates every release. → *Quality & delivery*

---

## Design decisions

### Shape

**Stateless, so scale is a replica count.** The need is internal and bounded,
and the design envelope was set accordingly: hundreds of documents a day at
launch, tens of thousands a day with scale-out, documents of up to a few
megabytes, the three-second target, and 99.8 % availability. Today's load — a
few hundred documents a day — sits well inside it: a typical document renders
in about a tenth of a second, the heaviest contract — with signature fields —
in under a second at the 90th percentile, documents are tens to a few hundred
kilobytes, and every production request so far has returned a document. Each
request carries everything it needs and leaves nothing behind — no database, no
stored documents, no stored requests — so pods are interchangeable and capacity
grows by adding replicas; a per-request time limit bounds each render, and
autoscaling bounds the fleet. The same shape is the data-protection story:
customer data exists only for the duration of one render and is never logged —
telemetry carries template names, sizes, durations and error codes — so data
minimisation is a property of the shape, not a policy.

**Vertical slices over a thin core.** The service owns no process and no state:
a request comes in, a document goes out. With no aggregate to protect and no
rule spanning requests, a rich domain model would be ceremony, so the
architecture follows the problem. Each endpoint is one vertical slice —
endpoint, handler, request, validator — over a thin layered core with two
boundaries that matter, the rendering library and template storage: both sit
in Infrastructure behind ports, and architecture tests fail the build if either
boundary is crossed.

**The legacy contract, not a fresh one.** The REST endpoints are shaped on the
legacy engine's — the same request envelope, field names and semantics — so a
consumer moves over by changing an endpoint, not by rewriting its integration;
an endpoint that takes the template inline was added beside it, so a new
consumer is not forced into the shared template storage. Either way the request
is size-capped and time-boxed before it reaches the renderer. The cost: the API
inherits the legacy shape, string-typed flags and a duplicate field alias
included.

### Rendering

**Buy the renderer, build the signature fields.** Rendering Word faithfully is a
product, not a side task, and the open-source options on .NET offer no direct
Word-to-PDF rendering, or come with licensing problems. Aspose, GemBox,
DevExpress and Telerik were assessed; Aspose and GemBox went into a proof of
concept on a real production contract, each as a full commercial suite and as a
hybrid with signature fields built in-house, and every output was signed end to
end by the signing service; performance and memory were measured, and the
hybrid options came out markedly faster and leaner than the full suites. The
winner: GemBox.Document for rendering, with signature fields built on the
open-source PdfSharp and PdfPig — the leanest option measured, the cheapest
licence, and no second commercial PDF library. The cost: larger files than the
closest competitor, and slightly lower fidelity on complex layouts — which is
what the compatibility work pays for (see *Legacy compatibility*).

**Features plug in; the core stays fixed.** Specialised behaviour — signature
fields, barcodes, loops, legacy fix-ups — is modelled as features rather than as
new parameters on the rendering pipeline. The engine runs every feature at fixed
points: before the template is loaded, before it is filled, and after the PDF is
rendered. Each feature takes part only where it needs to and skips itself when
the request does not ask for it. A feature is one folder that depends on the
engine's contracts, never on the rendering library — enforced by architecture
tests — so seven features run today, the library stays replaceable, and removing
a feature means deleting its folder and one registration line.

**Signature fields as the bridge to signing.** A template marks where each
signatory signs with a placeholder; after rendering, the service finds each
placeholder in the PDF and overlays an empty signature field named by the
consumer. The output is PDF/A-2u, PDF 2.0 — archival-grade and signable — and
the [Multi-Party Document Signing Service](https://ondmal.github.io/multi-party-document-signing/)
fills those fields. The two services share no API; their contract is the PDF
itself.

### Mechanics

**Templates behind a port, cached per pod — no Redis.** The legacy engine read
templates from a mounted file share. The service loads them through the storage
SDK behind a port: tests run against templates on disk, every storage call is
traced, and failures map to precise errors (storage unavailable, timed out)
rather than file-system exceptions. A caching decorator in front keeps each
pod's templates in memory with a time-to-live, so a pod keeps serving warm
templates while the share is unreachable. The original design included a shared
Redis cache; at today's volume it would add cost and an operational dependency
for little gain, and the cache is built on a two-level abstraction, so adding it
later is a registration change, not a redesign. The cost: each pod warms its
own cache, and clearing it means calling every pod.

**Authentication at the gateway.** The service runs no authentication of its
own: it is reachable only through the internal API gateway, which authenticates
every calling service with OAuth. The cost: the service trusts its perimeter — a
route that bypassed the gateway would bypass authentication — and authorisation
is coarse: any admitted consumer may call any endpoint. Acceptable for an
internal service whose callers are a handful of known systems and whose only
data is the templates they already own.

---

## Component view

![Component view: consuming services call the WebApi host through the internal API gateway; the WebApi host is the composition root with one vertical slice per endpoint over Application and Infrastructure; Application holds ports, handlers, a thin domain and the document features, and references no rendering library; Infrastructure implements the ports with template loaders, a per-pod cache, the GemBox rendering engine, signature fields and a barcode renderer; it reads shared template storage, and the service exports telemetry to Honeycomb](images/document-template-rendering-component-view.svg)

**One host, one slice per endpoint.** The WebApi host — also the composition
root — exposes two generation endpoints, by template name or with the template
sent inline, plus a per-pod cache clear and health probes.

**Two layers under it.** **Application** holds the ports, the slices' handlers
and validators, a thin domain of value objects, and the document features with
the engine's contracts; it references no rendering library. **Infrastructure**
implements the ports on GemBox.Document, PdfSharp and PdfPig and the storage
SDK; dependencies point to Application.

---

## The generation pipeline

![Generation pipeline: the consumer sends a template name and data; the service resolves the template to a version and a localised file, loads it from its cache or from template storage, fills in loops, tags, conditional sections and barcodes, renders PDF/A-2u, adds signature fields if requested, and streams the PDF back; a consumer can instead send the template inline, which skips resolving and loading; document features — fix-ups, loops, barcodes and signature fields — plug into loading, filling and the rendered PDF](images/document-template-rendering-pipeline.svg)

- **Resolve.** A template name leads to the template's folder, its
  configuration names the current version — unless the consumer pins one — and
  the language and country pick the localised file. Maintainers publish a new
  version next to the old one and switch the configuration; nothing is
  overwritten.
- **Load.** From the pod's cache, or from template storage on a miss.
  Alternatively, the consumer sends the template inline and both steps are
  skipped.
- **Fill.** Loops repeat per item, nested lists and objects included; tags and
  template variables — values a template version defines in its configuration —
  are replaced; conditional sections are kept or removed; and barcodes are drawn
  where the template has a placeholder picture, at its size and position.
- **Render and add signature fields.** The document is rendered as PDF/A-2u,
  PDF 2.0, and empty signature fields are added if requested. Nothing after
  loading does I/O, so the whole stretch runs synchronously.
- **Stream.** The PDF is streamed back without buffering, uncompressed — a PDF
  is compressed internally already. Any step can instead end in an RFC 7807
  problem response with a machine-readable error code and the trace id; nothing
  is kept, so a failed request is safe to retry.

---

## Legacy compatibility

- **Same fonts.** The legacy engine's font set is built into the image; a family
  missing from it falls back the way the legacy engine did, and fonts a template
  embeds are honoured. Fixes where the new renderer is simply wrong — bullet
  glyphs and boxes, page numbers wrapping in footers — apply to every document.
- **Quirks behind a flag.** Emulation of the legacy engine's layout quirks —
  table positioning, spacer cells, blanking unfilled tags — lives in one opt-in
  feature, switched on per request. It is a migration shim: once the templates
  are cleaned up, it is deleted as one unit.

---

## Runtime & infrastructure

Deployed on **Azure**, running on **Kubernetes** — one stateless workload rolled
out via Helm.

- **One workload, autoscaled.** At least two replicas, scaled out on CPU.
- **Minimal image.** A chiseled .NET image running as non-root on a read-only
  file system, with the font set built in.
- **Identity and secrets.** Workload identity to template storage — no stored
  keys; the renderer's licence key is injected from Key Vault at deployment.
- **Safe rollouts.** A pre-stop delay lets the ingress drain a pod, and the
  host's shutdown budget sits inside the grace period, so in-flight renders
  finish.

---

## Observability & operability

- **OpenTelemetry end to end** — every generation is one trace with a span per
  pipeline step and per feature, so a slow render shows where the time went;
  baggage from the caller — identifiers, not data — is copied onto every span,
  so a document lines up with the business flow that requested it.
- **Metrics** for generations (count, duration, size, errors) and for the
  template cache (hits, misses, load time), with a dashboard and alerts kept as
  code.

---

## Quality & delivery

- **Test strategy**: unit tests per layer, architecture tests that fail the
  build on a crossed boundary, and in-process integration tests over HTTP with
  templates on disk — around 650 automated tests in all.
- **Container end-to-end tests on the shipped image.** The built image runs
  under the pod's own constraints — non-root, read-only file system, limited CPU
  and memory — renders every fixture template, and is checked on content (text,
  fonts, images, signature fields) and on surviving concurrent load. The suite
  runs between build and push, so a failure stops the release. It caught what
  every in-process test missed: a native library the barcode renderer needed was
  absent from the minimal image, and the process crashed.
- **Benchmarks** (BenchmarkDotNet) for the rendering path, to compare changes.
- **Fully documented**: an architecture overview with recorded decisions and
  known technical debt, a contract specification, the feature-engine design, a
  **template authoring guide** for template maintainers, and operational docs
  (deployment, observability, test strategy).
