---
title: "mu-server 2.4.2 OpenAPI 集成"
category: synthesis
tags: [java, mu-server, openapi, api-documentation, 2.4.2]
sources: ["mu-server 2.4.2 @ tag mu-server-2.4.2 (https://github.com/3redronin/mu-server)"]
summary: "从 JAX-RS 资源自动生成 OpenAPI 3.x 规范, schema generators, 自定义 SchemaObjectCustomizer, 集成到 MuServerBuilder"
provenance:
  extracted: 0.85
  inferred: 0.10
  ambiguous: 0.05
base_confidence: 0.82
lifecycle: draft
lifecycle_changed: 2026-09-11
created: 2026-09-11
updated: 2026-09-11
---

# Draft 06 — OpenAPI 3 Implementation (`io.muserver.openapi`)

> Scope: the OpenAPI 3 object model + builder DSL + document generator.

## 1. Files at a glance

| File | LOC | Role |
|---|---|---|
| `OpenAPIObject` / `OpenAPIObjectBuilder` | — | Root document |
| `InfoObject` / `InfoObjectBuilder` | — | Title, version, description, license, contact |
| `ServerObject` / `ServerObjectBuilder` | — | Server URLs |
| `PathsObject` / `PathsObjectBuilder` | — | URL → path-item map |
| `PathItemObject` / `PathItemObjectBuilder` | — | Per-path operations |
| `OperationObject` / `OperationObjectBuilder` | — | Per-HTTP-method operation |
| `ParameterObject` / `ParameterObjectBuilder` | — | Query / header / path params |
| `RequestBodyObject` / `RequestBodyObjectBuilder` | — | Body description |
| `ResponsesObject` / `ResponsesObjectBuilder` | — | Status → Response map |
| `ResponseObject` / `ResponseObjectBuilder` | — | One status response |
| `MediaTypeObject` / `MediaTypeObjectBuilder` | — | Schema + example per content-type |
| `SchemaObject` / `SchemaObjectBuilder` | 504 / 942 | Type definition (largest in the project) |
| `ComponentsObject` / `ComponentsObjectBuilder` | — | Reusable parts |
| `SecuritySchemeObject` / `SecuritySchemeObjectBuilder` | — | OAuth, basic, etc. |
| `SecurityRequirementObject` / `SecurityRequirementObjectBuilder` | — | Required security |
| `OAuthFlowsObject` / `OAuthFlowsObjectBuilder` | — | OAuth flow container |
| `OAuthFlowObject` / `OAuthFlowObjectBuilder` | — | One flow |
| `TagObject` / `TagObjectBuilder` | — | Tag for grouping |
| `LinkObject` / `LinkObjectBuilder` | — | HATEOAS link |
| `CallbackObject` / `CallbackObjectBuilder` | — | Webhook callback |
| `EncodingObject` / `EncodingObjectBuilder` | — | multipart encoding |
| `ExampleObject` / `ExampleObjectBuilder` | — | Single example |
| `HeaderObject` / `HeaderObjectBuilder` | — | Response header |
| `DiscriminatorObject` / `DiscriminatorObjectBuilder` | — | Polymorphism |
| `ExternalDocumentationObject` / `...Builder` | — | external docs URL |
| `LicenseObject` / `LicenseObjectBuilder` | — | License info |
| `ContactObject` / `ContactObjectBuilder` | — | Contact info |
| `XmlObject` / `XmlObjectBuilder` | — | XML metadata |
| `ServerVariableObject` / `ServerVariableObjectBuilder` | — | Server URL template vars |
| `Jsonizer.java` / `JsonWriter.java` / `OpenApiUtils.java` | — | Serialization |

Total ≈ 4,000 LOC.

## 2. SchemaObject (504 LOC) + SchemaObjectBuilder (942 LOC)

The biggest single file. `SchemaObject` represents one OpenAPI schema;
`SchemaObjectBuilder` builds it from a Java `Class<?>` via reflection
(`schemaObjectFrom(Class<?>)`) or explicitly (`schemaObject()`).

### 2.1 SchemaObjectBuilder.fromClass (the magic)

`schemaObjectFrom(...)` walks a Java class's fields via reflection and
emits a `SchemaObject`:

- **Primitive** → `{type: "integer", format: "int32"}` etc.
- **String / Number / Boolean** → `{type: "string"}`.
- **`java.time.LocalDate`** → `{type: "string", format: "date"}`.
- **`java.time.Instant`** → `{type: "string", format: "date-time"}`.
- **BigDecimal / BigInteger** → string/number with appropriate format.
- **Enum** → string + enum values.
- **Map<K,V>** → `{type: "object", additionalProperties: schemaOf(V)}`.
- **Collection (List/Set/array)** → `{type: "array", items: schemaOf(element)}`.
- **Bean (custom class)** → `{type: "object", properties: {...}, required: [...]}`. Recurses into fields.

The reflection walks:
1. Class hierarchy → only fields declared on the actual class and parents
2. Static fields ignored
3. Transient fields ignored
4. `@Schema` annotation overrides type/format/description
5. `@JsonProperty` (Jackson) annotation provides alternate name
6. Generic type is unwrapped (e.g. `List<MyDto>` → `items: schemaOf(MyDto)`)

### 2.2 Polymorphism (`@Schema` `subTypes`)

If a field's declared type has `@Schema(subTypes=...)`, the emitted
schema gets `oneOf: [schemaOf(Sub1), schemaOf(Sub2), ...]` plus a
`discriminator` based on the `@JsonTypeInfo` mapping.

### 2.3 Custom schemas

`RestHandlerBuilder.addCustomSchema(MyDto.class, schema)` registers a
hand-built `SchemaObject` for a class. Whenever a method parameter or
return type uses `MyDto`, that schema is referenced (via `$ref` to
`#/components/schemas/MyDto`). The class name is the component key, but
if the class has `@Description("api-name")`, that's used instead.

`addSchemaObjectCustomizer(SchemaObjectCustomizer)` is a callback for
modifying schemas at generation time (e.g. add a `description`).

## 3. OpenApiDocumentor + OpenAPI generation

`OpenApiDocumentor` (in the rest package) is the class that walks the
registered resources and generates the final `OpenAPIObject`. It is
created in `RestHandlerBuilder.build()` (line 568).

### 3.1 The path walking algorithm

For each `ResourceClass`:
1. The class-level `@Path` is prepended to every method path.
2. For each `ResourceMethod`:
   - Build an `OperationObject` with:
     - `operationId` = method name + parameter types (overridable via
       `@Description(operationId=...)`)
     - `summary` / `description` from `@Description` or javadoc
     - `tags` from `@Tag`
     - `parameters` = one `ParameterObject` per `@QueryParam`,
       `@PathParam`, `@HeaderParam`, `@CookieParam`, `@MatrixParam`,
       `@FormParam` (form goes into the body instead)
     - `requestBody` from `@FormParam` or method parameter type with no
       annotation that isn't a JAX-RS parameter annotation
     - `responses` = synthetic 200/204/400/500/etc based on return type
       and `@ApiResponse`/`@ApiResponses` annotations
   - Path templates are normalised so `/foo/` and `/foo` map to the
     same key.

### 3.2 Output endpoints

If `withOpenApiJsonUrl(...)` is set, the JSON spec is served at that
URL. If `withOpenApiHtmlUrl(...)` is set, the HTML view is served.

The HTML view is a self-contained page that fetches the JSON and
renders it with a small inlined renderer (no Swagger UI dependency).
Default CSS lives at `/io/muserver/resources/api.css` and is loaded via
`ClassLoader.getResourceAsStream` if no override is provided.

## 4. Jsonizer / JsonWriter — the serializer

`OpenApiUtils.Jsonizer` + `JsonWriter` are mu-server's minimal JSON
serializer. They produce well-formed JSON for all OpenAPI object types
without pulling in Jackson/Gson.

The implementation is a streaming JSON writer: `writeString`, `writeInt`,
`writeObjectStart`, `writeObjectField`, `writeObjectEnd`, etc. It uses
`Appendable` so the documentor can write to any sink.

This is **one of the few places** where mu-server ships its own
implementation of a "common" library — to keep the dependency footprint
minimal (Jackson is only in the test scope).

## 5. SchemaObjectCustomizer / SchemaObjectCustomizerContext

```java
public interface SchemaObjectCustomizer {
    void customize(SchemaObject schema, SchemaObjectCustomizerContext context);
}
```

The context exposes:
- `getSchema()` — current schema
- `setSchema(SchemaObject)` — replace
- `getType()` — the Java type being converted
- `isRequest()` — true if this is a request-side schema (parameter/body)
- `isResponse()` — true if response-side
- `getPropertyName()` — field name (for nested customisation)

Useful for adding vendor extensions (`x-...`) or modifying the schema
based on the location.

## 6. Notable gaps vs Swagger Core

| Feature | mu-server | Swagger Core |
|---|---|---|
| `@Schema(subTypes)` | ✅ | ✅ |
| Polymorphism via `@JsonTypeInfo` | ✅ | ✅ |
| Bean Validation (`@NotNull` → `required: true`) | ❌ | ✅ |
| Vendor extensions in `@Operation` | partial | ✅ |
| `@ExampleObject` from `@Example` | ✅ | ✅ |
| `@Parameter(explode=true)` (style/explode) | partial | ✅ |
| Auto-tag grouping by class | ✅ | ✅ |

## 相关笔记

- [[draft-01-protocol-layer|协议层 (HTTP/1.1, HTTP/2, ALPN, HAProxy, 背压)]]
- [[draft-02-abstract-layer|抽象层 (Request/Response/HttpExchange)]]
- [[draft-03-dispatcher-layer|分发层 (NettyHandlerAdapter + MuServerBuilder + 路由)]]
- [[draft-04-handlers-library|内置 Handler 库 (CORS/CSRF/StaticResource/HttpRedirect)]]
- [[draft-05-rest-jax-rs|JAX-RS 3.0 / REST 支持]]
- [[draft-07-async-sse-websocket|异步 / SSE / WebSocket]]
- [[draft-08-tls-sni-http2|TLS / SNI / HTTP/2]]
- [[draft-09-utility-classes|工具类]]
- [[draft-10-evolution-and-comparison|【对比】0.0.3 → 2.2.9 → 2.4.2 演进]]
- [[summary|综合报告]]
- [[moc|MOC 导航]]
- [[mu-server-netty-analysis/summary|0.0.3-SNAPSHOT 旧版分析]]
- [[mu-server-2.2.9-analysis/summary|2.2.9 历史分析]]
