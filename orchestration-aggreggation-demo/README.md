## Orchestration/Aggregation in GraphQL

Summary: This example explains relations between `types` in the GraphQL schema to fetch data without having to do multiple API calls to the `GraphQL` or `Rest API's` backends.

We&#39;ve deployed a simple Petstore application to show this demo.

The schema defined in GraphQL schema definition language is shown below.

**Note:**  We can also auto-generate GraphQLApi schema&#39;s if your application implements Swagger Openapi spec.

**Schema Definition**

```
schemaDefinition: |
      type Pet {
        id: Int!
        name: String!
      }

      type Order {
        id: Int
        petId: Int!
        pet: Pet @resolve(name: "getPet") # One to One Relationship
      }

      type Query {
        getOrderById(orderId: Int!): Order @resolve(name: "getOrderById")
      }
```

As you can see we have defined a `Pet`, `Order` and the `Query` type here, we&#39;ve also made a one to one relationship with the order and pet, which means one order can have one pet based on the `orderId`

The following query will give you the output.

```
query {
    getOrderById(orderId: 1)
    {
        petId
        id
        pet {
            id
            name
        }
    }
}
```

The resolver config for this is as follows.

```
spec:
  executableSchema:
    executor:
      local:
        enableIntrospection: true
        resolutions:
          getPet:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /v3/pet/{$parent.petId}
              upstreamRef:
                name: default-petstore-8080
                namespace: gloo-system
          getOrderById:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /v3/store/order/{$args.orderId}
              upstreamRef:
                name: default-petstore-8080
                namespace: gloo-system
```

In the output shown above, we&#39;ve defined `restResolver` to resolve the respective REST API path&#39;s using the `upstreamRef` fields.

The entire GraphQLApi YAML is shown below.

```
apiVersion: graphql.gloo.solo.io/v1beta1
kind: GraphQLApi
metadata:
  name: default-petstore-8080
  namespace: gloo-system
spec:
  executableSchema:
    executor:
      local:
        enableIntrospection: true
        resolutions:
          getPet:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /v3/pet/{$parent.petId}
              upstreamRef:
                name: default-petstore-8080
                namespace: gloo-system
          getOrderById:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /v3/store/order/{$args.orderId}
              upstreamRef:
                name: default-petstore-8080
                namespace: gloo-system
    schemaDefinition: |-
      type Pet {
        id: Int!
        name: String!
      }
      type Order {
        id: Int
        petId: Int!
        pet: Pet @resolve(name: "getPet")
      }

      type Query {
        getOrderById(orderId: Int!): Order @resolve(name: "getOrderById")
      }
```
