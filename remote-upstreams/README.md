
## Fetching data from Remote Rest API&#39;s

Summary: In the example below we will connect to a Remote Rest API using GraphQL.

Prerequisites:

- kubectl
-  helm
- 2 Clusters or 1 Cluster and 1 Remote API

In this example I&#39;ve used the Istio Bookinfo example application, which has four services, product-page, details, reviews and ratings.

We will deploy the ratings service on the second cluster.

In the first cluster run the following commands

```
kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l app=details
kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l account=details

kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l app=reviews
kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l account=reviews

kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l app=productpage
kubectl apply --context cluster-1 -f  https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l account=productpage
```

On cluster two

```
kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l app=ratings
kubectl apply --context cluster-1 -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -l service=ratings
```

On `Cluster-2` create a VirtualService so we can route traffic to the Rest Endpoint we have

```
# This is for the 2nd cluster(remote)
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: default
  namespace: gloo-system
spec:
  virtualHost:
    domains:
    - '*'
    options:
      cors:
        allowCredentials: true
        allowHeaders:
        - content-type
        allowMethods:
        - POST
        - GET
        allowOriginRegex:
        - '/*'
    routes:
      - matchers:
        - prefix: '/'
        routeAction:
          single:
            upstream:
              name: 'default-ratings-9080'
              namespace: 'gloo-system'
```

Next, we will create an upstream in `Cluster-1`this will point to the Ratings service hosted in cluster-2. This can be a remote Rest API URL.

```yaml
kubectl apply --context cluster-1 -f <<EOF>
apiVersion: gloo.solo.io/v1
kind: Upstream
metadata:
  name: ratings-upstream
  namespace: gloo-system
spec:
  static:
    hosts:
      - addr: 192.168.31.151
        port: 8082
EOF
```

In `Cluster-1` we need to declare the BookInfo GraphQLApi Object.

```
kubectl apply --context cluster-1 -f <<EOF>
apiVersion: graphql.gloo.solo.io/v1beta1
kind: GraphQLApi
metadata:
  name: bookinfo-graphql
  namespace: gloo-system
spec:
  executableSchema:
    executor:
      local:
        enableIntrospection: true
        resolutions:
          Query|productsForHome:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /api/v1/products
              upstreamRef:
                name: default-productpage-9080
                namespace: gloo-system
          author:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /details/{$parent.id}
              response:
                resultRoot: "author"
              upstreamRef:
                name: default-details-9080
                namespace: gloo-system
          pages:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /details/{$parent.id}
              response:
                resultRoot: "pages"
              upstreamRef:
                name: default-details-9080
                namespace: gloo-system
          year:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /details/{$parent.id}
              response:
                resultRoot: "year"
              upstreamRef:
                name: default-details-9080
                namespace: gloo-system
          review:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /reviews/{$parent.id}
              response:
                resultRoot: "reviews"
              upstreamRef:
                name: default-reviews-9080
                namespace: gloo-system
          ratings:
            restResolver:
              request:
                headers:
                  :method: GET
                  :path: /ratings/{$parent.id}
              response:
                resultRoot: "ratings[*]"
                setters:
                  reviewer: "{$body[*][0]}"
                  numStars: "{$body[*][1]}"
              upstreamRef:
                name: ratings-upstream
                namespace: gloo-system
    schemaDefinition: |
      type Query {
        """Description of a book in HTML"""
        productsForHome: [Product] @resolve(name: "Query|productsForHome")
      }

      """Each book has a product entry"""
      type Product {
        """Unique identifier for books"""
      id: String
      """The book title"""
      title: String
      """Description of a book in HTML"""
      descriptionHtml: String
      """Author of the book"""
      author: String @resolve(name: "author")
      """Total number of pages in the book"""
      pages: Int @resolve(name: "pages")
      """Year of original publication"""
      year: Int @resolve(name: "year")
      """List of reader reviews for this book"""
      reviews: [Review] @resolve(name: "review")
      """List of reader ratings for this book"""
      ratings: [Rating] @resolve(name: "ratings")
      }

      """A book review"""
      type Review {
          """Name of the reviewer"""
          reviewer: String
          """Review details"""
          text: String
      }

      """A book rating"""
      type Rating {
          """Name of the user peforming the rating"""
          reviewer: String
          """Number of stars for this rating"""
          numStars: Int
      }
EOF
```

If you notice below

```yaml
ratings:
  restResolver:
    request:
      headers:
        :method: GET
        :path: /ratings/{$parent.id}
    response:
      resultRoot: "ratings[*]"
      setters:
        reviewer: "{$body[*][0]}"
        numStars: "{$body[*][1]}"
    upstreamRef:
      name: ratings-upstream
      namespace: gloo-system

#------------- snipped ------------------# 
"""A book rating"""
      type Rating {
          """Name of the user peforming the rating"""
          reviewer: String
          """Number of stars for this rating"""
          numStars: Int
      }
```

we have used the RestResolver, but here we don&#39;t need to worry about the Upstream being in the same cluster, we can point the Upstream to anything we want, this could be a REST API on a VM, a container, something in the cloud.

Once you have the `GraphQLApi` defined we can setup the `VirtualService` for `Cluster-1`

```
kubectl apply -f <<EOF>
apiVersion: gateway.solo.io/v1
kind: VirtualService
metadata:
  name: default
  namespace: gloo-system
spec:
  virtualHost:
    domains:
    - '*'
    options:
      cors:
        allowCredentials: true
        allowHeaders:
        - content-type
        allowMethods:
        - POST
        allowOriginRegex:
        - '/*'
    routes:
    #----------- Bookinfo GraphQL API --------
    - graphqlApiRef:
        name: bookinfo-graphql
        namespace: gloo-system
      matchers:
      - prefix: /graphql
EOF
```



Query

```
query ProductsForHome {
  productsForHome {
    id
    title
    descriptionHtml
    author
    pages
    year
    reviews {
      text
      reviewer
    }
    ratings {
        numStars
    }
  }
}
```

Response

```
{
    "data": {
        "productsForHome": [
            {
                "id": "0",
                "title": "The Comedy of Errors",
                "descriptionHtml": "<a href=\"https://en.wikipedia.org/wiki/The_Comedy_of_Errors\">Wikipedia Summary</a>: The Comedy of Errors is one of <b>William Shakespeare's</b> early plays. It is his shortest and one of his most farcical comedies, with a major part of the humour coming from slapstick and mistaken identity, in addition to puns and word play.",
                "author": "William Shakespeare",
                "pages": 200,
                "year": 1595,
                "reviews": [
                    {
                        "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!",
                        "reviewer": "Reviewer1"
                    },
                    {
                        "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.",
                        "reviewer": "Reviewer2"
                    }
                ],
                "ratings": [
                    {
                        "numStars": 5
                    },
                    {
                        "numStars": 4
                    }
                ]
            }
        ]
    }
}
```
