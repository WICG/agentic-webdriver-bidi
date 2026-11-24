# Commands and events in a restricted session

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

A restricted session supports only the commands and events in this document (see [#27](https://github.com/WICG/agentic-webdriver-bidi/issues/27) on the allowed set and discovery). The user agent rejects all other commands. The `commands` and `events` lists of an ACL can contain only the navigable commands and events below; session commands and events are always available and not controlled by the ACL.

Most of these commands and events are existing WebDriver BiDi commands and events. This proposal adds `session.setAcl` and `session.changed`. It also extends some existing commands, as the tables show.

Session commands and events:

| Name | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| [`session.new`](restricted-sessions.md#sessionnew) | Command | Existing, extended | Starts a restricted session if the capabilities contain `sessionType: "restricted"`. |
| `session.status` | Command | Existing | Returns whether the user agent is ready to create a new session. |
| `session.end` | Command | Existing | Ends the current session. |
| `session.subscribe` | Command | Existing | Subscribes to any events. The user agent emits only events allowed by the ACL. |
| `session.unsubscribe` | Command | Existing | Unsubscribes from events. |
| [`session.setAcl`](acl.md#sessionsetacl) | Command | New | Requests or retracts access. The parameters are `acl` and an optional `reason`. The result contains the granted and denied access. |
| [`session.changed`](restricted-sessions.md#sessionchanged) | Event | New | Tells the agent that the user or the user agent paused, resumed, or reconfigured the session. The fields include `acl` and `paused`. |

Commands and events that observe navigables:

| Name | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| `browsingContext.getTree` | Command | Existing | Returns the tree of navigables whose origins are allowed by the ACL. Navigables with disallowed origins are omitted. |
| `browsingContext.captureScreenshot` | Command | Existing | Returns a screenshot of a navigable. The user agent redacts the screenshot in accordance with the origins in the ACL. |
| `browsingContext.locateNodes` | Command | Existing, extended | Finds DOM elements by locator. The extension adds a locator for an `x`, `y` position and return sharedIds. |
| `log.entryAdded` | Event | Existing | Contains a console message or uncaught JavaScript error from a navigable, if allowed by the user agent. |
| `browsingContext.userPromptOpened` | Event | Existing | Tells the agent that a navigable opened a JavaScript dialog. |
| `browsingContext.navigationCommitted` | Event | Existing | Tells the agent that a navigable navigated. The event contains the new URL. |
| `browsingContext.fragmentNavigated` | Event | Existing | Tells the agent that a navigable navigated to a fragment URL. |
| `browsingContext.historyUpdated` | Event | Existing | Tells the agent that a navigable updated its history entry (for example via `history.pushState`). |
| `browsingContext.userPromptClosed` | Event | Existing | Tells the agent that a JavaScript dialog in a navigable closed. |
| `browsingContext.contextCreated` | Event | Existing | Tells the agent that a navigable was created. |
| `browsingContext.contextDestroyed` | Event | Existing | Tells the agent that a navigable was destroyed. |

Commands that change navigables:

| Name | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| `browsingContext.create` | Command | Existing | Creates a navigable. This command is not an actuation command, because the agent cannot actuate the new navigable without explicit access. |
| `browsingContext.close` | Command | Existing | Closes a navigable. |
| `browsingContext.navigate` | Command | Existing | Navigates a navigable. |
| `browsingContext.reload` | Command | Existing | Reloads a navigable. |
| `browsingContext.traverseHistory` | Command | Existing | Moves a navigable backward or forward in its session history. |
| `browsingContext.handleUserPrompt` | Command | Existing | Accepts or dismisses a JavaScript dialog in a navigable. |
| `input.performActions` | Command | Existing, extension planned | Sends trusted keyboard, pointer, and wheel actions to a navigable. |
| `input.releaseActions` | Command | Existing | Releases all keys and pointer buttons that are in the pressed state in a navigable. |
| `script.callFunction` | Command | Existing, restricted | Calls a function in a navigable (for example with `sharedId` node references from `browsingContext.locateNodes`). `target` must specify a browsing context and a non-empty `sandbox`. Awaits promises when `awaitPromise` is `true` and returns a serialized result (`script.RemoteValue`). |
| `script.evaluate` | Command | Existing, restricted | Evaluates script in a navigable. `target` must specify a browsing context and a non-empty `sandbox`. Awaits promises when `awaitPromise` is `true` and returns a serialized result (`script.RemoteValue`). |

`input.performActions` does not tell the agent the result of an action ([#5](https://github.com/WICG/agentic-webdriver-bidi/issues/5)). The agent needs this feedback:

- The position of a click and the element that the click hit.
- The distance of a scroll and the position of the viewport on the page after the scroll.
- The text that an input field received.

We must extend `input.performActions` to return this feedback in the command result or as an event, or add new commands that return this data.
