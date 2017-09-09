# mobx-apollo

# Installation
`yarn add mobx mobx-apollo`

# Usage
```javascript
import { query } from 'mobx-apollo';

type options = {
  client: apolloClientInstance, // new ApolloClient()
  query: gqlInstance, // gql`..`
  onError?: Function,
  onFetch?: Function, // invoked every time new data is fetched
  ...watchQueryOptions // (see Apollo Client docs)
};

class Store {
  @query allPosts = { ...options };

  // or without decorators
  constructor() {
    query(this, 'allPosts', { ...options });
  }
}

const store = new Store();

autorun(() => console.log(store.allPosts.data)); // [{ title: 'Hello World!' }]
```

## Example
```javascript
import React, { Component } from 'react';

import ApolloClient, { createNetworkInterface } from 'apollo-client';
import gql from 'graphql-tag';
import { inject, observer, Provider } from 'mobx-react';
import { query } from 'mobx-apollo';

// schema built with graphcool
// type Post implements Node {
//   createdAt: DateTime!
//   id: ID! @isUnique
//   updatedAt: DateTime!
//   title: String!
// }

// fragments
const postDetails = gql`
  fragment postDetails on Post {
    id
    title
  }
`;

// queries
const allPostsQuery = gql`
  {
    allPosts(orderBy: createdAt_DESC) {
      ...postDetails
    }
  }
  ${postDetails}
`;

// mutations
const createPostMutation = gql`
  mutation createPost($title: String!) {
    createPost(title: $title) {
      ...postDetails
    }
  }
  ${postDetails}
`;

// initializing an apollo client instance
const client = new ApolloClient({
  networkInterface: createNetworkInterface({
    uri: 'https://api.graph.cool/simple/v1/<project>',
    dataIdFromObject: o => o.id
  })
});

// building a mobx store
const postsStore = new class {
  @query allPosts = { client, query: allPostsQuery };

  createPost = title =>
    client.mutate({
      mutation: createPostMutation,
      variables: { title },
      refetchQueries: [{ query: allPostsQuery }]
    });
}();

// our main component
@inject('postsStore')
@observer
class Example extends Component {
  componentDidMount() {
    setTimeout(
      () =>
        this.props.postsStore
          .createPost('Hello World!')
          .catch(error => console.log('Error', error.message)),
      2500
    );
  }

  render() {
    const { allPosts } = this.props.postsStore;

    if (allPosts.error) console.log('Error', allPosts.error);
    else if (allPosts.loading) console.log('Loading ..');
    else if (allPosts.data.length > 0) console.log('Data', allPosts.data);
    else console.log('No data.');

    return null;
  }
}

// typically you would have multiple mobx stores here
const stores = { postsStore };

const ExampleContainer = () => (
  <Provider {...stores}>
    <Example />
  </Provider>
);

export default ExampleContainer;
```
