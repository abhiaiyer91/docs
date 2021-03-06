---
title: Network layer
order: 141
description: How to point your Apollo client to a different GraphQL server, or use a totally different protocol.
---

<h2 id="custom-network-interface">Custom network interface</h2>

You can define a custom network interface and pass it to the Apollo Client to send your queries in a different way. You might want to do this for a variety of reasons:

1. You want a custom transport that sends queries over Websockets instead of HTTP
2. You want to modify the query or variables before they are sent
3. You want to run your app against a mocked client-side schema and never send any network requests at all

All you need to do is create a `NetworkInterface` and pass it to the `ApolloClient` constructor.

<h3 id="NetworkInterface">interface NetworkInterface</h3>

This is the interface that an object should implement so that it can be used by the Apollo Client to make queries.

- `query(request: GraphQLRequest): Promise<GraphQLResult>` This function on your network interface is pretty self-explanatory - it takes a GraphQL request object, and should return a promise for a GraphQL result. The promise should be rejected in the case of a network error.

<h3 id="GraphQLRequest">interface GraphQLRequest</h3>

Represents a request passed to the network interface. Has the following properties:

- `query: string` The query to send to the server.
- `variables: Object` The variables to send with the query.
- `debugName: string` An optional parameter that will be included in error messages about this query. XXX do we need this?

<h3 id="GraphQLResult">interface GraphQLResult</h3>

This represents a result that comes back from the GraphQL server.

- `data: any` This is the actual data returned by the server.
- `errors: Array` This is an array of errors returned by the server.

<h2 id="query-batching">Query batching</h2>

Apollo Client can automatically batch multiple queries into one request when they are done within a 10 millisecond interval. This means that if you render several components, for example a navbar, sidebar, and content, and each of those do their own GraphQL query, they will all be sent in one roundtrip. This batching supports all GraphQL endpoints, including those that do not implement transport-level batching, by merging together queries into one top-level query with multiple fields.

To turn on query batching, pass the `shouldBatch: true` option to the `ApolloClient` constructor. In addition, you need to use a network interface that supports batching. The default network interface generated by `createNetworkInterface` supports batching out of the box, but you can convert any network interface to a batching one by calling `addQueryMerging` on it, as described [below](#addQueryMerging).

<h3 id="BatchingExample">Batching example</h3>

Batching is supported in the default network interface, so to turn it on normally you just have to pass `shouldBatch: true` to the constructor:

```javascript
import ApolloClient from 'apollo-client';

const apolloClient = new ApolloClient({
  shouldBatch: true,
});

// These two queries happen in quick suggestion, possibly in totally different
// places within your UI.
apolloClient.query({ query: firstQuery });
apolloClient.query({ query: secondQuery });

// You don't have to do anything special - Apollo will send the two queries as one request.
```

If you have developed a custom network interface, you can easily add batching via query merging to it with `addQueryMerging`:

```js
import { myCustomNetworkInterface } from './network';
import { addQueryMerging } from 'apollo-client';

const networkInterface = addQueryMerging(myCustomNetworkInterface);

const apolloClient = new ApolloClient({
  networkInterface,
  shouldBatch: true,
});

// Now queries will be batched
```

<h3 id="addQueryMerging" title="addQueryMerging">addQueryMerging(networkInterface)</h3>

```js
import { addQueryMerging } from 'apollo-client';

const batchingNetworkInterface = addQueryMerging(myCustomNetworkInterface);
```

This function takes an arbitrary `NetworkInterface` implementation and returns a `BatchedNetworkInterface` that batches queries together using query merging. You don't need to use this on the standard HTTP network interface generated by `createNetworkInterface`.

<h3 id="QueryMerging">How query merging works</h3>

Apollo can provide batching functionality over a standard `NetworkInterface` implementation that does not typically support batching. It does this by merging queries together under one root query and then unpacking the merged result returned by the server.

It's easier to understand with an example. Consider the following GraphQL queries:

```graphql
query someAuthor {
  author {
    firstName
    lastName
  }
}
```

```graphql
query somePerson {
  person {
    name
  }
}
```

If these two queries are fired within one batching interval, Apollo client will internally merge these two queries into the following:

```graphql
query __composed {
  ___someAuthor___requestIndex_0__fieldIndex_0: author {
    firstName
    lastName
  }

  ___somePerson___requestIndex_1__fieldIndex_0: person {
    name
  }
}
```

Once the results are returned for this query, Apollo will take care of unpacking the merged result and present your code with data that looks like the following:

```json
{
  "data": {
    "author": {
      "firstName": "John",
      "lastName": "Smith"
    }
  }
}
```

```json
{
  "data": {
    "person": {
      "name": "John Smith"
    }
  }
}
```

This means that your client code and server implementation can remain completely oblivious to the batching that Apollo performs.

<h3 id="BatchedNetworkInterface" title="BatchedNetworkInterface">interface BatchedNetworkInterface</h3>

If you want to implement a network interface that natively supports batching, for example by sending queries to a special endpoint that can handle multiple operations, you can do that by implementing a special method in your network interface, in addition to the normal `query` method:

- `batchQuery(request: GraphQLRequest[]): Promise<GraphQLResult[]>` This function on a batched network interface that takes an array of GraphQL request objects, submits a batched request that represents each of the requests and returns a promise. This promise is resolved with the results of each of the GraphQL requests. The promise should be rejected in case of a network error.
