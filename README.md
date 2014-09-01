# Oasis.js

[![Build Status](https://secure.travis-ci.org/tildeio/oasis.js.png?branch=master)](http://travis-ci.org/tildeio/oasis.js)

Oasis.js is a pleasant API for safe communication with untrusted code in
sandboxed iframes.

# Quick Start Example

For example, imagine we are using a third-party profile viewer to
display information about a user. We only want to expose the bare
minimum required to use the widget, without giving it access to all of
the parent's environment.

Here is what your application would look like:

```html
<!doctype html>

<html>
  <head>
    <script src="http://example.com/jquery.js"></script>
    <script src="http://example.com/oasis.js"></script>
  </head>
  <body>
    <script>
      var oasis = new Oasis();
      var sandbox = oasis.createSandbox({
        url: 'http://example.com/profile_viewer.html',
        type: 'html',
        capabilities: [ 'account' ]
      });

      sandbox.connect('account').then(function(port) {
        port.onRequest('profile', function () {
          return { email: 'wycats@gmail.com' };
        });
      });

      document.body.appendChild(sandbox.el);
    </script>
  </body>
</html>
```

And here is the profile viewer widget (hosted either on your domain
or a third-party's domain):

```html
<!doctype html>

<html>
  <head>
    <script src="http://example.com/jquery.js"></script>
    <script src="http://example.com/oasis.js"></script>
  </head>
  <body>
    <div>
      <p>Email: <span id="email"><img src="loading.png"></span></p>
    </div>
    <script>
      var oasis = new Oasis();
      oasis.connect('account').then(function(port) {
        port.request('profile').then(function(profile) {
          $("#email").html(profile.email);
        });
      });
    </script>
  </body>
</html>
```

# Overview

Traditionally, developers wishing to embed "arbitrary" JavaScript in their application
have faced a choice: they can directly embed it, running all the risks of malicious code
and style bleeding; or they can embed it in a nested iframe and leave it to its own
devices with the only communication between them being through shared backend servers.

Oasis addresses this by allowing two independent JavaScript containers (the "application"
and the "widget" in the above example) to communicate using messages.  On modern browsers,
this is supported internally through the MessageChannel API.  Oasis offers the benefits of
a clean, comprehensible interface and consistent polyfilling across browsers.

A **sandbox** is a nested environment which is fundamentally independent from its parent
(i.e. the only connection is through Oasis).  A sandbox can either be creating with a
visual aspect in an iframe and added to the DOM, or it may be created invisibly in a
web worker.

In order for a sandbox to function, it must download some code from somewhere; typically
this will be a complete, fresh HTML document with `<head>` and `<body>` portions.  Since
no code or styling is shared, this document is responsible for linking to and installing
all the relevant code.  In order to support this, any sandbox must be given a `url` parameter
when it is created.

In order for the two environments to communicate, there must be one or more channels for
communication.  When the sandbox is created, the creating environment must pass in an array
of "capability names", each name representing one **capability** that can be used for communication.

In general, these capabilities will be wrapped in **consumer**/**service** pairs and the consumers
and services can also be specified during sandbox creation.

# Building up the example above

Let's go though the code to produce the example presented above.

The basic outline of the HTML is fairly vanilla and should be obvious (if not, this
may not be the page for you).

Application (say at http://example.com/app.html)

```html
<!doctype html>

<html>
  <head>
    <script src="http://example.com/jquery.js"></script>
    <script src="http://example.com/oasis.js"></script>
  </head>
  <body>
    <script>
    </script>
  </body>
</html>
```

Widget (say at http://example.com/profile_viewer.html):

```html
<!doctype html>

<html>
  <head>
    <script src="http://example.com/jquery.js"></script>
    <script src="http://example.com/oasis.js"></script>
  </head>
  <body>
    <div>
    </div>
    <script>
    </script>
  </body>
</html>
```

Basically, both files load in jquery and oasis and have room to put other code.

## Instantiate Oasis

The `oasis.js` file defines the `Oasis` class, and the first thing that we need
to do on both sides is to create an instance:

```
      var oasis = new Oasis();
```

## Create the Sandbox

Inside the application, we then need to create a sandbox object.  It needs a
`url` pointing to where the widget can be found.  The type `html` says that the
specified URL points to an HTML resource which should be downloaded.

The capabilities array specifies the names of a set of **capabilities** or
services for which communication channels can be established between the two
environments.

```
      var sandbox = oasis.createSandbox({
        url: 'http://example.com/profile_viewer.html',
        type: 'html',
        capabilities: [ 'account' ]
      });
```

The name "account" here is neither magic nor random.  This is the name we have chosen
(as application developers) to give to the capability we want to provide between the two
environments.

We can now set up a `port` through which we will be able to talk to the sandbox.

```
      sandbox.connect('account').then(function(port) {
      });
```

Note that although everything may appear to be live here, nothing will happen until both
ends of the communication channel have been established.  That requires the sandbox to
do its half of the handshaking.  Any messages that are sent _before_ the connection is
completed will be queued and delivered upon connection.  It is the application developer's
responsibility to make sure that either their code is resilient in the face of large backlogs and
multiple messages being delivered simultaneously or they stop that happening through an
additional handshaking phase.

## Create the Oasis Connection in the Sandbox

> Currently, it seems that you need to specifically request
> that your sandbox calls back into the Oasis runtime to make
> the initial connection.  To do this, use the following code:

```
     oasis.autoInitializeSandbox(Oasis.adapters);
```

> I believe this should be automatic (based on the name), so
> hopefully this requirement will go away.

From within the sandbox, it is necessary to connect back to the same capability
defined above.  Because this is an asynchronous operation, we use promises to obtain
the actual connection (called a `port`).

The code we want to write can then be contained within this block.

```
      oasis.connect('account').then(function(port) {
      });
```

Note that here we use `oasis.connect` directly, rather than having a separate handle
as we did on the application side.  This enables us to talk to the "parent" of the sandbox
rather than talking to a nested sandbox.

Once both `connect` operations have completed and called into the callback functions,
the connection will have been established and the communications can begin.

## Logging

As with all software, it will often be the case that things mysteriously do not work.

In order to see into the mysteries of Oasis, it's best to turn on logging.  To do this,
you need to call `enable()` on the `logger` object, an entry on the `oasis` object.

```
      oasis.logger.enable();
```

And logging can also be turned off by calling `disable()`.
























## Creating Sandboxes

Sandboxed applications or widgets can be hosted as JavaScript or HTML.  Both can
be sandboxed inside an iframe, but Oasis can also sandbox JavaScript widgets
inside a web worker.

Sandboxes are created via the `createSandbox` API.

Here is an example of creating an iframe sandbox for a JavaScript widget:
```js
oasis.createSandbox({
  url: 'http://example.com/profile_viewer.js',
  capabilities: [ 'account' ]
});
```

When creating JavaScript sandboxes it is necessary to host Oasis on the same
domain as the sandboxed JavaScript (see [Browser Support](#requirements--browser-support)).

Here is an example of creating an iframe sandbox for an HTML widget:
```js
oasis.createSandbox({
  url: 'http://example.com/profile_viewer.html',
  type: 'html',
  capabilities: [ 'account' ]
});
```

When creating HTML sandboxes, it is the sandbox's responsibility to load Oasis
(typically via a script tag in the head element).

Sandboxed widgets that require no UI can be loaded as web workers:
```js
  url: 'http://example.com/profile_information.js',
  capabilities: [ 'account' ],
  adapter: oasis.adapters.webworker
```

The application can grant specific privileges to the sandbox, like opening windows.

```js
oasis.createSandbox({
  url: 'http://example.com/profile_viewer.html',
  type: 'html',
  capabilities: [ 'account' ],
  sandbox: {
    popups: true
  }
});
```

### Starting Sandboxes

Web worker sandboxes will start immediately.  HTML (ie iframe) sandboxes will
start as soon as their DOM element is placed in the document.  The simplest way
to do this is to append them to the body:

```js
document.body.appendChild(sandbox.el);
```

But they can be placed anywhere in the DOM.  Please note that once in the DOM
the sandboxes should not be moved: iframes moved within documents are reloaded
by the browser.

# API

## Connecting to Ports Directly

For simple applications it can be convenient to connect directly to ports for
a provided capability.

When doing so, you can send messages via `send`.  Messages can be sent in either
direction.
```js
  // in the environment
  sandbox.connect('account').then(function(port) {
    port.send('greeting', 'Hello World!')
  });

  // in the sandbox
  oasis.connect('account').then(function(port) {
    port.on('greeting', function (message) {
      document.body.innerHTML = '<strong>' + message + '</strong>';
    });
  });
```

You can also request data via `request` and respond to data via `onRequest`.
```js
  // in the environment
  sandbox.connect('account').then(function(port) {
    port.onRequest('profile', function () {
      return { name: 'Yehuda Katz' };
    })
  });

  // in the sandbox
  oasis.connect('account').then(function(port) {
    port.request('profile').then( function (name) {
      document.body.innerHTML = 'Hello ' + name;
    });
  });
```

You can also respond to requests with promises, in case you need to retrieve the
data asynchronously.  This example uses
[rsvp](http://github.com/tildeio/rsvp.js), but any
[Promises/A+](http://promises-aplus.github.com/promises-spec/) implementation is
supported.
```js
  // in the environment
  sandbox.connect('account').then(function(port) {
    port.onRequest('profile', function () {
      return new Oasis.RSVP.Promise( function (resolve, reject) {
        setTimeout( function () {
          // Here we're using `setTimeout`, but a more realistic case would
          // involve XMLHttpRequest, IndexedDB, FileSystem &c.
          resolve({ name: 'Yehuda Katz' });
        }, 1);
      });
    })
  });

  // in the sandbox
  oasis.connect('account').then(function(port) {
    // the sandbox code remains unchanged
    port.request('profile').then( function (name) {
      document.body.innerHTML = 'Hello ' + name;
    });
  });
```

## Using Services and Consumers

You can provide services for a sandbox's capabilities to take advantage of a
shorthand for specifying events and request handlers.

```js
  var AccountService = Oasis.Service.extend();
  var sandbox = oasis.createSandbox({
    url: 'http://example.com/profile_viewer.js',
    capabilities: [ 'account' ],
    services: {
      account: AccountService
    }
  });
```

This functionality is available within the sandbox as well: simply specify
consumers when connecting, rather than connecting to each port individually.

```js
var AccountConsumer = Oasis.Consumer.extend();
oasis.connect({
  consumers: {
    account: AccountConsumer
  }
})
```

Note that `Oasis.Service` and `Oasis.Consumer` are class-like, so we refer to
them via `Oasis`.  `oasis`, which we've been using for things like
`createSandbox`, is an instance of `Oasis` created automatically.  You normally
only need this implicit instance, but it's possible to have multiple groups of
sandboxes isolated from each other, although this is an advanced feature.

Services and Consumers can use an `events` shorthand for conveniently defining
event handlers:
```js
  var AccountService = Oasis.Service.extend({
    events: {
      updatedName: function(newName) {
        user.set('name', newName);
      }
    }
  });
```

They can also use a `requests` shorthand for easily defining request handlers.
```js
  var UserService = Oasis.Service.extend({
    requests: {
      basicInformation: function(user) {
        switch (user) {
          case 'wycats':
            return { name: 'Yehuda Katz' };
          case 'hjdivad':
            return { name: 'David J. Hamilton' };
        }
      },

      // The `requests` shorthand also supports asynchronous responses via
      // promises.
      extraInformation: function(user) {
        return new Oasis.RSVP.Promise( function (resolve, reject) {
          // if `loadExtraInformationAsynchronously` returned a promise we could
          // return it directly, as with jQuery's `ajax`.
          loadExtraInformationAsynchronously( function(userInformation) {
            resolve(userInformation);
          });
        });
      }
    }
  });
```

## Wiretapping Sandboxes

Sometimes it's helpful to listen to many, or even all, messages sent to or
received from, a sandbox.  This can be particularly useful in testing.

```js
  sandbox.wiretap( function(capability, message) {
    console.log(capability, message.type, message.data, message.direction);
  });
```

# Requirements & Browser Support

Oasis.js is designed to take advantage of current and upcoming features in
modern browsers.

- `<iframe sandbox>`: An HTML5 feature that allows strict sandboxing of content,
  even served on the same domain.  Available in all Evergreen browsers and
  IE10+.
- `MessageChannel`: An HTML5 feature that allows granular communication between
  iframes.  It replaces the need to do cumbersome multiplexing over a single
  `postMessage` channel.  Available in all Evergreen browsers (and IE10+) with
  the exception of Firefox.
- `postMessage` structured data: An HTML5 feature that allows sending structured
  data, not just strings, over `postMessage`.

Oasis.js supports Chrome, Firefox, Safari 6, and Internet Explorer 8+.  Support
for older browsers depends on polyfills.

- [MessageChannel.js](https://github.com/tildeio/MessageChannel.js) polyfills
  `MessageChannel` where it is unavailable (IE8, IE9 and Firefox).
- [Kamino.js](https://github.com/tildeio/kamino.js) polyfills `postMessage`
  structured data for Internet Explorer.

Support for IE8 and IE9 depends on the sandboxes being hosted on an origin that
differs from the environment, as these versions of IE do not support `<iframe
sandbox>`.  Oasis.js will refuse to create a sandbox if the sandbox attribute is
not supported and the domains are the same.

# Building Oasis.js

Make sure you have node and grunt installed.  Then, run:
```sh
npm install
grunt build
```

# Testing Oasis.js

To run the Oasis.js test, run:
```sh
grunt server
```

Then navigate to `http://localhost:8000`


# Samples

The easiest way to see the samples is to run the test server and navigate to
`http://localhost:8000/samples`.

