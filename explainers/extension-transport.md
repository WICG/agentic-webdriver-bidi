# Extension transport

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

WebDriver BiDi defines one transport: WebSocket. A WebSocket transport needs a network endpoint in the browser. A browser extension cannot use that endpoint easily, and an open port increases the attack surface of the browser.

[w3c/webdriver-bidi#996](https://github.com/w3c/webdriver-bidi/pull/996) refactors the WebDriver BiDi specification to allow non-network transports. It defines an abstract message transport and an abstract connection. Each connection has steps to send a message and steps to close the connection. The WebSocket transport becomes one implementation of the message transport. The message processing algorithms use the abstract connection only.

We propose a second implementation of the message transport: an extension transport. A browser extension uses a WebExtensions API to open a connection to the browser in which it runs (see [#38](https://github.com/WICG/agentic-webdriver-bidi/issues/38) for the manifest permission). The connection does not use the network.

The intended home for the extension transport is the [WebExtensions specification](https://w3c.github.io/webextensions/specification/).

## `browser.driver.connect()`

`browser.driver.connect()` creates a new WebDriver BiDi connection and returns a [`runtime.Port`](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/runtime/Port) object. The returned port sends and receives WebDriver BiDi messages for that connection.

`connect()` does not create a session. The extension sends `session.new` on the port to create a session. The extension sends `session.end` to end the session.

The extension transport supports restricted sessions only. An extension cannot create a standard session on this transport, because a standard session gives full access to the browser.

```ts
browser.driver = {
  // Creates a new WebDriver BiDi connection and returns a runtime.Port.
  connect(): browser.runtime.Port;

  // Resolves a tab id to the id of the top-level navigable in the tab.
  getNavigableId(tabId: number): Promise<string>;

  // Resolves a navigable id to the id of the tab that contains the navigable.
  getTabId(navigableId: string): Promise<number>;
}
```

The returned [`runtime.Port`](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/runtime/Port) uses these members for the WebDriver BiDi connection:

- `onMessage`: a standard WebExtensions [`events.Event`](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/events/Event) with `addListener`, `removeListener`, and `hasListener`. The browser calls each listener with `(message, port)` for each incoming message. A message is a command response, an error response, or an event.
- `onDisconnect`: a standard WebExtensions [`events.Event`](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/events/Event). The browser fires this event at most once when it closes the connection, for example after `session.end`, on setup failure, or on error. Calling `port.disconnect()` does not fire this event on the same port. The listener receives the `Port` object. If the connection closed due to an error, `port.error` (or `browser.runtime.lastError`) describes the error.
- `postMessage(message)`: sends one message to the browser. This method does not return the command response. The response arrives through `onMessage`. If the port is disconnected or `message` is not serializable, `postMessage()` throws synchronously.
- `disconnect()`: closes the port and the connection.

When the port closes for any reason, the browser ends any active session on that connection. This applies to `port.disconnect()`, extension context unload (see [#39](https://github.com/WICG/agentic-webdriver-bidi/issues/39) for service worker lifetime), and browser close.

Each `message` is a JSON-compatible [serializable value](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Chrome_incompatibilities#data_cloning_algorithm) defined by the [WebDriver BiDi protocol definition](https://www.w3.org/TR/webdriver-bidi/#protocol-definition). Outgoing messages sent through `postMessage()` match [`Command`](https://www.w3.org/TR/webdriver-bidi/#cddl-type-command), and incoming messages delivered through `onMessage` match [`Message`](https://www.w3.org/TR/webdriver-bidi/#cddl-type-message). Like `runtime.connectNative()`, the port uses JSON serialization, so JSON-compatible values serialize identically across browsers. Valid serializable values that do not match `Command` receive a WebDriver BiDi error response (for example, `invalid argument` or `unknown command`) through `onMessage`.

The API maps to the abstract connection of [w3c/webdriver-bidi#996](https://github.com/w3c/webdriver-bidi/pull/996) as follows:

| Abstract connection | Extension transport |
| :--- | :--- |
| Establish a connection | `browser.driver.connect()` adds the new connection to `connections not associated with a session`. |
| The transport receives a message from the local end | `port.postMessage(message)` converts `message` and handles it as an incoming message. |
| Send a connection message to the local end | The browser converts the message and calls the `port.onMessage` listeners. |
| Close the connection | After delivering pending messages to `port.onMessage`, the browser closes the port and calls the `port.onDisconnect` listeners. |
| The connection is closed | The extension calls `port.disconnect()`, the extension context unloads, or the browser closes the port. The browser also ends any active session. |

Unlike `handle a connection closing` in [w3c/webdriver-bidi#996](https://github.com/w3c/webdriver-bidi/pull/996), closing the port also runs `end the session` and `cleanup the session` for any active session.

## `browser.driver.getNavigableId(tabId)`

Extension APIs identify a tab with a tab id. WebDriver BiDi identifies a navigable with a navigable id. `browser.driver.getNavigableId(tabId)` resolves the tab id to the id of the top-level navigable in that tab. The method returns a promise that resolves with the navigable id.

```ts
// Resolves a tab id to the id of the top-level navigable in the tab.
browser.driver.getNavigableId(tabId: number): Promise<string>;
```

The extension can use the navigable id in commands, and as a key in the `perNavigable` part of the ACL.

## `browser.driver.getTabId(navigableId)`

`browser.driver.getTabId(navigableId)` is the reverse of `getNavigableId(tabId)`. It resolves the navigable id to the id of the tab that contains the navigable. The method returns a promise that resolves with the tab id.

```ts
// Resolves a navigable id to the id of the tab that contains the navigable.
browser.driver.getTabId(navigableId: string): Promise<number>;
```

The extension can use the tab id in extension APIs, for example to find the tab for a WebDriver BiDi event.

## Example: an extension starts a restricted session

In this example, the user clicks the extension action on a tab. The extension opens a connection and starts a restricted session. Then it requests permission to read the top-level navigable in that tab.

```js
browser.action.onClicked.addListener(async (tab) => {
  const port = browser.driver.connect();

  port.onMessage.addListener((message) => {
    // Process command responses and events.
  });

  port.onDisconnect.addListener((disconnectedPort) => {
    const error = disconnectedPort.error ?? browser.runtime.lastError;
    if (error) {
      // Handle transport error.
    }
    // Handle connection close.
  });

  port.postMessage({
    id: 1,
    method: "session.new",
    params: {
      capabilities: {alwaysMatch: {sessionType: "restricted"}}
    }
  });

  // ... wait for the command with id 1 to finish ...

  const navigableId = await browser.driver.getNavigableId(tab.id);

  port.postMessage({
    id: 2,
    method: "session.setAcl",
    params: {
      acl: {
        perNavigable: {
          [navigableId]: {
            origins: [new URL(tab.url).origin],
            commands: ["browsingContext.captureScreenshot"],
            events: []
          }
        }
      },
      reason: "Read the tab that the user selected."
    }
  });

  // ... wait for the command with id 2 to finish and check the effective ACL ...
});
```

> [!NOTE]
> The port is a low-level interface. An extension can wrap the port in a promise-based interface that is easier to use. For example, a `send(method, params)` function can return a promise that resolves with the response that has the same `id`.
