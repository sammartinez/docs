---
title: Make your data searchable
description: The @searchable directive handles streaming the data of an @model object type to Amazon Elasticsearch Service and configures search resolvers that search that information.
---

## @searchable

The `@searchable` directive handles streaming the data of an `@model` object type to Amazon Elasticsearch Service and configures search resolvers that search that information.

> **Migration warning**: You might observe duplicate records on search operations, if you deployed your GraphQL schema using CLI version older than 4.14.1 and have thereafter updated your schema & deployed the changes with a CLI version between 4.14.1 - 4.16.1.
Please use this Python [script](https://github.com/aws-amplify/amplify-cli/blob/master/packages/graphql-elasticsearch-transformer/scripts/ddb_to_es.py) to remove the duplicate records from your Elasticsearch cluster. [This script](https://github.com/aws-amplify/amplify-cli/blob/master/packages/graphql-elasticsearch-transformer/scripts/ddb_to_es.py) indexes data from your DynamoDB Table to your Elasticsearch Cluster. View an example of how to call the script with the following parameters [here](https://aws-amplify.github.io/docs/cli-toolchain/graphql#example-of-calling-the-script).

> **Billing warning**: `@searchable` incurs an additional cost depending on instance size. For more information refer to [ElasticSearch service pricing](https://aws.amazon.com/elasticsearch-service/pricing/)

### Definition

```graphql
# Streams data from DynamoDB to Elasticsearch and exposes search capabilities.
directive @searchable(queries: SearchableQueryMap) on OBJECT
input SearchableQueryMap { search: String }
```

### Usage

Given the following schema an index is created for Post, if there are more types with `@searchable` the directive will create an index for it, and those posts in Amazon DynamoDB are automatically streamed to the post index in Amazon ElasticSearch via AWS Lambda and connect a searchQueryField resolver.

```graphql
type Post @model @searchable {
  id: ID!
  title: String!
  createdAt: String!
  updatedAt: String!
  upvotes: Int
}
```

You may then create objects in DynamoDB that will be automatically streamed to lambda
using the normal `createPost` mutation.

```graphql
mutation CreatePost {
  createPost(input: { title: "Stream me to Elasticsearch!" }) {
    id
    title
    createdAt
    updatedAt
    upvotes
  }
}
```

And then search for posts using a `match` query:

```graphql
query SearchPosts {
  searchPosts(filter: { title: { match: "Stream" }}) {
    items {
      id
      title
    }
  }
}
```

There are multiple `SearchableTypes` generated in the schema, based on the datatype of the fields you specify in the Post type.

The `filter` parameter in the search query has a searchable type field that corresponds to the field listed in the Post type. For example, the `title` field of the `filter` object, has the following properties (containing the operators that are applicable to the `string` type):

* `eq` - which uses the Elasticsearch keyword type to match for the exact term.
* `ne` - this is the inverse operation of `eq`.
* `matchPhrase` - searches using the Elasticsearch's [Match Phrase Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/query-dsl-match-query-phrase.html) to filter the documents in the search query.
* `matchPhrasePrefix` - This uses the Elasticsearch's [Match Phrase Prefix Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/query-dsl-match-query-phrase-prefix.html) to filter the documents in the search query.
* `multiMatch` - Corresponds to the Elasticsearch [Multi Match Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/query-dsl-multi-match-query.html).
* `exists` - Corresponds to the Elasticsearch [Exists Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/query-dsl-exists-query.html).
* `wildcard` - Corresponds to the Elasticsearch [Wildcard Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/query-dsl-wildcard-query.html).
* `regexp` - Corresponds to the Elasticsearch [Regexp Query](https://www.elastic.co/guide/en/elasticsearch/reference/6.2/query-dsl-regexp-query.html).

The `sort` parameter can be used to specify the order of the search results, can be ascending (`asc`) or descending (`desc`), if not specified ascending order is used.

The `limit` parameter controls the number of search results returned. If not specified the default value is 100.

For example, you can filter using the wildcard expression to search for posts using the following `wildcard` query:

```graphql
query SearchPosts {
  searchPost(filter: { title: { wildcard: "S*Elasticsearch!" }}) {
    items {
      id
      title
    }
  }
}
```

The above query returns all documents whose `title` begins with `S` and ends with `Elasticsearch!`.

Moreover you can use the `filter` parameter to pass a nested `and`/`or`/`not` condition. By default, every operation in the filter properties is *AND* ed. You can use the `or` or `not` properties in the `filter` parameter of the search query to override this behavior. Each of these operators (`and`, `or`, `not` properties in the filter object) accepts an array of searchable types which are in turn joined by the corresponding operator. For example, consider the following search query:

```graphql
query SearchPosts {
  searchPost(filter: {
    title: { wildcard: "S*" }
    or: [
      { createdAt: { eq: "08/20/2018" } },
      { updatedAt: { eq: "08/20/2018" } }
    ]
  }) {
    items {
      id
      title
    }
  }
}
```

Assuming you used the `createPost` mutation to create new posts with `title`, `createdAt` and `updatedAt` values, the above search query will return you a list of all `Posts`, whose `title` starts with `S` _and_ have `createdAt` _or_ `updatedAt` value as `08/20/2018`.

Here is a complete list of searchable operations per GraphQL type supported as of today:

| GraphQL Type        | Searchable Operation           |
|-------------:|:-------------|
| String      | `ne`, `eq`, `match`, `matchPhrase`, `matchPhrasePrefix`, `multiMatch`, `exists`, `wildcard`, `regexp` |
| Int     | `ne`, `gt`, `lt`, `gte`, `lte`, `eq`, `range`      |
| Float | `ne`, `gt`, `lt`, `gte`, `lte`, `eq`, `range`      |
| Boolean | `eq`, `ne`      |

### Known limitations

- `@searchable` is not compatible with DataStore but you can use it with the API category.
- `@searchable` is not compatible with Amazon ElasticSearch t2.micro instance as it only works with ElasticSearch version 1.5 and 2.3 and Amplify CLI only supports instances with ElasticSearch version >= 6.x.
- `@searchable` is not compatible with the @connection directive
- Support for adding the `@searchable` directive does not yet provide automatic indexing for any existing data to Elasticsearch. View the feature request [here](https://github.com/aws-amplify/amplify-cli/issues/98).

### Backfill your Elasticsearch index from your DynamoDB table

The following Python [script](https://github.com/aws-amplify/amplify-cli/blob/master/packages/graphql-elasticsearch-transformer/scripts/ddb_to_es.py) creates an event stream of your DynamoDB records and sends them to your Elasticsearch Index. This will help you backfill your data should you choose to add `@searchable` to your @model types at a later time.

**Example of calling the script**:

```bash
python3 ddb_to_ess.py \
  --rn 'us-west-2' \ # Use the region in which your table and elasticsearch domain reside
  --tn 'Post-XXXX-dev' \ # Table name
  --lf 'arn:aws:lambda:us-west-2:123456789xxx:function:DdbToEsFn-<api__id>-dev' \ # Lambda function ARN
  --esarn 'arn:aws:dynamodb:us-west-2:123456789xxx:table/Post-<api__id>-dev/stream/2019-20-03T00:00:00.350' # Event source ARN
```
