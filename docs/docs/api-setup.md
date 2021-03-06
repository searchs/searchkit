---
id: api-setup
title: API Setup
sidebar_label: API Setup
slug: /quick-start/api-setup
---

## Setup Apollo Server
First step is to setup Apollo Server within our app. We are using [Next](https://github.com/vercel/next.js/) & [Apollo Server Micro](https://www.apollographql.com/docs/apollo-server/v1/servers/micro/) where we will use [API routes](https://nextjs.org/docs/api-routes/introduction) for our app. 

Also see [AWS lambda example](https://searchkit.co/docs/examples/aws-lambda) if you're not using NextJS.

First install `apollo-server-micro`

``` yarn add apollo-server-micro ```

then create a `graphql.js` file in the pages/api folder

```javascript

import { ApolloServer, gql } from 'apollo-server-micro'

export const config = {
  api: {
    bodyParser: false
  }
}

const typeDefs = [
  gql`
    type Query {
      root: String
    }

    type Mutation {
      root: String
    }
  `
]

const server = new ApolloServer({
  typeDefs,
  resolvers: {},
  introspection: true,
  playground: true,
  context: {}
})

export default server.createHandler({ path: '/api/graphql' })

```

Once saved, you should be able to access the [graphql playground](http://localhost:3000/api/graphql) locally.

#### Dont use Next?
Apollo Docs provides how to [add Apollo Server](https://www.apollographql.com/docs/apollo-server/v1/servers/express/) to any Node based app such as express and AWS Lambda should you wish not to use Next.

## Add Searchkit Schema

Install the NPM module for searchkit

```yarn add @searchkit/apollo-resolvers```

then add searchkit schema and resolvers to apollo-server

```javascript

import { ApolloServer, gql } from 'apollo-server-micro'
import {
  MultiMatchQuery,
  SearchkitResolvers,
  SearchkitSchema
} from '@searchkit/apollo-resolvers'

const searchkitConfig = {
  host: 'http://localhost:9200',
  index: 'my_index',
  hits: {
    fields: []
  },
  query: new MultiMatchQuery({ fields: [] }),
  facets: []
}

const typeDefs = [
  gql`
    type Query {
      root: String
    }

    type Mutation {
      root: String
    }

    type HitFields {
    }
  `,
  SearchkitSchema
]

export const config = {
  api: {
    bodyParser: false
  }
}

const server = new ApolloServer({
  typeDefs,
  resolvers: {
    ...SearchkitResolvers(searchkitConfig)
  },
  introspection: true,
  playground: true,
  context: {}
})

export default server.createHandler({ path: '/api/graphql' })

```

Make sure you replace the host, index. This will setup a basic searchkit API and visiting the [API playground](http://localhost:3000/api/graphql), you should be able to do a simple query to make sure its connected to your elasticsearch instance correctly.

```gql
{
  results {
    hits {
      items {
        id
      }
    }
  }
}
```

![Setup GQL](../static/img/api-setup-1.jpg)

## Setup Hit Values

Next you want to configure the fields that come back for each field. For our IMDB search app, we want to show the title, writers, actors, plot and poster for each hit. First locate the `HitFields` type definition and add the fields that you want to display per hit item.

You will need to define the type per field. See [GraphQL types](https://graphql.org/learn/schema/#type-system) to learn more.

```gql
    type HitFields {
      title: String
      writers: [String]
      actors: [String]
      plot: String
      poster: String
    }
```

Now update your query in [graphql playground](http://localhost:3000/api/graphql) to include these hit fields

```gql
{
  results {
    hits {
      items {
        id
        fields {
          title
          writers
          actors
          plot
          poster
        }
      }
    }
  }
}

```

## Configuring Query

Next you want to configure the fields to search on when you provide a text query. To do this, locate the `MultiMatchQuery({ fields: [] })` and pass in the elasticsearch fields that you want to be queried on. 

For Searchkit Demo, we want to search the following fields: actors, writers, title and plot. We want title to be the most important field so we will boost its importance by 4. The configuration for this would be

```new MultiMatchQuery({ fields: ['actors', 'writers', 'title^4', 'plot'] })```

Once this has been setup, you should be able to use the query param in the GQL query.

```gql
{
  results(query: "heat") {
    hits {
      items {
        id
        fields {
          title
        }
      }
    }
  }
}

```

## Configuring Facets
Next we want to setup facets for our search. Facets are configured within the searchkit configuration. 

```javascript

const searchkitConfig = {
  host: 'http://localhost:9200',
  index: 'my_index',
  query: new MultiMatchQuery({ fields: [] }),
  facets: [
    new RefinementSelectFacet({ field: 'type.raw', id: 'type', label: 'Type' }),
    new RefinementSelectFacet({ field: 'writers.raw', id: 'writers', label: 'Writers' }),
    new RefinementSelectFacet({ field: 'actors.raw', id: 'actors', label: 'Actors' })
  ]
}

```

### Facets Response

Once configured, API will return all facets values for fields that have aggregations. 

```
{
  results(query: "heat") {
    facets {
      id
      label
      type
      entries {
        id
        label
        count
      }
    }
  }
}
```

See below the response in playground

![api-setup-2](../static/img/api-setup-2.jpg)

### Adding Filters

When the user chooses to filter by movies, you can add the filter like so

```gql
{
  results(query: "heat", filters: [{id: "type", value: "Movie"}]) {
    facets {
      id
      label
      type
      entries {
        id
        label
        count
      }
    }
  }
}
```

Searchkit will query elasticsearch to show only hit + facet results that have a type field value of movie.  



 




