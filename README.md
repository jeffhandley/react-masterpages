# React-Composition

Composing independently-interactive, universal-rendering, flux/redux-driven React components together is hard work.  React-Composition is here to help.

## Use Cases

There are countless scenarios for when you want to compose independent React components together, but here are just a few salient ones:

1. Page Layouts (a.k.a. master pages)
1. Application navigation components
1. Other common components that render on many pages
1. Dashboard screens that comprise many independent sections
1. Large, complex single-page applications that are broken down into smaller units

While small projects can choose a single interaction model and integrate everything together, larger projects cannot.  Instead, components need to be built with independence and composability in mind.

Consider a few of the following reasons why the interaction models of the above components might diverge:

1. Different teams responsible for different components have differing preferences for their Flux implementation
1. Different teams are updating their components and flux implementations on different schedules
1. A hot new Flux implementation (or other interaction model) comes along and teams want to start using it
1. You simply have the goal of keeping the separate concerns of the large application decoupled from one another (as you should)
1. Some components support universal rendering, some don't need to (many dashboard components take a long time to fetch data and therefore don't support universal rendering)

With all of the above in mind, React-Composition provides useful components and defines a simple model for composing React-based modules together, with support for universal rendering where you want it.

## Composition

Let's think through how a page is constructed:

The outer "shell" of an application needs to be rendered as static HTML markup--this will be the `<html>`, `<head>`, and `<body>` elements.

Within the `<body>`, we must support a composition of any number of independent components, each into their own container element.

Components rendered into these containers often need to feed information out to the shell for elements to be rendered into the `<head>` element of the page.

Components that have client-side interactions need to emit initial state data as well as client JavaScript to operate the interactions and re-render.

## Programming Model

By taking multiple Flux implementations into account and studying their similarities, we can extract a common programming model for both the server and client.

### Server-Side Programming Model

In order to render a React component on the server, we end up taking 2 steps:

1. Invoke the initial "page load" action that will (asynchronously) load the data needed for the component into the store.
2. Render the React component (using `ReactDOMServer.renderToString()`)

The "page load" action could actually be a combination of multiple async operations to fetch whatever data is needed--this varies drastically from component to component.  And there are of course many scenarios where all data needed is available without making any async calls.

We can boil this work down into a simple API signature:

``` js
function loadPage(req, callback)
```

#### Parameters

##### req
The HTTP Request object.

##### callback
Function to execute when any async loading is done, taking the following parameters:

**Page**
The React component (not an element) to be rendered as static HTML, representing the output page.

**actions**
Any action creators that need to be exposed on the component's public API--connected to any Flux implementation the component uses.

### Client-Side Programming Model

The server rendering of the page, from the component exposed in the `loadPage` callback, can include `<script>` tags that drive client-side interactions and rendering.  For client-side rendering to get bootstrapped for any component, the following steps need to be taken:

1. Load the components initial state into the store, using an object that was serialized within the rendered page
1. Render the component (causing zero flicker on first client-side render)

The client-side programming model is simpler than the server-side model because we don't need to worry as much about asynchronously rendering from the server.  Client-side bundles can be much more autonomous, but some patterns do still emerge.

``` js
function initialize(state, callback)
```

#### Parameters

##### state
The state object that was serialized from the server

##### callback
Function to execute when the state has been loaded and the React component has been re-rendered on the client, taking the following parameters:

**actions**
Any action creators that need to be exposed on the component's public API--connected to any Flux implementation the component uses.

## Server Example

For our example, we'll look first at the Express-driven server that executes and renders a page.  Because the programming model is so simple, this  should work well with the routing implementation of your choice.

``` jsx
import express from 'express';
import React from 'react';
import ReactDOMServer from 'react-dom/server';

const app = express();

app.use('/client', express.static('lib/client'));

app.get('/hello', (req, res, next) => {
    const { default: loadPage } = require('./pages/hello/server');

    loadPage(req, (Page, pageActions) => {
        res.send(ReactDOMServer.renderToStaticMarkup(<Page />));
    });
});

const server = app.listen(3000, () => {
    console.log('Listening on port 3000');
});
```

It is important to note that the `loadPage` function was imported from a server-specific module for the hello page.  Examining the `hello/server.js` module, we can see that it's very straight-forward.

``` jsx
import React from 'react';

export default (req, callback) => {
    callback(
        React.createClass({
            render() {
                return (
                    <html>
                        <head>
                            <title>Hello World</title>
                        </head>
                        <body>
                            Hello World
                        </body>
                    </html>
                );
            }
        })
    );
}
```

## Serializing State

The next step is extending our server example to support universal rendering.  We'll modify the `hello/server.js` file to create some initial state from the querystring and include that state in the rendered page.

``` jsx
import React from 'react';
import url from 'url';

const Hello = React.createClass({
    propTypes: {
        to: React.PropTypes.string.isRequired
    },

    render() {
        return (
            <span>Hello {this.props.to}</span>
        );
    }
});

export default (req, callback) => {
    const { to = "World" } = url.parse(req.url, true).query;
    const state = { hello: { to }};
    const renderState = `window.RenderState = ${JSON.stringify(state)};`;

    callback(
        React.createClass({
            render() {
                return (
                    <html>
                        <head>
                            <title>Hello World</title>
                        </head>
                        <body>
                            <Hello to={to} />
                            <script dangerouslySetInnerHTML={{ __html: renderState }} />
                        </body>
                    </html>
                );
            }
        })
    );
}
```

Running this example, we see the rendered HTML on the page as:

``` html
<html>
<head>
  <title>Hello World</title>
</head>
<body><span>Hello World</span>
  <script>
    window.RenderState = {
      "hello": {
        "to": "World"
      }
    };
  </script>
</body>
</html>
```

Changing the `to` parameter on the querystring changes whom the page says hello to.

### Universal Rendering

Adding universal rendering does require a couple of steps.  These steps will likely plug into your existing client bundling system.

We'll start by extracting the Hello component out into a separate module.

``` jsx
import React from 'react';

export default React.createClass({
    propTypes: {
        to: React.PropTypes.string.isRequired
    },

    render() {
        return (
            <span>Hello {this.props.to}</span>
        );
    }
});
```

Then we'll add a `hello/client.js` file that will be the entry point for our client-side bundle for this page.  You will need to execute webpack against this entry point as part of your own infrastructure.

``` jsx
import React from 'react';
import ReactDOM from 'react-dom';
import Hello from './Hello';

const state = window.RenderState.hello;
const container = document.getElementById('hello-container');

ReactDOM.render(
    <Hello to={state.to} />,
    container
);
```

Lastly, we'll circle back to the `hello/server.js` file to update it.  We need to utilize the extracted `<Hello>` component, wrap it in a `<div id="hello-container">`, and then include a `<script>` tag for rendering the client bundle.

``` jsx
import React from 'react';
import url from 'url';
import Hello from './Hello';

export default (req, callback) => {
    const { to = "World" } = url.parse(req.url, true).query;
    const state = { hello: { to }};
    const renderState = `window.RenderState = ${JSON.stringify(state)};`;

    callback(
        React.createClass({
            render() {
                return (
                    <html>
                        <head>
                            <title>Hello World</title>
                        </head>
                        <body>
                            <div id='hello-container'>
                                <Hello to={to} />
                            </div>
                            <script dangerouslySetInnerHTML={{ __html: renderState }} />
                            <script src='/client/pages/hello.js' />
                        </body>
                    </html>
                );
            }
        })
    );
}
```

This pattern is now performing universal rendering of a React component.  Because the server programming model is asynchronous, the implementation of the `loadPage` function in the server module could:

* Instantiate a Flux loop
* Execute async flux actions
* Fetch data from remote sources

React-Composition does not prescribe details for how to implement those concerns, as you can rely on your preferred Flux implementation and patterns.

Similarly, on the client side, your client entry point could also easily connect your React component to your Flux loop after loading the state from the page.

## React-Composition Utilities

With the base functionality in place, it's time to introduce the React-Composition utilities that can make this work even better.

You might have noticed the glaring issue with the above example: **the entire page is rendered as static HTML, which means the initial client-side React rendering will be inefficient (and potentially cause flicker).**

To address this, the `<Hello>` component needs to be emitted using the `ReactDOMServer.renderToString` method instead of being lumped in with the rest of the page that gets emitted through `renderToStaticMarkup`.

### RenderContainer

This can be accomplished using React-Composition's `RenderContainer` component.  A `RenderContainer` will render its contents using `renderToString()`, even when it is being rendered with `renderToStaticMarkup()`.

`RenderContainer` does more though.  It will also emit the serialized state for the component to be rendered inside the container, along with the `<script>` tag for the client-side bundle.

But importantly, `RenderContainer` does not emit the `<script>` tags directly--it instead uses `react-side-effect` build up the necessary state and clients that must be rendered as part of a page template.

### Page Templates

Page templates are an essential part of React-Composition.  Page templates are how the outer HTML shell of the page is decoupled from every page instance in the application, while allowing each page to select the applicable template.

Page templates have the responsibility of capturing the sets of render states and clients and emitting the `<script>` tags in the desired place within the HTML.

### Creating a Page Template

Let's update the `hello/server.js` module to use a `RenderContainer` and define its own page template.

``` jsx
import React from 'react';
import ReactDOMServer from 'react-dom/server';
import url from 'url';
import Hello from './Hello';
import RenderContainer from '../../components/RenderContainer';

const loadTemplate = (req, callback) => {
    callback(
        React.createClass({
            propTypes: {
                children: React.PropTypes.element.isRequired
            },
            render() {
                const body = ReactDOMServer.renderToStaticMarkup(this.props.children);
                const container = RenderContainer.rewind();

                return (
                    <html>
                        <head>
                            <title>Hello World</title>
                        </head>
                        <body>
                            <div dangerouslySetInnerHTML={{ __html: body }} />
                            <container.Component />
                        </body>
                    </html>
                );
            }
        })
    );
}

export default (req, callback) => {
    const { to = "World" } = url.parse(req.url, true).query;
    const state = { hello: { to }};

    loadTemplate(req, (Template) => {
        callback(
            React.createClass({
                render() {
                    return (
                        <Template>
                            <RenderContainer
                              clientSrc='/client/pages/hello.js'
                              id='hello-container'
                              state={state}>
                                <Hello to={to} />
                            </RenderContainer>
                        </Template>
                    );
                }
            })
        );
    });
}
```

As you can see, the `loadTemplate` function is defined using the same signature as the page itself--following this pattern makes it extremely easy to have page templates populate their own data from the request, perform async operations when necessary, and return React components to be used as containers wrapped around pages.

We can easily extract the `loadTemplate` function out into its own module.  Then, if the page template itself needs to have its own universally-rendered components, it could wrap them in `RenderContainer` elements and wrap itself in a higher-order template.  The calling page would be completely unaware of this factoring.

#### Multiple Page Sections

Many page templates will need to have multiple sections, rather than just a single child.  This can be easily achieved by defining props on the page template component as React elements.

Below, the `loadTemplate` function seen above has been refactored to accept a body and a footer rather than simply children.  It's also using a helpful `renderSections` function from `RenderContainer`.

``` jsx
import React from 'react';
import url from 'url';
import Hello from './Hello';
import RenderContainer from '../../components/RenderContainer';

export default (req, callback) => {
    callback(
        React.createClass({
            propTypes: {
                body: React.PropTypes.element.isRequired,
                footer: React.PropTypes.element
            },
            render() {
                const container = RenderContainer.renderSections(this.props);

                return (
                    <html>
                        <head>
                            <title>Hello World</title>
                        </head>
                        <body>
                            <container.sections.body />
                            <container.state />
                            <container.clients />
                            <hr />
                            <container.sections.footer />
                        </body>
                    </html>
                );
            }
        })
    );
}
```

The `hello/server.js` module has been updated to use these body and footer props.

``` jsx
import React from 'react';
import ReactDOMServer from 'react-dom/server';
import url from 'url';
import Hello from './Hello';
import loadTemplate from './template';

export default (req, callback) => {
    const { to = "World" } = url.parse(req.url, true).query;
    const state = { hello: { to }};

    loadTemplate(req, (Template) => {
        callback(
            React.createClass({
                render() {
                    return (
                        <Template
                            body={
                                <RenderContainer
                                  clientSrc='/client/pages/hello.js'
                                  id='hello-container'
                                  state={state}>
                                    <Hello to={to} />
                                </RenderContainer>
                            }
                            footer={
                                <span>It's nice to see you again!</span>
                            }
                        />
                    );
                }
            })
        );
    });
}
```

#### Clean Output

With the `RenderContainer` in place and being used by both the page and the template, we get very clean univerally-rendering pages.  Here's the (beautified) HTML output we now see.

``` html
<html>
<head>
  <title>Hello World</title>
</head>
<body>
  <div>
    <div>
      <div id="hello-container"><span data-reactid=".upsep0qubk" data-react-checksum="1638870632"><span data-reactid=".upsep0qubk.0">Hello </span><span data-reactid=".upsep0qubk.1">Jeff</span></span>
      </div>
      <noscript></noscript>
      <noscript></noscript>
    </div>
  </div>
  <script>
    window.RenderState = {
      "hello-container": {
        "hello": {
          "to": "Jeff"
        }
      }
    };
  </script>
  <div>
    <script src="/client/pages/hello.js"></script>
  </div>
  <hr/>
  <div><span>It&#x27;s nice to see you again!</span></div>
</body>
</html>
```

## Summary

The patterns set forth by react-composition are very straight-forward:

1. Server-side modules export functions that accept the request and a callback.  The callback will receive a React component and optionally action creators that the consumer can invoke.
1. The server-side React components use the `RenderContainer` component to wrap around any universally-rendered components and their flux implementations.
1. Those components wrap themselves in page templates that render the outer `<html>` skeleton and consume the state and clients from the `RenderContainer`.
1. Client-side entry points load the component state and initialize the flux loop and rendering

With this approach, we gain highly-composable, universally-rendering React components and flexible page templates.
