# graphql-js-client

Feature light client library for fetching resources via GraphQL.

## Table Of Contents

- [Installation](#installation)
- [Examples](#examples)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [License](http://github.com/Shopify/graphql-js-client/blob/master/LICENSE.md)

## Installation
**With Yarn:**
```bash
$ yarn add graphql-js-client
```
**With NPM:**
```bash
$ npm install graphql-js-client
```

## Examples

`GraphQLClient` requires a "type bundle" which is a set of ES6 modules generated by [graphql-js-schema](https://github.com/Shopify/graphql-js-schema)
that represent your GraphQL schema.

#### Initialization

```javascript
import GraphQLClient from 'graphql-js-client';

// This is the generated type bundle from graphql-js-schema
import types from './types.js';

const client = new GraphQLClient(types, {
  url: 'https://graphql.myshopify.com/api/graphql',
  fetcherOptions: {
    headers: `Authorization: Basic aGV5LXRoZXJlLWZyZWluZCA=`
  }
});
```

#### Creating and sending a query

```javascript
const query = client.query((root) => {
  root.add('shop', (shop) => {
    shop.add('name');
    shop.addConnection('products', {args: {first: 10}}, (product) => {
      product.add('title');
    });
  });
});

/* Will generate the following query:

  query {
    shop {
      name
      products (first: 10) {
        pageInfo {
          hasNextPage
          hasPreviousPage
        }
        edges {
          cursor
          node {
            id
            title
          }
        }
      }
    }
  }

Note: things that implement Node will automatically have the id added to the
query. `addConnection` will automatically build out the connection query with
all information necessary for pagination.

*/

let objects;

client.send(query).then(({model, data}) => {
  objects = model;
  console.log(model); // The serialized model with rich features
  console.log(data); // The raw data returned from the API endpoint
});
```

#### Refetching Nodes

```javascript
client.refetch(objects.shop.products[0]).then((product) => {
  console.log(product); // This is a fresh instance of the product[0]
});
```

In the above example, `objects.shop.products[0]` implements the Node interface.
The client is smart enough to understand this, and generate the following query:

```javascript
query {
  node (id: 'abc123') {
    __typename
    ... on Product {
      id
      title
    }
  }
}
```

It then resolves directly with the `node` field, which is a product in this
case.


#### Pagination

```javascript
const query = client.query((root) => {
  root.add('shop', (shop) => {
    shop.add('name');
    shop.addConnection('products', {args: {first: 10}}, (product) => {
      product.add('title');
      product.addConnection('variants', {args: {first: 50}}, (variant) => {
        variant.add('title');
        variant.add('price');
      });
    });
  });
});

client.send(query).then(({model}) => {
  client.fetchNextPage(model.shop.products).then((products) => {
    console.log(products); // resolves with the next 10 products.
  });

  client.fetchNextPage(model.shop.products[0].variants).then((variants) => {
    console.log(variants); // resolves with the next 50 variants. Page size is
                           // taken from the previous `first` argument.
  });
});
```

The client understands the [Relay](https://facebook.github.io/relay/graphql/connections.htm)
specification, and will send the following query for the call
`client.fetchNextPage(model.shop.products)`:

```graphql
query {
  shop {
    name
    products (first: 10, after: 'abc123') {
      pageInfo {
        hasNextPage
        hasPreviousPage
      }
      edges {
        cursor
        node {
          id
          title
          variants (first: 50) {
            pageInfo {
              hasNextPage
              hasPreviousPage
            }
            edges {
              cursor
              node {
                id
                title
                price
              }
            }
          }
        }
      }
    }
  }
}
```

The client can also use the Node interface to optimize queries for nested
paginated sets. `fetchNextPage` in the second case
`client.fetchNextPage(model.shop.products[0].variants)` will generate the
following query:

```graphql
query {
  node (id: '1') {
    __typename
    ... on Product {
      id
      variants (first: 50, after: 'abc123') {
        id
        title
        price
      }
    }
  }
}
```

In both cases, `fetchNextPage` resolves with the models you're paginating, since
the object graph to those models may not be obvious due to the query generation
algorithm. Page size and fields on the paginated object are retained, while
fields not in the paginated set are pruned.

## Documentation

For full API documentation, check out the [API docs](https://shopify.github.io/graphql-js-client).

## Contributing

#### Setting up:

```bash
$ git clone git@github.com:Shopify/graphql-js-client.git
$ cd graphql-js-client
$ yarn install
```

#### Running the tests in a browser

```bash
$ yarn start
```

Then visit [http://localhost:4200](http://localhost:4200)

#### Running the tests in node

```bash
$ yarn test
```

## License

MIT, see [LICENSE.md](http://github.com/Shopify/graphql-js-client/blob/master/LICENSE.md) for details.

<img src="https://cdn.shopify.com/shopify-marketing_assets/builds/19.0.0/shopify-full-color-black.svg" width="200" />
