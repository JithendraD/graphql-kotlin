# GraphQL Kotlin Federated Schema Generator
[![Maven Central](https://img.shields.io/maven-central/v/com.expediagroup/graphql-kotlin-federation.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:%22com.expediagroup%22%20AND%20a:%22graphql-kotlin-federation%22)
[![Javadocs](https://img.shields.io/maven-central/v/com.expediagroup/graphql-kotlin-federation.svg?label=javadoc&colorB=brightgreen)](https://www.javadoc.io/doc/com.expediagroup/graphql-kotlin-federation)

`graphql-kotlin-federation` extends the functionality of `graphql-kotlin-schema-generator` and allows you to easily
generate federated GraphQL schemas directly from the code. Federated schemas rely on a number of directives to
instrument the behavior of the underlying graph, see [the documentation](federated-directives).
Once all the federated objects are annotated, you will also have to configure corresponding [FederatedTypeResolver]s
that are used to instantiate federated objects and finally generate the schema using `toFederatedSchema` function
([link](https://github.com/ExpediaGroup/graphql-kotlin/blob/master/graphql-kotlin-federation/src/main/kotlin/com/expedia/graphql/federation/toFederatedSchema.kt#L18)).

See more

* [Federation Spec](https://www.apollographql.com/docs/apollo-server/federation/federation-spec/)

## Installation

Using a JVM dependency manager, simply link `graphql-kotlin-federation` to your project.

With Maven:

```xml
<dependency>
  <groupId>com.expediagroup</groupId>
  <artifactId>graphql-kotlin-federation</artifactId>
  <version>${latestVersion}</version>
</dependency>
```

With Gradle:

```groovy
compile(group: 'com.expediagroup', name: 'graphql-kotlin-federation', version: "$latestVersion")
```

## Usage

In order to generate valid federated schemas, you will need to annotate both your base schema and the one extending it. Federated Gateway (e.g. Apollo) will then combine the individual graphs to form single federated graph.

#### Base Schema

Base schema defines GraphQL types that will be extended by schemas exposed by other GraphQL services. In the example below, we define base `Product` type with `id` and `description` fields. `id` is the primary key that uniquely identifies the `Product` type object and is specified in `@key` directive.

```kotlin
@KeyDirective(fields = FieldSet("id"))
data class Product(val id: Int, val description: String)

class ProductQuery {
  fun product(id: Int): Product? {
    // grabs product from a data source, might return null
  }
}

// Generate the schema
val federatedTypeRegistry = FederatedTypeRegistry(emptyMap())
val config = FederatedSchemaGeneratorConfig(supportedPackages = listOf("org.example"), hooks = FederatedSchemaGeneratorHooks(federatedTypeRegistry))
val queries = listOf(TopLevelObject(ProductQuery()))

toFederatedSchema(config, queries)
```

Generates the following schema with additional federated types

```graphql
schema {
  query: Query
}

union _Entity = Product

type Product @key(fields : "id") {
  description: Int!
  id: String!
}

type Query {
  _entities(representations: [_Any!]!): [_Entity]!
  _service: _Service
  product(id: Int!): Product!
}

type _Service {
  sdl: String!
}
```

#### Extended Schema

Extended federated GraphQL schemas provide additional functionality to the types already exposed by other GraphQL services. In the example below, `Product` type is extended to add new `reviews` field to it. Primary key needed to instantiate the `Product` type (i.e. `id`) has to match the `@key` definition on the base type. Since primary keys are defined on the base type and are only referenced from the extended type, all of the fields that are part of the field set specified in `@key` directive have to be marked as `@external`.

```kotlin
@KeyDirective(fields = FieldSet("id"))
@ExtendsDirective
data class Product(@ExternalDirective val id: Int) {

    fun reviews(): List<Review> {
        // returns list of product reviews
    }
}

data class Review(val reviewId: String, val text: String)

// Generate the schema
val productResolver = object: FederatedTypeResolver<Product> {
    override fun resolve(keys: Map<String, Any>): Product {
        val id = keys["id"]?.toString()?.toIntOrNull()
        // instantiate product using id
    }
}
val federatedTypeRegistry = FederatedTypeRegistry(mapOf("Product" to productResolver))
val config = FederatedSchemaGeneratorConfig(supportedPackages = listOf("org.example"), hooks = FederatedSchemaGeneratorHooks(federatedTypeRegistry))

toFederatedSchema(config)
```

Generates the following federated schema

```graphql
schema {
  query: Query
}

union _Entity = Product

type Product @extends @key(fields : "id") {
  id: Int! @external
  reviews: [Review!]!
}

type Query {
  _entities(representations: [_Any!]!): [_Entity]!
  _service: _Service
}

type Review {
  reviewId: String!
  text: String!
}

type _Service {
  sdl: String!
}
```

Federated Gateway will then combine the schemas from the individual services to generate single schema.

#### Federated GraphQL schema

```graphql
schema {
  query: Query
}

type Product {
  description: String!
  id: String!
  reviews: [Review!]!
}

type Review {
  reviewId: String!
  text: String!
}

type Query {
  product(id: String!): Product!
}
```

## Documentation

There are more examples in our [documentation](https://expediagroup.github.io/graphql-kotlin),
or you can view the [javadocs](https://www.javadoc.io/doc/com.expediagroup/graphql-kotlin-federation) for all published versions.

If you have a question about something you can not find in our documentation or javadocs, feel free to [create an issue](https://github.com/ExpediaGroup/graphql-kotlin/issues) and tag it with the question label.
