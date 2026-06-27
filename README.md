# learning-bom — Centralized Dependency Version Management

This BOM (Bill of Materials) is the **single source of truth** for all
third-party dependency versions used across `com.org.llm` services.
No service pom should declare a `<version>` tag on any dependency that is
managed here.

## How to import

In `super-pom` (already wired — service poms need not do this themselves):

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.org.learning</groupId>
            <artifactId>learning-bom</artifactId>
            <version>1.0.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## Managed frameworks and versions

### Platform BOMs (imported in order; first declaration wins on conflicts)

| BOM | Version |
|---|---|
| `spring-boot-dependencies` | 4.1.0 |
| `spring-cloud-dependencies` | 2025.1.2 |
| `spring-ai-bom` | 2.0.0 |
| `testcontainers-bom` | 1.21.3 |
| `langchain4j-bom` | 1.16.3 |

### Individually pinned dependencies

| Group / Artifact | Version property | Version |
|---|---|---|
| `io.github.resilience4j:resilience4j-spring-boot3` | `resilience4j.version` | 2.3.0 |
| `io.github.resilience4j:resilience4j-reactor` | `resilience4j.version` | 2.3.0 |
| `io.github.resilience4j:resilience4j-circuitbreaker` | `resilience4j.version` | 2.3.0 |
| `io.github.resilience4j:resilience4j-micrometer` | `resilience4j.version` | 2.3.0 |
| `io.github.resilience4j:resilience4j-retry` | `resilience4j.version` | 2.3.0 |
| `io.github.mweirauch:micrometer-jvm-extras` | `micrometer-jvm-extras.version` | 0.2.2 |
| `io.micrometer:micrometer-context-propagation` | — | 1.1.2 |
| `net.logstash.logback:logstash-logback-encoder` | `logstash-logback.version` | 8.1 |
| `net.javacrumbs.shedlock:shedlock-spring` | `shedlock.version` | 5.16.0 |
| `net.javacrumbs.shedlock:shedlock-provider-redis-spring` | `shedlock.version` | 5.16.0 |
| `org.apache.pdfbox:pdfbox` | `pdfbox.version` | 3.0.7 |
| `org.apache.poi:poi-ooxml` | `poi-ooxml.version` | 5.5.1 |
| `net.sourceforge.tess4j:tess4j` | `tess4j.version` | 5.13.0 |
| `com.anthropic:anthropic-java` | `anthropic-java.version` | 2.34.0 |
| `org.springframework.ai:spring-ai-openai-spring-boot-starter` | `spring-ai.version` | 2.0.0 |
| `org.apache.avro:avro` | `avro.version` | 1.12.0 |
| `io.confluent:kafka-avro-serializer` | `confluent.version` | 7.7.1 |
| `dev.langchain4j:langchain4j-community-redis` | `langchain4j-redis.version` | 1.16.0-beta26 |
| `io.swagger.parser.v3:swagger-parser` | `swagger-parser.version` | 2.1.22 |
| `org.springdoc:springdoc-openapi-starter-webflux-ui` | `springdoc.version` | 2.8.9 |
| `org.springdoc:springdoc-openapi-starter-webmvc-ui` | `springdoc.version` | 2.8.9 |
| `com.google.zxing:core` | `zxing.version` | 3.5.3 |
| `com.google.zxing:javase` | `zxing.version` | 3.5.3 |
| `dev.samstevens.totp:totp` | `samstevens-totp.version` | 1.7.1 |
| `org.openapitools:jackson-databind-nullable` | `jackson-databind-nullable.version` | 0.2.6 |

## How to add a new dependency version

1. **Add a version property** to `<properties>` using the convention
   `<artifactId>.version` (e.g. `<my-lib.version>1.2.3</my-lib.version>`).

2. **Add the dependency** to `<dependencyManagement><dependencies>` under the
   appropriate section comment:

   ```xml
   <dependency>
       <groupId>com.example</groupId>
       <artifactId>my-lib</artifactId>
       <version>${my-lib.version}</version>
   </dependency>
   ```

3. **If the library ships its own BOM**, prefer importing that BOM over pinning
   individual artifacts — follow the existing pattern for `langchain4j-bom`.

4. Bump the BOM version in `pom.xml` (follow semantic versioning), run
   `mvn install` in this directory, then bump the reference in `super-pom`.

5. Child modules can then declare the dependency without a `<version>` tag.
