# README

This was originally posted [on Medium](http://medium.com). If you're cloning the repo, run `npm install` and then `npm start`.

# Relay 101: Building A Hacker News Client

[React](https://facebook.github.io/react/) lets you build user interface components with JavaScript; [Relay](https://facebook.github.io/relay/) lets you easily connect your React components to data from a remote server. Relay accomplishes this by being opinionated - it assumes certain things about your server and your app, which increases the barrier to entry but may be worth it for many projects.

Without Relay, you have to manually download, transform, and cache each slice of your server's data in your React app. Tools like Flux and Redux help prevent some bugs that occur in this process, but still leave considerable room for human error in apps with lots of data flowing to and from the server. Relay removes most of the common boilerplate and enables app engineers to concisely and safely retrieve the data they want.

Once upon a time, Rails showed how to make a blog in 15 minutes. In that tradition, we're going to make a Hacker News client with Relay. This assumes you are familiar with Node, NPM, and React, but nothing more.

## Getting GraphQL

Currently Relay requires that your server expose a [GraphQL](https://facebook.github.io/graphql/) endpoint. GraphQL is very neat, but unless you work at Facebook you probably don't such an endpoint handy.

Instead of making our own, we're going to use [GraphQLHub](http://www.graphqlhub.com/)'s GraphQL endpoint. GraphQLHub is a burgeoning repository of GraphQL translations of existing APIs, such as the APIs for Hacker News and Reddit - I also happen to maintain it :)

This little tutorial will bring you up to speed on basic GraphQL syntax, and you definitely don't need to read the specification before starting. If you're curious about writing a GraphQL endpoint, check out _[Your First GraphQL Server](https://medium.com/@clayallsopp/your-first-graphql-server-3c766ab4f0a2)_ sometime.

## Setting Up The Project

In 2015, there is a glut of tools used to create JavaScript apps in the browser. The ones we're going to use today are [Webpack](https://webpack.github.io/), to bundle our code into something the browser understands, and [Babel](https://babeljs.io/), to compile our React and Relay code into something Webpack understands. These tools are endorsed by the Relay team, but it's also possible to use Relay without them in your other projects.

We're going to intentionally breeze through a lot of the Webpack and Babel setup, since we're here to talk about Relay - feel free to drop a Note on any part if you want more information.

Let's start by making a new Node project:

```
$ mkdir relay-101 && cd ./relay-101
$ npm init
# you can hit Enter a bunch of times
```

This will create a package.json file in your directory with some pre-populated information. Time to start installing some packages:

```
$ npm install webpack@1.12.2 webpack-dev-server@1.11.0 babel-core@5.8.25 babel-loader@5.3.2 --save
```

Webpack looks for a configuration file called `webpack.config.js`, so we should make that in the same directory:

```
$ touch webpack.config.js
```

And then paste this into it:

```
var path = require('path');

module.exports = {
  entry: path.resolve(__dirname, 'index.js'),
  module: {
    loaders: [
      {
        test: /\.js$/,
        loader: 'babel',
        query: {stage: 0}
      }
    ]
  },
  output: {filename: 'index.bundle.js', path: './'}
};
```

You'll notice that we're referring to an index.js file in there. Go ahead and create that with something really simple:

```
$ echo 'alert("hello relay!");' > index.js
```

As we continue, all of our app's code will go into that file, so keep it handy in your editor.

We've still got a few more steps - in the package.json file, add a "start" entry to your scripts:

```
{
  ...
  "scripts": {
    "start": "./node_modules/.bin/webpack-dev-server",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
}
```

What this does is allow us to type "npm start" and have it run the Webpack development server we installed earlier. Give it a shot and leave it running in its own terminal session/tab:

```
$ npm start
> relay-101-test@1.0.0 start ~/relay-101
> webpack-dev-server

http://localhost:8080/webpack-dev-server/
webpack result is served from /
```

Open [http://localhost:8080/webpack-dev-server](http://localhost:8080/webpack-dev-server) and see - a list of files! Which is good, but what we need is some HTML for our app. Back in your project folder, create index.html and fill it with some content:

```
$ touch index.html

# paste this inside of index.html
<html>
<head></head>
<body>
  <div id='container'>
  </div>
  <script src="/index.bundle.js" charset="utf-8"></script>
</body>
</html>
```

Refresh the dev server, and see the expected pop-up:

![popup](popup.png)

Now the fun can start.

## Building A Component

Our little app is going to mimic the [Hacker News](https://news.ycombinator.com) front page, and our first UI component is what each individual post will look like.

### Static Data

To start creating the component, we need to install the React and React-DOM packages:

```
$ npm install react@0.14.0-rc1 react-dom@0.14.0-rc1 --save
```

Note that we have very specific version requirements. Back in index.js, remote our old alert and start by defining an Item component:

```
// inside index.js

let React    = require('react');
let ReactDOM = require('react-dom');

class Item extends React.Component {
  render() {
    let item = this.props.store.item;

    return (
      <div key={item.id}>
        <h1><a href={item.url}>{item.title}</a></h1>
        <h2>{item.score} - {item.by.id}</h2>
        <hr />
      </div>
    );
  }
};
```

Note that all of our props come from this "store" object - we'll see why in a moment, but go with it for now.

Let's get something on the screen - render a dummy Item like so:

```
// at the bottom of index.js

let mountNode = document.getElementById('container');
let item = {
  id  : '1337',
  url : 'http://google.com',
  title : 'Google',
  score : 100,
  by : { id : 'clay '}
};
let store = { item };
let rootComponent = <Item store={store} />;
ReactDOM.render(rootComponent, mountNode);
```

Refresh your dev server and you should see something like this:

![dummy](dummy.png)

### Data From The Server

Time to add some Relay. Instead of using a static item, we're going to fetch the item by its ID from GraphQLHub. Start by installing some Relay packages:

```
$ npm install react-relay@0.3.2 babel-relay-plugin@0.2.5 sync-request@2.0.1 graphql@0.4.4 --save
```

Why did we install more than just react-relay? Well, tThe current implementation of Relay is going to require us to do a bit more setup - specifically, we need to connect this "babel-relay-plugin" into Babel. The plugin will talk to the GraphQLHub endpoint and generate some more configuration for Relay.

To connect the plugin, open up webpack.config.js and edit the "query" option:

```
module.exports = {
  ...
  module: {
    loaders: [
      {
        ...,
        // note that this is different!
        query: {stage: 0, plugins: ['./babelRelayPlugin']}
      }
    ]
  }
  ...
};
```

This tells Babel to look for a plugin file called babelRelayPlugin.js. Create that file and copy-paste the boilerplate:

```
$ touch babelRelayPlugin.js

// inside that file
var babelRelayPlugin   = require('babel-relay-plugin');
var introspectionQuery = require('graphql/utilities').introspectionQuery;
var request            = require('sync-request');

var graphqlHubUrl = 'http://www.GraphQLHub.com/graphql';
var response = request('GET', graphqlHubUrl, {
  qs: {
    query: introspectionQuery
  }
});

var schema = JSON.parse(response.body.toString('utf-8'));

module.exports = babelRelayPlugin(schema.data, {
  abortOnError: true,
});
```

Cool - now kill your `npm start` process and restart it. Now every time your app re-bundles, it will query the GraphQLHub server (using GraphQL's super neat introspection API) and prepare our Relay code.

Back in index.js, time to finally import Relay:

```
let React    = require('react');
let ReactDOM = require('react-dom');
let Relay    = require('react-relay');
```

What now? We're going to wrap our Item component with a [higher-order component](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750#fd19) component. This higher-order component will be created and managed by Relay, which is where the magic happens:

```
class Item extends React.Component {
  ...
}
Item = Relay.createContainer(Item, {
  fragments: {
    store: () => Relay.QL`
      fragment on HackerNewsAPI {
        item(id: 8863) {
          title,
          score,
          url
          by {
            id
          }
        }
      }
    `,
  },
});
```

Boom, that happened. In plain-english, this is what's happening:

> Hey Relay, I'm going to re-define my Item component as a new component which wraps the original. For the component's `store` prop, I need the data described in this GraphQL fragment. I know I need it on a "HackerNewsAPI" because I explored the API via http://GraphQLHub.com/playground/hn.

Note that we only describe a GraphQL fragment (fragments are analogous to aliases/symlinks in a query), not the final query for how to pull the data. This is one of Relay's strengths - a component declares exactly what data it needs, not how to retrieve it.

But at some point we do need a finalized GraphQL query, which is where Relay Routes come into play. Relay.Route has nothing to do with browser history or URLs - instead, it has to do with creating a "root query," in which all of the various fragments by the initial set of components exist.

So, let's make a Relay Route. Add this below our new Item definition:

```
Item = ...;

class HackerNewsRoute extends Relay.Route {
  static routeName = 'HackerNewsRoute';
  static queries = {
    store: ((Component) => {
      // Component is our Item
      return Relay.QL`
      query root {
        hn { ${Component.getFragment('store')} },
      }
    `}),
  };
}
```

Note that our GraphQL now begins with the root query. Relay allows injection of fragments and variables, which is how components share (but do not duplicate) their data requirements to their parent components.

Time to get something on the screen! Change our old rendering code to this:

```
class HackerNewsRoute ... {

}

Relay.injectNetworkLayer(
  new Relay.DefaultNetworkLayer('http://www.GraphQLHub.com/graphql')
);

let mountNode = document.getElementById('container');
let rootComponent = <Relay.RootContainer
  Component={Item}
  route={new HackerNewsRoute()} />;
ReactDOM.render(rootComponent, mountNode);
```

The Relay RootContainer is the top-level component which kicks off a query with a component hierarchy. We do a bit of networking setup, and then render the new component into the DOM. You should see this in your browser:

![withdata](withdata.png)

## A List Of Components

We have something starting to resemble the front page of Hacker News. Instead of hard-coding one item, we need to show a list of the top items. In the Relay world, this pattern (a list) generally requires us to create a List component, which embeds many individual item components (each requesting their specific data).

In code, start by creating a new TopItems component:

```
class TopItems extends React.Component {
  render() {
    let items = this.props.store.topStories.map(
      store => <Item store={store} />
    );
    return <div>
      { items }
    </div>;
  }
}
```

We could go through the same "create mock data" exercise as earlier, but instead we're going to skip straight to wrapping TopItems with Relay:

```
TopItems = Relay.createContainer(TopItems, {
  fragments: {
    store: () => Relay.QL`
      fragment on HackerNewsAPI {
        topStories { ${Item.getFragment('store')} },
      }
    `,
  },
});
```

Now instead of requesting one item, we request the "topStories". For each story, GraphQL will request the data from the Item's fragment, so we'll get only the data we need.

But hang on - currently our Item fragment requests a specific item (#8836). We need to update our query to only be a fragment on individual HackerNewsItem objects:

```
Item = Relay.createContainer(Item, {
  fragments: {
    store: () => Relay.QL`
      fragment on HackerNewsItem {
        id
        title,
        score,
        url
        by {
          id
        }
      }
    `,
  },
});
```

And since we're no longer requesting an item in our fragment, we need to change how the prop access works in the render function:

```
class Item extends React.Component {
  render() {
    let item = this.props.store;
    // ...
  }
}
```

One last tweak - we need to change the Relay RootContainer to use our new TopItems component:

```
let rootComponent = <Relay.RootContainer
  Component={TopItems}
  route={new HackerNewsRoute()} />;
```

Voila! Check your app in the browser:

![feed](feed.png)

## Variables in Queries

So now we have the basic knowledge to start building Relay apps, but I want to show off one more Relay feature: variables.

In most apps, queries aren't static and we often need to request different data at runtime. One way Relay allows to accomplish this is to embed variables in our GraphQL queries. For our little app, we're going to change which types of stories we fetch (the top, or the newest, etc).

To start, we need to change our TopItems query:

```
TopItems = Relay.createContainer(TopItems, {
  initialVariables: {
    storyType: "top"
  },
  fragments: {
    store: () => Relay.QL`
      fragment on HackerNewsAPI {
        stories(storyType: $storyType) { ${Item.getFragment('store')} },
      }
    `,
  },
});
```

The dollar-sign-prefixed "storyType" denotes a GraphQL variable (note that this isn't an ES6 string interpolation). We give it an initial value of "top" via the initialVariables configuration, which lets our component render immediately.

That's the only Relay-level change we need to make, which is pretty sweet. We haven't changed anything related to how an individual component renders or requests data - that process is totally decoupled.

Now we need to edit our TopItems component rendering to account for switching story types. Update the render method to look like this:

```
class TopItems extends React.Component {
  render() {
    let items = this.props.store.stories.map(
      store => <Item store={store} />
    );
    let variables = this.props.relay.variables;

    // To reduce the perceived lag
    // There are less crude ways of doing this, but this works for now
    let currentStoryType = (this.state && this.state.storyType) || variables.storyType;

    return <div>
      <select onChange={this._onChange.bind(this)} value={currentStoryType}>
        <option value="top">Top</option>
        <option value="new">New</option>
        <option value="ask">Ask HN</option>
        <option value="show">Show HN</option>
      </select>
      { items }
    </div>;
  }

  // to be continued
```

Some new stuff going on here! We're now accessing the "relay" prop, which has some special properties. Any component created with Relay has this prop injected - if we wanted to just unit test our TopItems component, we could inject a mock object ourselves.

Aside from the Relay variables, everything else so far is vanilla React - we create a new select element, give it an initial value, and get ready to respond when it changes. When that change happens, we need to tell Relay to use a new variable value. This looks like:

```
class TopItems extends React.Component {
  render() {
    // ...
  }

  _onChange(ev) {
    let storyType = ev.target.value;
    this.setState({ storyType });
    this.props.relay.setVariables({
      storyType
    });
  }
}
```

It's that simple - Relay will detect what part of the query has changed and re-fetch as needed. We also set the local component state, which makes it feel a bit snappier.

Refresh your browser and you should be able to switch between stories with ease. If you switch between story types, you'll notice that Relay won't request new data if you've already loaded that particular feed.

![types](types.png)

## Relay 102

So, that's a whirlwind introduction to Relay. We haven't even touched on mutations (which is how you write data back to the server) or how to handle a loading spinner while data fetches. Relay may not be right for every app or team, but it's a very interesting take on a common problem that might help some lightbulbs go off.

The final source of this app is available [on Github](https://github.com/clayallsopp/relay-101). Follow me [@clayallsopp](http://twitter.com/clayallsopp) and/or [@GraphQLHub](http://twitter.com/GraphQLHub) for updates on more of this stuff.
