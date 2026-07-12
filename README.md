# learning-bom

<img src="image/maven-logo.png" alt="logo" width="80"/>

## Table of contents

1. 🔨 [1. The problem: dependency version sprawl across a multi-repo organization](#1-the-problem-dependency-version-sprawl-across-a-multi-repo-organization)
2. 🔨 [2. How Maven's `<dependencyManagement>` + `scope=import` mechanism actually works](#2-how-mavens-dependencymanagement--scopeimport-mechanism-actually-works)
3. 🔨 [3. How this BOM is structured, section by section](#3-how-this-bom-is-structured-section-by-section)
4. 🏷️ [4. Deliberately held-back versions — and why](#4-deliberately-held-back-versions--and-why)
5. 🔨 [5. Position in the workspace's three-tier dependency-management chain](#5-position-in-the-workspaces-three-tier-dependency-management-chain)
6. 🚀 [6. How to import (already wired — service repos should not repeat this)](#6-how-to-import-already-wired--service-repos-should-not-repeat-this)
7. 🔨 [7. How to add a new managed dependency](#7-how-to-add-a-new-managed-dependency)
8. 🔨 [8. Quick reference — all managed dependencies](#8-quick-reference--all-managed-dependencies)
9. 🔨 [9. Versioning policy for this BOM itself](#9-versioning-policy-for-this-bom-itself)

**groupId:** `com.org.learning` · **artifactId:** `learning-bom` · **packaging:** `pom` · **current version:** `1.1.0`

This repository is a Maven **Bill of Materials (BOM)**: a `pom`-packaged artifact whose entire
purpose is to declare *versions*, not code. It contains no Java sources, produces no jar, and is
never placed on anyone's classpath directly. Its only job is to be `import`-scoped into the
`dependencyManagement` of one other POM — [`super-pom`](../super-pom) — so that every leaf
service in this developer's `~/projects` workspace (`llm-text2sql`, `llm-chat`, `llm-rag`,
`llm-gateway`, `llm-mcp`, `llm-mcp-gateway`, `llm-deep-agent`, `llm-eval`, `llm-langchain4j`,
`llm-OKF`, `learning-kafka`, `learning-graphql`, `learning-axon`, `learning-shedlock`,
`learning-reactive`, `learning-utility`, `learning-wiremock`, `learning-testing-mutation`, and
others) resolves *exactly the same* version of Spring Boot, Spring Cloud, Spring AI,
Resilience4j, Testcontainers, PDFBox, POI, Tesseract, the Anthropic Java SDK, LangChain4j, Avro,
Confluent's Kafka tooling, Shedlock, and a dozen smaller libraries — without any of those repos
ever writing a `<version>` tag themselves.

This document explains, in depth, what problem a BOM solves, how Maven's import-scope mechanism
actually works under the hood, how this particular BOM is organized section by section, why
several versions in it are *deliberately* held back from latest, and how it sits in the
three-tier dependency-management chain that spans this developer's multi-repo Maven setup.

---

## 1. The problem: dependency version sprawl across a multi-repo organization

This workspace is not one monolith — it is roughly twenty independent Maven repos, each with its
own `pom.xml`, built and released independently, sharing a substantial overlapping set of
third-party libraries. Every one of them needs Spring Boot. Most need Spring AI and/or
LangChain4j. Several need Testcontainers for integration tests, Resilience4j for circuit
breaking, Shedlock for distributed scheduling, or PDFBox/POI/Tesseract for document ingestion.

Without a shared mechanism, each repo's `pom.xml` would need its own `<version>` tag for each of
these dependencies. In practice, that produces a specific, well-known set of failure modes:

- **Version drift.** `llm-chat` pins `resilience4j` 2.2.0 while `llm-gateway` pins 2.4.0. Nobody
  planned this; it just happened because each repo was bumped at a different time by whoever was
  touching it that week. Now behavior (and available APIs) differs between services that are
  supposed to be part of the same platform.
- **The "diamond dependency" problem.** Two libraries a service depends on directly or
  transitively (say, `spring-ai-starter-model-anthropic` and `langchain4j-community-redis`) may
  each transitively drag in different versions of a shared third artifact (e.g. Jackson,
  Netty, or Reactor). Maven's "nearest wins" transitive-resolution rule then picks a version
  based on POM depth and declaration order, essentially arbitrarily, unless something upstream
  pins it explicitly.
- **Upgrade fatigue.** Bumping Spring Boot from 4.0 to 4.1 across twenty repos means twenty pull
  requests, twenty rounds of "did I get the version string right," and twenty chances for a typo
  or a forgotten transitive dependency bump (e.g. remembering that Resilience4j's Spring Boot 3
  integration also needs to move in lockstep).
- **Silent incompatibility.** Some libraries are version-sensitive to their *host* framework in
  ways that are not enforced by Maven at all — they simply fail at runtime. This BOM currently
  encodes exactly one such case explicitly (Resilience4j 2.4.0 refusing to start under Spring
  Boot 4.x — see §4 below), and a BOM is precisely the place to encode and centralize that
  knowledge once, rather than rediscovering it independently in every repo that upgrades.
- **No single place to reason about "what are we actually running."** When a security advisory
  drops for, say, Apache POI, an engineer needs one file to open, not twenty.

A BOM solves all of this by inverting the responsibility: instead of every consuming repo
declaring "I want version X of library L," the BOM declares "this organization's officially
sanctioned version of library L is X," and consuming repos simply declare *that they use L*,
with no version at all. The version is resolved for them, transitively, from a single
upstream source of truth. Upgrading becomes a one-line property bump in one repo
(`learning-bom`), followed by a version bump of the BOM dependency in `super-pom`, rather than N
edits scattered across N repos.

This is precisely the same pattern Spring itself uses for `spring-boot-dependencies`, and that
Spring Cloud, Spring AI, and Testcontainers each ship as `*-bom` artifacts — `learning-bom` sits
one layer above all of those, aggregating them together with everything else this organization's
services need that isn't covered by an upstream BOM.

---

## 2. How Maven's `<dependencyManagement>` + `scope=import` mechanism actually works

Maven has two related-but-distinct concepts that are frequently confused:

### 2.1 `<dependencyManagement>` on its own

A `<dependencyManagement>` block does **not** add any dependency to a project's classpath. It is
purely a table of "if this artifact ever ends up on the classpath (declared directly by this
POM, one of its children, or pulled in transitively), and no closer declaration overrides it, use
this version / scope / exclusions." A child POM that inherits a parent's
`dependencyManagement` and then declares:

```xml
<dependency>
    <groupId>org.apache.pdfbox</groupId>
    <artifactId>pdfbox</artifactId>
</dependency>
```

with **no** `<version>` gets the version from the nearest enclosing `dependencyManagement` entry
that matches on `groupId:artifactId` (and `classifier`/`type` if present). This is standard
parent → child POM inheritance and works the same way whether the managed entry was declared
directly in the parent or arrived there via an import (see below).

### 2.2 `scope=import` — the BOM-specific piece

A plain `<dependency>` entry inside `<dependencyManagement>` can point at another `pom`-packaged
artifact and declare `<scope>import</scope>`, e.g. this BOM's own top section:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>${spring-boot.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

This tells Maven: *"go fetch `spring-boot-dependencies:4.1.0`'s own
`<dependencyManagement>` block wholesale, and splice all of its entries into mine, as if I had
typed them all out by hand."* Two properties of this mechanism matter a great deal in practice:

1. **It is additive/textual, not inherited-by-reference.** The importing POM's own
   `dependencyManagement` entries are merged with everything imported. Order matters: if two
   imported BOMs (or an import and a locally-declared entry) both manage the same
   `groupId:artifactId`, **the first declaration encountered wins** — later ones for the same
   coordinate are silently ignored. This is exactly why this BOM's platform-BOM block is
   annotated `<!-- import order matters: first declaration wins -->` and always imports
   `spring-boot-dependencies` before `spring-cloud-dependencies`, `spring-ai-bom`, and
   `testcontainers-bom` — Spring Cloud's and Spring AI's own BOMs sometimes manage a dependency
   (e.g. a Jackson or Netty version) that Spring Boot also manages, and the intent here is for
   Spring Boot's own coordinated, mutually-tested version set to take precedence.
2. **`scope=import` only works inside `<dependencyManagement>`, on a `<type>pom</type>`
   dependency.** It is a compile-time signal to Maven's model builder, not a runtime classpath
   scope like `compile` or `runtime` — the resulting effective POM behaves as though those
   `<dependencyManagement>` entries were pasted in directly.

### 2.3 Why this matters for a 3-tier chain

Because import-scope is resolved when Maven builds the **effective POM**, and because normal
parent/child inheritance also merges `dependencyManagement` top-down, the two compose
transparently: a POM can *import* a BOM in its own `dependencyManagement`, and every one of that
POM's *children* (via ordinary `<parent>` inheritance) inherits the fully-resolved, merged
management table — without the child ever mentioning the BOM's `groupId:artifactId` at all. This
is exactly the trick this workspace relies on (§5): `super-pom` imports `learning-bom`, and every
leaf repo that has `super-pom` as its Maven `<parent>` inherits `learning-bom`'s entire managed
version table for free, with zero awareness that `learning-bom` exists.

---

## 3. How this BOM is structured, section by section

The full `pom.xml` is organized as one `<dependencyManagement>` block with clearly commented
sections. Below is a walkthrough of each, matching the actual file.

### 3.1 Platform BOMs (imported first, in a deliberate order)

```xml
<!-- ===== Platform BOMs (import order matters: first declaration wins) ===== -->
```

| Imported BOM | Property | Version |
|---|---|---|
| `org.springframework.boot:spring-boot-dependencies` | `spring-boot.version` | `4.1.0` |
| `org.springframework.cloud:spring-cloud-dependencies` | `spring-cloud.version` | `2025.1.2` |
| `org.springframework.ai:spring-ai-bom` | `spring-ai.version` | `2.0.0` |
| `org.testcontainers:testcontainers-bom` | `testcontainers.version` | `1.21.4` |

These four are themselves BOMs maintained by upstream projects, each managing dozens to hundreds
of their own coordinated artifacts (e.g. `spring-boot-dependencies` manages the versions of
`spring-boot-starter-web`, `jackson-databind`, `tomcat-embed-core`, and hundreds more;
`spring-cloud-dependencies` manages Spring Cloud Gateway/Config/OpenFeign/etc.; `spring-ai-bom`
manages every `spring-ai-*-spring-boot-starter` and the underlying model-client artifacts;
`testcontainers-bom` manages every `testcontainers` module such as `postgresql`, `kafka`,
`junit-jupiter`). Importing them here, in this order, means every downstream repo automatically
gets a mutually-tested, internally-consistent set of framework versions without `learning-bom`
having to know or re-declare any of their individual member artifacts.

### 3.2 Oracle JDBC

```xml
<!-- ===== Oracle JDBC ===== -->
```
`com.oracle.database.jdbc:ojdbc17` at `${ojdbc.version}` (`23.26.2.0.0`). Oracle does not publish
a BOM covering this driver in a way that's convenient to import here, so it is pinned directly as
an individually-managed artifact.

### 3.3 Resilience4j

```xml
<!-- ===== Resilience4j ===== -->
```
Five artifacts — `resilience4j-spring-boot3`, `resilience4j-reactor`,
`resilience4j-circuitbreaker`, `resilience4j-micrometer`, `resilience4j-retry` — all pinned
together to `${resilience4j.version}` (`2.3.0`). See §4.1 for why this is held below latest.

### 3.4 Observability

```xml
<!-- ===== Observability ===== -->
```
`io.github.mweirauch:micrometer-jvm-extras` (`${micrometer-jvm-extras.version}` = `0.3.0`) adds
JVM metrics (GC, classloading, thread pools) beyond what core Micrometer instruments by default.
`io.micrometer:context-propagation` (`${micrometer-context-propagation.version}` = `1.2.1`)
carries `ThreadLocal`/`Reactor Context` state (e.g. MDC, tracing spans) across async and reactive
boundaries.

### 3.5 Structured logging

```xml
<!-- ===== Structured logging ===== -->
```
`net.logstash.logback:logstash-logback-encoder` (`${logstash-logback.version}` = `9.0`) — emits
JSON-structured log lines consumable by a log aggregator (ELK/Loki/etc.) instead of plain-text
log4j-style formatting.

### 3.6 Distributed scheduling

```xml
<!-- ===== Distributed scheduling ===== -->
```
`net.javacrumbs.shedlock:shedlock-spring`, `shedlock-provider-jdbc-template`, and
`shedlock-provider-redis-spring`, all at `${shedlock.version}` (`5.16.0`). Shedlock prevents the
same `@Scheduled` job from running concurrently on more than one instance of a horizontally-scaled
service, using a JDBC row lock or a Redis lock as the distributed mutex depending on the provider
chosen. See §4.3 for why this is held back from the 7.x line.

### 3.7 Document processing

```xml
<!-- ===== Document processing ===== -->
```
| Artifact | Property | Version | Purpose |
|---|---|---|---|
| `org.apache.pdfbox:pdfbox` | `pdfbox.version` | `3.0.7` | Read/write/manipulate PDF documents |
| `org.apache.poi:poi-ooxml` | `poi-ooxml.version` | `5.5.1` | Read/write Office Open XML (`.docx`/`.xlsx`/`.pptx`) |
| `net.sourceforge.tess4j:tess4j` | `tess4j.version` | `5.19.0` | JNA bindings to Tesseract OCR |

These three together cover the document-ingestion pipeline used by the RAG/document-processing
repos in this workspace: extracting text from PDFs, Office documents, and scanned/image-based
pages via OCR.

### 3.8 AI SDKs

```xml
<!-- ===== AI SDKs ===== -->
```
`com.anthropic:anthropic-java` at `${anthropic-java.version}` (`2.48.0`) — the official Anthropic
Java SDK, used directly (outside of Spring AI's own abstraction) wherever a repo needs
lower-level access to the Claude Messages API, streaming, or tool-use primitives that Spring AI's
starter doesn't expose.

### 3.9 langchain4j

```xml
<!-- ===== langchain4j ===== -->
```
`dev.langchain4j:langchain4j-bom` is imported (`scope=import`, like the platform BOMs in §3.1) at
`${langchain4j-bom.version}` (`1.17.1`), managing the core LangChain4j modules (chains, memory,
embedding stores, tool integration, etc.) as one coordinated set.

Separately, further down the file:

```xml
<dependency>
    <!-- Community modules live on their own (beta) version track, outside langchain4j-bom. -->
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-community-redis</artifactId>
    <version>${langchain4j-redis.version}</version>
</dependency>
```

`langchain4j-community-redis` is pinned individually at `${langchain4j-redis.version}`
(`1.17.0-beta27`) rather than picked up from the BOM import, precisely because the
`langchain4j-community` modules (Redis, and others in the wider LangChain4j ecosystem) release on
their own beta cadence, independent of and often lagging or leading the core BOM's version line.
Importing them from the core BOM isn't possible since they aren't managed by it in the first
place — this entry exists because a repo in this workspace (`llm-rag` or similar) needs
Redis-backed embedding storage.

### 3.10 Kafka / Avro ecosystem

```xml
<!-- Kafka / Avro ecosystem -->
```
| Artifact | Property | Version |
|---|---|---|
| `org.apache.avro:avro` | `avro.version` | `1.12.1` |
| `io.confluent:kafka-avro-serializer` | `confluent.version` | `8.3.0` |

Avro provides the schema/serialization format; Confluent's `kafka-avro-serializer` integrates
Avro (de)serialization with the Confluent Schema Registry for Kafka producers/consumers. Note
that `io.confluent` artifacts are **not** published to Maven Central — any repo consuming this
managed version needs Confluent's Maven repository configured (`super-pom` already declares it
under `<repositories>`).

### 3.11 Test utilities

```xml
<!-- ===== Test utilities ===== -->
```
`io.swagger.parser.v3:swagger-parser` (`${swagger-parser.version}` = `2.1.45`) — parses/validates
OpenAPI/Swagger specification documents, used in tests that assert a service's generated OpenAPI
spec is well-formed or matches a contract.

### 3.12 OpenAPI / Swagger UI

```xml
<!-- ===== OpenAPI / Swagger UI ===== -->
```
`org.springdoc:springdoc-openapi-starter-webflux-ui` and
`org.springdoc:springdoc-openapi-starter-webmvc-ui`, both at `${springdoc.version}` (`3.0.3`) —
generates OpenAPI 3 documentation and a Swagger UI page from Spring MVC or WebFlux controller
annotations, depending on which stack a given repo uses.

### 3.13 QR code processing

```xml
<!-- ===== QR code processing ===== -->
```
`com.google.zxing:core` and `com.google.zxing:javase`, both at `${zxing.version}` (`3.5.4`) —
generate/decode QR codes (and other barcode formats); `javase` layers `java.awt`/ImageIO bindings
on top of the platform-independent `core`.

### 3.14 TOTP

```xml
<!-- ===== TOTP ===== -->
```
`dev.samstevens.totp:totp` at `${samstevens-totp.version}` (`1.7.1`) — implements RFC 6238
Time-based One-Time Password generation/validation, used for 2FA/MFA flows.

### 3.15 `openapi-generator` "spring" template runtime dependency

```xml
<!-- ===== openapi-generator "spring" template runtime dependency ===== -->
```
`org.openapitools:jackson-databind-nullable` at `${jackson-databind-nullable.version}`
(`0.2.10`) — this is not a library any repo depends on deliberately; it's a small runtime shim
that the `openapi-generator-maven-plugin`'s `spring` template (configured in `super-pom`, see
its `openapi-generator-maven-plugin.version` property) emits references to in generated model
classes, in order to distinguish "field absent" from "field explicitly set to `null`" in JSON.
It's pinned here so generated code always compiles against a known-good version regardless of
which repo runs the generator.

---

## 4. Deliberately held-back versions — and why

Three properties in this BOM are pinned *below* the latest available upstream release, each with
an inline comment in `pom.xml` explaining the reasoning. These are the most important entries in
the whole file to understand, because they represent accumulated debugging knowledge that would
otherwise have to be rediscovered independently by whoever next runs
`mvn versions:display-dependency-updates` and blindly bumps everything to latest.

### 4.1 Resilience4j — held at `2.3.0`

```xml
<!-- resilience4j 2.4.0 ships SpringBoot3Verifier that refuses Spring Boot 4.x at startup — hold at 2.3.0 -->
<resilience4j.version>2.3.0</resilience4j.version>
```

Resilience4j 2.4.0 introduced a `SpringBoot3Verifier` component whose job is to fail fast if the
Resilience4j Spring Boot integration is loaded under a Spring Boot major version it wasn't
verified against. Despite this BOM already being on **Spring Boot 4.1.0**, Resilience4j 2.4.0's
verifier hard-codes an expectation of Spring Boot 3.x and throws at application startup rather
than degrading gracefully. Rather than patching around a third-party startup verifier, the
pragmatic fix is to stay one minor version back, on 2.3.0, which predates the verifier's
introduction and works correctly against Spring Boot 4.x. This will need to be revisited once
Resilience4j ships a release whose verifier is aware of Spring Boot 4.

### 4.2 Testcontainers — held at `1.21.4`

```xml
<!-- Testcontainers 2.x renames module artifacts (junit-jupiter/postgresql/kafka/r2dbc no longer managed) — staying on latest 1.x until code migrates -->
<testcontainers.version>1.21.4</testcontainers.version>
```

Testcontainers' 2.x line is a breaking artifact-coordinate rename: modules that used to be
managed under coordinates such as `org.testcontainers:junit-jupiter`,
`org.testcontainers:postgresql`, `org.testcontainers:kafka`, and `org.testcontainers:r2dbc` are
restructured under different artifact IDs/groupIds in 2.x, meaning `testcontainers-bom` 2.x would
simply stop managing the coordinates that every integration test module in this workspace
currently imports (unqualified, relying on this BOM). Upgrading the version property alone,
without also touching every test module's dependency declarations across every repo, would break
builds workspace-wide. The BOM intentionally stays on the latest available `1.x` release
(`1.21.4`) until the module rename is deliberately migrated everywhere at once.

### 4.3 Shedlock — held at `5.16.0`

```xml
<!-- shedlock 7.x drops net.javacrumbs.shedlock.micrometer + AopMode used by learning-shedlock — hold at 5.16.0 until code migrates -->
<shedlock.version>5.16.0</shedlock.version>
```

Shedlock's 7.x line removes the `net.javacrumbs.shedlock.micrometer` package and the `AopMode`
configuration option, both of which are actively used by the `learning-shedlock` repo in this
workspace. Bumping this property to 7.x today would break `learning-shedlock`'s compilation the
moment it picks up the new BOM version, with no source-compatible substitute available yet in
that repo. As with Testcontainers, the version is intentionally frozen until the dependent code
is migrated off the removed APIs, at which point the property can be bumped and the migration
done in the same change.

**The general pattern:** each of these three comments documents a *specific, verified,
reproducible reason* a naive `mvn versions:use-latest-releases` would break something, discovered
once so it doesn't have to be rediscovered. Anyone tempted to "clean up" these pins by bumping
them to latest should first address the underlying blocker (verifier awareness, module rename
migration, or removed-API migration) called out in the comment, in the same change.

---

## 5. Position in the workspace's three-tier dependency-management chain

This BOM is never a Maven `<parent>` of anything — it is `pom`-packaged and only ever
`scope=import`-ed. The actual **parent/child inheritance chain** and the **BOM import** are two
separate mechanisms that compose, as described in §2.3:

- **`learning-bom`** (this repo) — declares no parent, packaging `pom`, only a
  `<dependencyManagement>` block. Never depended on directly by a leaf service.
- **`super-pom`** (`com.org.llm:super-pom`) — its own Maven `<parent>` is
  `org.springframework.boot:spring-boot-starter-parent` (so it inherits Spring Boot's default
  plugin configuration, resource filtering, and encoding settings). Inside its own
  `<dependencyManagement>`, it `import`-scopes `learning-bom` at `${learning-bom.version}`
  (currently `1.1.0`, matching this BOM's own `pom.xml` `<version>` — the two are kept in lockstep
  by hand today). `super-pom` also centrally wires shared build plugins (Spotless, the Maven
  Enforcer plugin, JaCoCo, the `openapi-generator-maven-plugin`, the `git-commit-id-maven-plugin`,
  Surefire/Failsafe with Java 25 module-system flags) that every leaf repo inherits alongside the
  dependency versions.
- **Leaf repos** (`llm-text2sql`, `llm-chat`, `llm-rag`, `llm-gateway`, `llm-mcp`,
  `llm-mcp-gateway`, `llm-deep-agent`, `llm-eval`, `llm-langchain4j`, `llm-OKF`,
  `learning-kafka`, `learning-graphql`, `learning-axon`, `learning-shedlock`,
  `learning-reactive`, `learning-utility`, `learning-wiremock`,
  `learning-testing-mutation`, ...) — each declares `com.org.llm:super-pom` as its Maven
  `<parent>` and nothing more. A leaf repo's own `pom.xml` typically lists dependencies with
  **no `<version>` tag at all** (see `llm-text2sql/pom.xml`'s `<dependencies>` block for a live
  example: `spring-boot-starter-web`, `spring-ai-starter-model-anthropic`, etc., all unversioned).
  Every version is resolved transitively: leaf → `super-pom` (ordinary parent inheritance) →
  `learning-bom` (import, merged into `super-pom`'s effective `dependencyManagement`) → the
  concrete pinned version or platform-BOM entry.

The leaf repo never mentions `learning-bom`'s `groupId:artifactId` anywhere in its own `pom.xml`.
This is the entire point of the pattern: centralizing the *knowledge* of which version to use, in
exactly one place, while every consumer stays completely unaware of where that knowledge lives.

### 5.1 Chain diagram

```mermaid
flowchart TD
    subgraph BOM["learning-bom  (pom, this repo)"]
        direction TB
        A1["Platform BOM imports:<br/>spring-boot-dependencies 4.1.0<br/>spring-cloud-dependencies 2025.1.2<br/>spring-ai-bom 2.0.0<br/>testcontainers-bom 1.21.4<br/>langchain4j-bom 1.17.1"]
        A2["Individually managed:<br/>resilience4j 2.3.0 · shedlock 5.16.0<br/>pdfbox 3.0.7 · poi-ooxml 5.5.1 · tess4j 5.19.0<br/>anthropic-java 2.48.0 · avro 1.12.1<br/>confluent 8.3.0 · springdoc 3.0.3 · ..."]
    end

    SP["super-pom  (com.org.llm:super-pom)<br/>parent = spring-boot-starter-parent<br/>imports learning-bom via &lt;dependencyManagement&gt;<br/>+ wires shared build plugins"]

    BOM -- "scope=import<br/>(dependencyManagement only, no code)" --> SP

    SP -- "&lt;parent&gt; inheritance" --> L1["llm-text2sql"]
    SP -- "&lt;parent&gt; inheritance" --> L2["llm-chat"]
    SP -- "&lt;parent&gt; inheritance" --> L3["llm-rag"]
    SP -- "&lt;parent&gt; inheritance" --> L4["llm-gateway / llm-mcp / llm-mcp-gateway"]
    SP -- "&lt;parent&gt; inheritance" --> L5["learning-kafka / learning-axon / learning-shedlock"]
    SP -- "&lt;parent&gt; inheritance" --> L6["... every other leaf repo"]

    L1 -.->|"no &lt;version&gt; tags —<br/>resolved transitively"| BOM
    L2 -.-> BOM
    L3 -.-> BOM
```

### 5.2 Managed libraries grouped by category

```mermaid
flowchart LR
    BOM(("learning-bom"))

    BOM --> WEB["Web / Framework<br/>spring-boot-dependencies<br/>spring-cloud-dependencies<br/>springdoc-openapi (webflux/webmvc)<br/>swagger-parser"]

    BOM --> AI["AI / LLM<br/>spring-ai-bom<br/>anthropic-java<br/>langchain4j-bom<br/>langchain4j-community-redis"]

    BOM --> DATA["Data / Documents<br/>pdfbox · poi-ooxml · tess4j<br/>ojdbc17"]

    BOM --> MSG["Messaging<br/>avro<br/>kafka-avro-serializer (Confluent)"]

    BOM --> RESIL["Resilience / Ops<br/>resilience4j (5 modules)<br/>shedlock (3 providers)<br/>micrometer-jvm-extras<br/>micrometer context-propagation<br/>logstash-logback-encoder"]

    BOM --> TEST["Testing<br/>testcontainers-bom"]

    BOM --> MISC["Misc utilities<br/>zxing (QR)<br/>samstevens totp<br/>jackson-databind-nullable"]
```

---

## 6. How to import (already wired — service repos should not repeat this)

`super-pom` already performs this import; leaf repos should never need to add it themselves —
they simply inherit `super-pom` as their `<parent>`:

```xml
<!-- inside super-pom's pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.org.learning</groupId>
            <artifactId>learning-bom</artifactId>
            <version>${learning-bom.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

If a leaf repo ever finds itself declaring a `<version>` for something already listed in §3
below, that is a signal something is wrong (either the repo isn't actually inheriting `super-pom`
correctly, or the dependency truly isn't managed here yet and belongs in this BOM instead).

---

## 7. How to add a new managed dependency

1. **Add a version property** to `<properties>`, following the existing
   `<artifactId>.version` convention (e.g. `<my-lib.version>1.2.3</my-lib.version>`).
2. **Add the dependency** under `<dependencyManagement><dependencies>`, placed in (or under a
   new) section comment matching the categorization used throughout this file.

   ```xml
   <dependency>
       <groupId>com.example</groupId>
       <artifactId>my-lib</artifactId>
       <version>${my-lib.version}</version>
   </dependency>
   ```
3. **If the library ships its own BOM**, prefer importing that BOM (`type=pom`,
   `scope=import`) over pinning individual artifacts by hand — follow the existing pattern used
   for `spring-boot-dependencies`, `spring-cloud-dependencies`, `spring-ai-bom`,
   `testcontainers-bom`, and `langchain4j-bom`. Place new platform-BOM imports mindfully with
   respect to the "first declaration wins" rule from §2.2 if the new BOM might manage a
   coordinate one of the existing platform BOMs already manages.
4. **If you deliberately hold a version back from latest** (as in §4), add an inline comment
   directly above the property explaining precisely what breaks at the newer version and what
   condition would need to be true before it's safe to bump — future maintainers (including
   yourself, six months from now) should not have to rediscover the reason from scratch.
5. **Bump this BOM's own `<version>`** in `pom.xml` (semantic versioning: patch for
   version-only bumps to existing entries, minor for new managed dependencies, major for anything
   that could break a consumer, e.g. removing a managed entry), run `mvn install` in this
   directory to publish the new version to the local repository, then bump
   `<learning-bom.version>` in `super-pom`'s `pom.xml` to match.
6. Leaf repos pick up the change automatically the next time they build against the updated
   `super-pom` — no leaf repo `pom.xml` needs to change.

---

## 8. Quick reference — all managed dependencies

### Platform BOMs (import order matters — first declaration wins on overlapping coordinates)

| BOM | Version |
|---|---|
| `org.springframework.boot:spring-boot-dependencies` | `4.1.0` |
| `org.springframework.cloud:spring-cloud-dependencies` | `2025.1.2` |
| `org.springframework.ai:spring-ai-bom` | `2.0.0` |
| `org.testcontainers:testcontainers-bom` | `1.21.4` |
| `dev.langchain4j:langchain4j-bom` | `1.17.1` |

### Individually pinned dependencies

| Group / Artifact | Version property | Version |
|---|---|---|
| `com.oracle.database.jdbc:ojdbc17` | `ojdbc.version` | `23.26.2.0.0` |
| `io.github.resilience4j:resilience4j-spring-boot3` | `resilience4j.version` | `2.3.0` (held — §4.1) |
| `io.github.resilience4j:resilience4j-reactor` | `resilience4j.version` | `2.3.0` (held — §4.1) |
| `io.github.resilience4j:resilience4j-circuitbreaker` | `resilience4j.version` | `2.3.0` (held — §4.1) |
| `io.github.resilience4j:resilience4j-micrometer` | `resilience4j.version` | `2.3.0` (held — §4.1) |
| `io.github.resilience4j:resilience4j-retry` | `resilience4j.version` | `2.3.0` (held — §4.1) |
| `io.github.mweirauch:micrometer-jvm-extras` | `micrometer-jvm-extras.version` | `0.3.0` |
| `io.micrometer:context-propagation` | `micrometer-context-propagation.version` | `1.2.1` |
| `net.logstash.logback:logstash-logback-encoder` | `logstash-logback.version` | `9.0` |
| `net.javacrumbs.shedlock:shedlock-spring` | `shedlock.version` | `5.16.0` (held — §4.3) |
| `net.javacrumbs.shedlock:shedlock-provider-jdbc-template` | `shedlock.version` | `5.16.0` (held — §4.3) |
| `net.javacrumbs.shedlock:shedlock-provider-redis-spring` | `shedlock.version` | `5.16.0` (held — §4.3) |
| `org.apache.pdfbox:pdfbox` | `pdfbox.version` | `3.0.7` |
| `org.apache.poi:poi-ooxml` | `poi-ooxml.version` | `5.5.1` |
| `net.sourceforge.tess4j:tess4j` | `tess4j.version` | `5.19.0` |
| `com.anthropic:anthropic-java` | `anthropic-java.version` | `2.48.0` |
| `org.apache.avro:avro` | `avro.version` | `1.12.1` |
| `io.confluent:kafka-avro-serializer` | `confluent.version` | `8.3.0` |
| `dev.langchain4j:langchain4j-community-redis` | `langchain4j-redis.version` | `1.17.0-beta27` |
| `io.swagger.parser.v3:swagger-parser` | `swagger-parser.version` | `2.1.45` |
| `org.springdoc:springdoc-openapi-starter-webflux-ui` | `springdoc.version` | `3.0.3` |
| `org.springdoc:springdoc-openapi-starter-webmvc-ui` | `springdoc.version` | `3.0.3` |
| `com.google.zxing:core` | `zxing.version` | `3.5.4` |
| `com.google.zxing:javase` | `zxing.version` | `3.5.4` |
| `dev.samstevens.totp:totp` | `samstevens-totp.version` | `1.7.1` |
| `org.openapitools:jackson-databind-nullable` | `jackson-databind-nullable.version` | `0.2.10` |

---

## 9. Versioning policy for this BOM itself

`learning-bom`'s own `<version>` follows semantic versioning as a signal to `super-pom` (its sole
consumer):

- **Patch** (`1.1.0` → `1.1.1`): version-number-only bumps to existing managed entries (e.g.
  bumping `pdfbox.version`), no additions or removals.
- **Minor** (`1.1.0` → `1.2.0`): a new managed dependency or platform BOM is added; existing
  consumers are unaffected unless they start using the new artifact.
- **Major** (`1.1.0` → `2.0.0`): a managed entry is removed, or an existing property's semantics
  change in a way that could silently change resolved versions for existing consumers (e.g.
  swapping which platform BOM wins on an overlapping coordinate).

`super-pom` pins the consumed version explicitly via `<learning-bom.version>`, so a new
`learning-bom` release never affects any leaf repo until `super-pom` is deliberately updated to
point at it — there is no "floating latest" resolution anywhere in this chain.
