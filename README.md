chrome-remote-interface
=======================

[Chrome Debugging Protocol] interface that helps to instrument Chrome (or any
other suitable [implementation](#implementations)) by providing a simple
abstraction of commands and notifications using a straightforward JavaScript
API.

`chrome-remote-interface` is listed
among [third-party Chrome Debugging Protocol clients][3rd-party].

[3rd-party]: https://developer.chrome.com/devtools/docs/debugging-clients#chrome-remote-interface

Sample API usage
----------------

The following snippet loads `https://github.com` and dumps every request made:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome(function (chrome) {
    with (chrome) {
        Network.requestWillBeSent(function (params) {
            console.log(params.request.url);
        });
        Page.loadEventFired(close);
        Network.enable();
        Page.enable();
        once('ready', function () {
            Page.navigate({'url': 'https://github.com'});
        });
    }
}).on('error', function (err) {
    console.error('Cannot connect to Chrome:', err);
});
```

Installation
------------

    npm install chrome-remote-interface

Implementations
---------------

This module should work with every application implementing the
[Chrome Debugging Protocol]. In particular, it has been tested against the
following implementations:

Implementation             | Protocol version   | [Protocol] | [List] | [New] | [Activate] | [Close] | [Version]
---------------------------|--------------------|------------|--------|-------|------------|---------|-----------
[Google Chrome][1.1]       | [tip-of-tree][1.2] | yes        | yes    | yes   | yes        | yes     | yes
[Microsoft Edge][2.1]      | [partial][2.2]     | yes        | yes    | no    | no         | no      | yes
[Node.js][3.1] ([v6.3.0]+) | [node][3.2]        | yes        | no     | no    | no         | no      | yes

[1.1]: https://www.chromium.org/
[1.2]: https://chromedevtools.github.io/debugger-protocol-viewer/tot/
[2.1]: https://www.microsoft.com/windows/microsoft-edge
[2.2]: https://github.com/Microsoft/edge-diagnostics-adapter/wiki/Supported-features-and-API
[3.1]: https://nodejs.org/
[3.2]: https://chromedevtools.github.io/debugger-protocol-viewer/v8/

[v6.3.0]: https://nodejs.org/en/blog/release/v6.3.0/

[Protocol]: #moduleprotocoloptions-callback
[List]: #modulelistoptions-callback
[New]: #modulenewoptions-callback
[Activate]: #moduleactivateoptions-callback
[Close]: #modulecloseoptions-callback
[Version]: #moduleversionoptions-callback

Setup
-----

An instance of either Chrome itself or another implementation needs to be
running on a known port in order to use this module (defaults to
`localhost:9222`).

### Chrome/Chromium

#### Desktop

Start Chrome with the `--remote-debugging-port` option, for example:

    google-chrome --remote-debugging-port=9222

#### Android

Plug the device and enable the [port forwarding][adb], for example:

    adb forward tcp:9222 localabstract:chrome_devtools_remote

[adb]: https://developer.chrome.com/devtools/docs/remote-debugging-legacy

### Edge

Install and run the [Edge Diagnostics Adapter][edge-adapter].

[edge-adapter]: https://github.com/Microsoft/edge-diagnostics-adapter

### Node.js

Start Node.js with the `--inspect` option, for example:

    node --inspect=9222 script.js

Bundled client
--------------

This module comes with a bundled client application that can be used to
interactively control a remote instance.

### Tab management

The bundled client exposes subcommands to interact with the HTTP frontend
(e.g., [List](#modulelistoptions-callback), [New](#modulenewoptions-callback),
etc.), run with `--help` to display the list of available options.

Here are some examples:

```javascript
$ chrome-remote-interface new 'http://example.com'
{ description: '',
  devtoolsFrontendUrl: '/devtools/inspector.html?ws=localhost:9222/devtools/page/b049bb56-de7d-424c-a331-6ae44cf7ae01',
  id: 'b049bb56-de7d-424c-a331-6ae44cf7ae01',
  thumbnailUrl: '/thumb/b049bb56-de7d-424c-a331-6ae44cf7ae01',
  title: '',
  type: 'page',
  url: 'http://example.com/',
  webSocketDebuggerUrl: 'ws://localhost:9222/devtools/page/b049bb56-de7d-424c-a331-6ae44cf7ae01' }
$ chrome-remote-interface close 'b049bb56-de7d-424c-a331-6ae44cf7ae01'
```

### Inspection

Using the `inspect` subcommand it is possible to
perform [command execution](#chromedomainmethodparams-callback)
and [event binding](#chromedomaineventcallback) in a REPL fashion. But unlike
the regular API the callbacks are overridden to conveniently display the result
of the commands and the message of the events. Also, the event binding is
simplified here, executing a shorthand method (e.g., `Page.loadEventFired()`)
toggles the event registration.

Here is a sample session:

```javascript
$ chrome-remote-interface inspect
>>> Runtime.evaluate({expression: 'window.location.toString()'})
{ result:
   { result:
      { type: 'string',
        value: 'https://www.google.it/_/chrome/newtab?espv=2&ie=UTF-8' },
     wasThrown: false } }
>>> Page.enable()
{ result: {} }
>>> Page.loadEventFired() // registered
{ 'Page.loadEventFired': true }
>>> Page.loadEventFired() // unregistered
{ 'Page.loadEventFired': false }
>>> Page.loadEventFired() // registered
{ 'Page.loadEventFired': true }
>>> Page.navigate({url: 'https://github.com'})
{ result: { frameId: '28677.1' } }
{ 'Page.loadEventFired': { timestamp: 21385.383076 } }
>>> Runtime.evaluate({expression: 'window.location.toString()'})
{ result:
   { result: { type: 'string', value: 'https://github.com/' },
     wasThrown: false } }
```

#### Event filtering

To reduce the amount of data displayed by the event listeners it is possible to
provide a filter function. In this example only the resource URL is shown:

```javascript
$ chrome-remote-interface inspect
>>> Network.enable()
{ result: {} }
>>> Network.requestWillBeSent(params => params.request.url)
{ 'Network.requestWillBeSent': 'params => params.request.url' }
>>> Page.navigate({url: 'https://www.wikipedia.org'})
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/' }
{ result: { frameId: '5530.1' } }
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/portal/wikipedia.org/assets/img/Wikipedia_wordmark.png' }
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/portal/wikipedia.org/assets/img/Wikipedia-logo-v2.png' }
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/portal/wikipedia.org/assets/js/index-3b68787aa6.js' }
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/portal/wikipedia.org/assets/js/gt-ie9-c84bf66d33.js' }
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/portal/wikipedia.org/assets/img/sprite-bookshelf_icons.png?16ed124e8ca7c5ce9d463e8f99b2064427366360' }
{ 'Network.requestWillBeSent': 'https://www.wikipedia.org/portal/wikipedia.org/assets/img/sprite-project-logos.png?9afc01c5efe0a8fb6512c776955e2ad3eb48fbca' }
```

Embedded documentation
----------------------

In both the REPL and the regular API every object of the protocol is *decorated*
with the information available through the [Chrome Debugging Protocol]. The
`category` field determines if the member is a `command`, an `event` or a
`type`.

Remember that the REPL interface provides completion. For example to learn how
to call `Page.navigate`:

```javascript
>>> Page.navigate
{ [Function]
  category: 'command',
  parameters: { url: { type: 'string', description: 'URL to navigate the page to.' } },
  returns:
   [ { name: 'frameId',
       '$ref': 'FrameId',
       hidden: true,
       description: 'Frame id that will be navigated.' } ],
  description: 'Navigates current page to the given URL.',
  handlers: [ 'browser', 'renderer' ] }
```

To learn about the parameters returned by the `Network.requestWillBeSent` event:

```javascript
>>> Network.requestWillBeSent
{ [Function]
  category: 'event',
  description: 'Fired when page is about to send HTTP request.',
  parameters:
   { requestId: { '$ref': 'RequestId', description: 'Request identifier.' },
     frameId:
      { '$ref': 'Page.FrameId',
        description: 'Frame identifier.',
        hidden: true },
     loaderId: { '$ref': 'LoaderId', description: 'Loader identifier.' },
     documentURL:
      { type: 'string',
        description: 'URL of the document this request is loaded for.' },
     request: { '$ref': 'Request', description: 'Request data.' },
     timestamp: { '$ref': 'Timestamp', description: 'Timestamp.' },
     wallTime:
      { '$ref': 'Timestamp',
        hidden: true,
        description: 'UTC Timestamp.' },
     initiator: { '$ref': 'Initiator', description: 'Request initiator.' },
     redirectResponse:
      { optional: true,
        '$ref': 'Response',
        description: 'Redirect response data.' },
     type:
      { '$ref': 'Page.ResourceType',
        optional: true,
        hidden: true,
        description: 'Type of this resource.' } } }
```

To inspect the `Network.Request` (note that unlike commands and events, types
are named in upper camel case) type:

```javascript
>>> Network.Request
{ category: 'type',
  id: 'Request',
  type: 'object',
  description: 'HTTP request data.',
  properties:
   { url: { type: 'string', description: 'Request URL.' },
     method: { type: 'string', description: 'HTTP request method.' },
     headers: { '$ref': 'Headers', description: 'HTTP request headers.' },
     postData:
      { type: 'string',
        optional: true,
        description: 'HTTP POST request data.' },
     mixedContentType:
      { optional: true,
        type: 'string',
        enum: [Object],
        description: 'The mixed content status of the request, as defined in http://www.w3.org/TR/mixed-content/' },
     initialPriority:
      { '$ref': 'ResourcePriority',
        description: 'Priority of the resource request at the time request is sent.' } } }
```

Chrome Debugging Protocol versions
----------------------------------

`chrome-remote-interface` uses the [local version] of the protocol descriptor by
default. This file is manually updated from time to time using
`scripts/update-protocol.sh` and pushed to this repository.

This behavior can be changed by setting the `remote` option to `true`
upon [connection](#moduleoptions-callback), in which case the remote instance is
*asked* to provide its own protocol descriptor.

Currently Chrome is not able to do that (see [#10]), so the protocol descriptor
is fetched from the proper [source repository].

To override the above behavior there are basically three options:

- pass a custom protocol descriptor upon [connection](#moduleoptions-callback)
  (`protocol` option);

- use the *raw* version of the [commands](#chromesendmethod-params-callback)
  and [events](#event-method) interface;

- update the local copy with `scripts/update-protocol.sh` (not present when
  fetched with `npm install`).

[local version]: lib/protocol.json
[#10]: https://github.com/cyrus-and/chrome-remote-interface/issues/10
[source repository]: https://chromium.googlesource.com/chromium/src/+/master/third_party/WebKit/Source/

API
---

### module([options], [callback])

Connects to a remote instance using the [Chrome Debugging Protocol].

`options` is an object with the following optional properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`;
- `chooseTab`: determines which tab this instance should attach to. The behavior
  changes according to the type:

  - a `function` that takes the array returned by the `List` method and returns
    the numeric index of a tab;
  - a tab `object` like those returned by the `New` and `List` methods;
  - a `string` representing the raw WebSocket URL, in this case `host` and
    `port` are not used to fetch the tab list.

  Defaults to a function which returns the currently active tab (`function
  (tabs) { return 0; }`);
- `protocol`: [Chrome Debugging Protocol] descriptor object. Defaults to use the
  protocol chosen according to the `remote` option;
- `remote`: a boolean indicating whether the protocol must be fetched *remotely*
  or if the local version must be used. It has no effect if the `protocol`
  option is set. Defaults to `false`.

`callback` is a listener automatically added to the `connect` event of the
returned `EventEmitter`. When `callback` is omitted a `Promise` object is
returned which becomes fulfilled if the `connect` event is triggered and
rejected if any of the `disconnect` or `error` events are triggered.

The `EventEmitter` supports the following events:

#### Event: 'connect'

```javascript
function (chrome) {}
```

Emitted when the connection to the WebSocket is established.

`chrome` is an instance of the `Chrome` class.

#### Event: 'disconnect'

```javascript
function () {}
```

Emitted when an instance closes the WebSocket connection.

This may happen for example when the user opens DevTools for the currently
inspected Chrome tab.

#### Event: 'error'

```javascript
function (err) {}
```

Emitted when `http://host:port/json` cannot be reached or if it is not possible
to connect to the WebSocket.

`err` is an instance of `Error`.

### module.Protocol([options], [callback])

Fetch the [Chrome Debugging Protocol] descriptor.

`options` is an object with the following optional properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`;
- `remote`: a boolean indicating whether the protocol must be fetched *remotely*
  or if the local version must be returned. If it is not possible to fulfill the
  request then the local version is used. Defaults to `false`.

`callback` is executed when the protocol is fetched, it gets the following
arguments:

- `err`: a `Error` object indicating the success status;
- `protocol`: an object with the following properties:
   - `remote`: a boolean indicating whether the returned descriptor is the
     remote version or not (due to user choice or error);
   - `descriptor`: the [Chrome Debugging Protocol] descriptor.

When `callback` is omitted a `Promise` object is returned.

For example:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome.Protocol(function (err, protocol) {
    if (!err) {
        console.log(JSON.stringify(protocol.descriptor, null, 4));
    }
});
```

### module.List([options], [callback])

Request the list of the available open tabs of the remote instance.

`options` is an object with the following optional properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`.

`callback` is executed when the list is correctly received, it gets the
following arguments:

- `err`: a `Error` object indicating the success status;
- `tabs`: the array returned by `http://host:port/json/list` containing the tab
  list.

When `callback` is omitted a `Promise` object is returned.

For example:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome.List(function (err, tabs) {
    if (!err) {
        console.log(tabs);
    }
});
```

### module.New([options], [callback])

Create a new tab in the remote instance.

`options` is an object with the following optional properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`;
- `url`: URL to load in the new tab. Defaults to `about:blank`.

`callback` is executed when the tab is created, it gets the following arguments:

- `err`: a `Error` object indicating the success status;
- `tab`: the object returned by `http://host:port/json/new` containing the tab.

When `callback` is omitted a `Promise` object is returned.

For example:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome.New(function (err, tab) {
    if (!err) {
        console.log(tab);
    }
});
```

### module.Activate([options], [callback])

Activate an open tab of the remote Chrome instance.

`options` is an object with the following properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`;
- `id`: Tab id. Required, no default.

`callback` is executed when the response to the activation request is
received. It gets the following arguments:

- `err`: a `Error` object indicating the success status;

When `callback` is omitted a `Promise` object is returned.

For example:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome.Activate({'id': 'CC46FBFA-3BDA-493B-B2E4-2BE6EB0D97EC'}, function (err) {
    if (!err) {
        console.log('success! tab is closing');
    }
});
```

### module.Close([options], [callback])

Close an open tab of the remote instance.

`options` is an object with the following properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`;
- `id`: Tab id. Required, no default.

`callback` is executed when the response to the close request is received. It
gets the following arguments:

- `err`: a `Error` object indicating the success status;

When `callback` is omitted a `Promise` object is returned.

For example:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome.Close({'id': 'CC46FBFA-3BDA-493B-B2E4-2BE6EB0D97EC'}, function (err) {
    if (!err) {
        console.log('success! tab is closing');
    }
});
```

Note that the callback is fired when the tab is *queued* for removal, but the
actual removal will occur asynchronously.

### module.Version([options], [callback])

Request version information from the remote instance.

`options` is an object with the following optional properties:

- `host`: HTTP frontend host. Defaults to `localhost`;
- `port`: HTTP frontend port. Defaults to `9222`.

`callback` is executed when the version information is correctly received, it
gets the following arguments:

- `err`: a `Error` object indicating the success status;
- `info`: a JSON object returned by `http://host:port/json/version` containing
  the version information.

When `callback` is omitted a `Promise` object is returned.

For example:

```javascript
const Chrome = require('chrome-remote-interface');
Chrome.Version(function (err, info) {
    if (!err) {
        console.log(info);
    }
});
```

### Class: Chrome

#### Event: 'event'

```javascript
function (message) {}
```

Emitted when the remote instance sends any notification through the WebSocket.

`message` is the object received, it has the following properties:

- `method`: a string describing the notification (e.g.,
  `'Network.requestWillBeSent'`);
- `params`: an object containing the payload.

Refer to the [Chrome Debugging Protocol] specification for more information.

For example:

```javascript
chrome.on('event', function (message) {
    if (message.method === 'Network.requestWillBeSent') {
        console.log(message.params);
    }
});
```

#### Event: '`<method>`'

```javascript
function (params) {}
```

Emitted when the remote instance sends a notification for `<method>` through the
WebSocket.

`params` is an object containing the payload.

This is just a utility event which allows to easily listen for specific
notifications (see [`'event'`](#event-event)), for example:

```javascript
chrome.on('Network.requestWillBeSent', console.log);
```

#### Event: 'ready'

```javascript
function () {}
```

Emitted every time that there are no more pending commands waiting for a
response from the remote instance. The interaction is asynchronous so the only
way to serialize a sequence of commands is to use the callback provided by the
`chrome.send` method. This event acts as a barrier and it is useful to avoid the
callback hell in certain simple situations.

For example to load a URL only after having enabled the notifications of both
`Network` and `Page` domains:

```javascript
chrome.Network.enable();
chrome.Page.enable();
chrome.once('ready', function () {
    chrome.Page.navigate({'url': 'https://github.com'});
});
```

In this particular case, not enforcing this kind of serialization may cause that
the remote instance does not properly deliver the desired notifications the
client.

#### chrome.send(method, [params], [callback])

Issue a command to the remote instance.

`method` is a string describing the command.

`params` is an object containing the payload.

`callback` is executed when the remote instance sends a response to this
command, it gets the following arguments:

- `error`: a boolean value indicating the success status, as reported by the
  remote instance;
- `response`: an object containing either the response (`result` field, if
  `error === false`) or the indication of the error (`error` field, if `error
  === true`).

When `callback` is omitted a `Promise` object is returned instead, with the
fulfilled/rejected states implemented according to the `error` parameter.

Note that the field `id` mentioned in the [Chrome Debugging Protocol]
specification is managed internally and it is not exposed to the user.

For example:

```javascript
chrome.send('Page.navigate', {'url': 'https://github.com'}, console.log);
```

#### chrome.`<domain>`.`<method>`([params], [callback])

Just a shorthand for:

```javascript
chrome.send('<domain>.<method>', params, callback);
```

For example:

```javascript
chrome.Page.navigate({'url': 'https://github.com'}, console.log);
```

#### chrome.`<domain>`.`<event>`(callback)

Just a shorthand for:

```javascript
chrome.on('<domain>.<event>', callback);
```

For example:

```javascript
chrome.Network.requestWillBeSent(console.log);
```

#### chrome.close([callback])

Close the connection to the remote instance.

`callback` is executed when the WebSocket is successfully closed.

When `callback` is omitted a `Promise` object is returned.

Contributors
------------

- [Andrey Sidorov](https://github.com/sidorares)
- [Greg Cochard](https://github.com/gcochard)

Resources
---------

- [Chrome Debugging Protocol]
- [Chrome Debugging Protocol Viewer](https://chromedevtools.github.io/debugger-protocol-viewer/)
- [Chrome Debugging Protocol Google group](https://groups.google.com/forum/#!forum/chrome-debugging-protocol)
- [Showcase Chrome Debugging Protocol Clients](https://developer.chrome.com/devtools/docs/debugging-clients)

[Chrome Debugging Protocol]: https://developer.chrome.com/devtools/docs/debugger-protocol
