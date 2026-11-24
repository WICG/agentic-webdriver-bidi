# Access control lists

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

An access control list (ACL) defines the access of a [restricted session](restricted-sessions.md). This document describes the ACL format, the `session.setAcl` command, and the actuation UI that the ACL controls.

The user agent sends the [`session.changed`](restricted-sessions.md#sessionchanged) event when the user or the user agent changes the ACL.

## The access control list

The ACL has two parts:

- `default` defines the access of the full session, for all navigables.
- `perNavigable` adds more access for specific navigables. Each entry extends the `default` configuration. It does not replace it.

Access control is a property of the session. The ACL does not need an entry for each navigable. If a navigable has no `perNavigable` entry, the session has only the `default` access in that navigable (see [#36](https://github.com/WICG/agentic-webdriver-bidi/issues/36) for descendant subframes). An empty entry has the same effect as no entry. The agent requests access in a specific navigable only if the session needs access that `default` does not include:

- The ACL has an entry for each [actuated navigable](restricted-sessions.md#terminology), because actuation commands are valid only in a `perNavigable` entry (see [#28](https://github.com/WICG/agentic-webdriver-bidi/issues/28) on `perNavigable`-only restrictions).
- For a navigable that is not actuated, the ACL needs an entry only for access that is not in `default`, or for commands and events that the user agent allows only in a `perNavigable` entry.

Each part contains three lists (see [#26](https://github.com/WICG/agentic-webdriver-bidi/issues/26) for full-access shortcuts):

- `origins`: the origins that the agent can access.
- `commands`: the names of the commands that the agent can send.
- `events`: the names of the events that the agent can receive.

The `commands` and `events` lists can contain only the names in [Commands and events in a restricted session](commands-and-events.md#commands-and-events-in-a-restricted-session).

```js
{
  default: {
    origins: [...originURL],
    commands: [...commandName],
    events: [...eventName]
  },
  perNavigable: {
    "<navigable id>": {
      origins: [...originURL],
      commands: [...commandName],
      events: [...eventName]
    }
  }
}
```

The user agent enforces the ACL (see [#30](https://github.com/WICG/agentic-webdriver-bidi/issues/30) for error codes):

- A command fails if the ACL does not contain the command for the target navigable.
- A command fails if the ACL does not contain the origin of the target document (see [#8](https://github.com/WICG/agentic-webdriver-bidi/issues/8) for pausing instead of aborting).
- The user agent does not send an event if the ACL does not contain the event for the source navigable.

User agents may restrict some commands and events to be valid only in a `perNavigable` entry. It is recommended that actuation commands are valid only in a `perNavigable` entry, because they are the signal for [actuation UI](#actuation-ui) (see [#28](https://github.com/WICG/agentic-webdriver-bidi/issues/28) on `perNavigable`-only restrictions).

Event subscriptions are independent of the ACL `events` list. The agent uses `session.subscribe` and `session.unsubscribe` as usual; events not allowed by the ACL are filtered out before they are emitted (see [#29](https://github.com/WICG/agentic-webdriver-bidi/issues/29) for subscriptions).

The ACL does not control these commands and events. They are always available in a restricted session:

- `session.status`
- `session.end`
- `session.subscribe`
- `session.unsubscribe`
- `session.setAcl`
- `session.changed`

## `session.setAcl`

The agent sends `session.setAcl` to change the ACL of the session. This command is the only method to get access. There are two uses:

- **Request access.** The agent can provide a `reason`. The user or the user agent can grant or deny the request.
- **Retract access.** For example, the agent removes the access to a navigable after it completes an interaction.

The command accepts two parameters:

- `acl`: the full ACL that the agent requests for the session.
- `reason`: an optional text from the agent that explains the request. The user agent may deny a access request without a `reason`.

The result contains two fields:

- `acl`: the effective ACL of the session after the change. It can be smaller than the requested ACL (if the user agent denied access) or larger (if the user agent grants default access). User-agent-default access remains in the effective ACL even if omitted from a subsequent `session.setAcl` call.
- `denied`: the access that the user agent denied. The user agent omits this field if it granted the full request.

The `denied` field contains two fields:

- `acl`: the denied origins, commands, and events, in the ACL format.
- `reason`: an optional text from the user agent that explains the denial. The agent can use it to select a different approach.

We expect agents to request access only when necessary, and to retract it after use. This is especially important for actuation commands, because they control the [actuation UI](#actuation-ui).

### Ordering of concurrent calls

`session.setAcl` carries the full ACL and may wait on a user prompt. Without ordering, out-of-order prompts lose updates, `denied` is computed against a moving base, and a pending request can undo a revocation.

- The user agent queues only `session.setAcl` calls and processes them strictly one after another, in arrival order. Other commands are not queued and continue to run against the current effective ACL while a `session.setAcl` call is pending.
- A queued `session.setAcl` call is evaluated against the effective ACL at the time it runs, after any revocation by the user. A revoked entry that the request still contains counts as a new request (the user agent prompts again or denies it) and is never silently restored.
- If the agent sends a second `session.setAcl` call before it sees the first result, the second call's full ACL wins. Avoiding this race is the client's responsibility.

### Example: first request

The agent requests access for the first time after `session.new`.

Request:

```json
{
  "id": 2,
  "method": "session.setAcl",
  "params": {
    "acl": {
      "default": {
        "origins": ["https://shop.example"],
        "commands": [
          "browsingContext.getTree",
          "browsingContext.captureScreenshot"
        ],
        "events": ["browsingContext.navigationCommitted"]
      }
    },
    "reason": "Find the item that the user asked for on shop.example."
  }
}
```

Response. The user agent grants the full request:

```json
{
  "id": 2,
  "type": "success",
  "result": {
    "acl": {
      "default": {
        "origins": ["https://shop.example"],
        "commands": [
          "browsingContext.getTree",
          "browsingContext.captureScreenshot"
        ],
        "events": ["browsingContext.navigationCommitted"]
      },
      "perNavigable": {}
    }
  }
}
```

### Example: more access for one navigable

The agent requests more access to navigable `A1B2C3`.

Request:

```json
{
  "id": 3,
  "method": "session.setAcl",
  "params": {
    "acl": {
      "default": {
        "origins": ["https://shop.example"],
        "commands": [
          "browsingContext.getTree",
          "browsingContext.captureScreenshot"
        ],
        "events": ["browsingContext.navigationCommitted"]
      },
      "perNavigable": {
        "A1B2C3": {
          "origins": ["https://shop.example", "https://pay.example"],
          "commands": ["browsingContext.navigate", "input.performActions", "script.evaluate"],
          "events": ["browsingContext.userPromptOpened", "log.entryAdded"]
        }
      }
    },
    "reason": "Add the selected item to the cart and open the checkout page."
  }
}
```

Response. In this example, the user agent denies `script.evaluate`, `log.entryAdded`, and the `https://pay.example` origin, and it gives a reason:

```json
{
  "id": 3,
  "type": "success",
  "result": {
    "acl": {
      "default": {
        "origins": ["https://shop.example"],
        "commands": [
          "browsingContext.getTree",
          "browsingContext.captureScreenshot"
        ],
        "events": ["browsingContext.navigationCommitted"]
      },
      "perNavigable": {
        "A1B2C3": {
          "origins": ["https://shop.example"],
          "commands": ["browsingContext.navigate", "input.performActions"],
          "events": ["browsingContext.userPromptOpened"]
        }
      }
    },
    "denied": {
      "acl": {
        "perNavigable": {
          "A1B2C3": {
            "origins": ["https://pay.example"],
            "commands": ["script.evaluate"],
            "events": ["log.entryAdded"]
          }
        }
      },
      "reason": "The user did not allow access to payment pages. A policy does not allow script evaluation and console messages."
    }
  }
}
```

## Actuation UI

User agents are encouraged to show dedicated UI while an agent actuates a navigable ([#6](https://github.com/WICG/agentic-webdriver-bidi/issues/6)). For example, the user agent can:

- Show that the agent controls the navigable.
- Show other UI that helps the user follow the agent, for example click visualization.
- Show a control, for example a stop button, that lets the user stop the agent.

The ACL is the signal for this UI. A navigable is an actuated navigable while its `perNavigable` entry in the effective ACL contains at least one actuation command. Thus:

- The actuation starts when `session.setAcl` grants an actuation command for the navigable.
- The actuation ends when the effective ACL no longer contains an actuation command for the navigable. This occurs when the agent retracts the access with `session.setAcl`, or when the user or the user agent revokes it. It also occurs when the navigable closes (see [#37](https://github.com/WICG/agentic-webdriver-bidi/issues/37)) or the session ends.

The actuation continues across navigations in the navigable, and across commands that are not actuation commands. The agent can request access to more than one navigable at the same time. Each of these navigables is an actuated navigable.

Actuation commands are valid only in a `perNavigable` entry. If an agent requests an actuation command in the `default` part of the ACL, the user agent denies it. Thus the user agent always knows which navigables the agent can actuate.

The actuation commands include:

- `browsingContext.handleUserPrompt`
- `browsingContext.navigate`
- `browsingContext.reload`
- `browsingContext.traverseHistory`
- `input.performActions`
- `input.releaseActions`
- `script.callFunction`
- `script.evaluate`

`script.callFunction` and `script.evaluate` are actuation commands because script can click elements and change the page. See [#33](https://github.com/WICG/agentic-webdriver-bidi/issues/33) for read-only script evaluation.

An agent that only observes a navigable does not cause actuation UI. For example, a `perNavigable` entry with only `browsingContext.captureScreenshot` does not make the navigable an actuated navigable.

When the user clicks the stop button, the user agent pauses the session and sends [`session.changed`](restricted-sessions.md#sessionchanged) with `paused: true`. The user agent can also revoke the actuation commands from the ACL.

The agent must retract the actuation commands for a navigable when it completes the actuation. Otherwise the user agent continues to show the actuation UI. The protocol cannot enforce this. The user agent can revoke access at any time, for example after a period of inactivity (see [#34](https://github.com/WICG/agentic-webdriver-bidi/issues/34)), and send `session.changed`.

In this example, the agent completes its task on navigable `A1B2C3`. It retracts the `perNavigable` entry, and the user agent removes the actuation UI:

```json
{
  "id": 6,
  "method": "session.setAcl",
  "params": {
    "acl": {
      "default": {
        "origins": ["https://shop.example"],
        "commands": [
          "browsingContext.getTree",
          "browsingContext.captureScreenshot"
        ],
        "events": ["browsingContext.navigationCommitted"]
      },
      "perNavigable": {}
    }
  }
}
```

## Detailed design discussion

### Commands and events are separate lists

Events can expose page data, for example console messages or URLs of navigations. Thus the ACL controls events as well as commands. Separate lists let the agent request an event without a related command.

### Default and per-navigable access

Most tasks operate on one tab or a small number of tabs. The `perNavigable` part lets the user grant more access for one tab only. The agent can continue to use the `default` access for other tabs. Additionally, user agent needs to know which tabs are being controlled by the agent.

### The ACL is the signal for actuation UI

The user agent must know when to show actuation UI. If the user agent finds this out from the commands that it receives, the UI is not reliable. For example, it can show the UI after an `input.performActions` command and remove it after a timeout. Then the UI shows after the first action, not before it. The UI can also flicker between two actions of one task.

The per-navigable ACL does not have these problems. The agent requests actuation commands for a navigable before the first action, and retracts them when the task is complete. The actuation continues across more than one `session.setAcl` call, and across commands that are not actuation commands. Thus the user agent can show the UI before the first action and keep it until the agent retracts the access.

This design does not add commands to the protocol. The ACL already contains the necessary information.

The cost is that the actuation UI depends on the agent: the agent must retract the actuation commands after use. The protocol cannot enforce this, but the user agent can revoke access at any time.

## Considered alternatives

### Generic access control with allowlists and denylists

A more general capability can contain allowlists and denylists for modules, commands, events, and parameters. This alternative has more flexibility, but it is more complex to specify and to show to a user. This proposal uses allowlists only, and it adds origins, per-navigable scope, and in-protocol access requests.

### ACL in `session.new`

An earlier version of this document accepted an ACL as a capability in `session.new`. The result contained the granted ACL. We replaced this design because it made two methods to request access. The request in `session.new` also had no `reason` parameter, and a user prompt could delay session creation.
