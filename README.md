# Agentic extensions for WebDriver BiDi

## Authors

- Alex Rudenko (Google)
- Maksim Sadym (Google)

## Participate

- [Issue tracker](https://github.com/WICG/agentic-webdriver-bidi/issues)
- [Original proposal and discussion](https://github.com/WICG/proposals/issues/320)

## Table of contents

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
- [Proposed solution](#proposed-solution)
  - [Terminology](#terminology)
  - [The access control list](#the-access-control-list)
  - [`session.new`](#sessionnew)
  - [`session.setAcl`](#sessionsetacl)
  - [`session.changed`](#sessionchanged)
- [Example: a full session](#example-a-full-session)
- [How this solution addresses the use cases](#how-this-solution-addresses-the-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
- [Roadmap](#roadmap)
- [Considered alternatives](#considered-alternatives)
- [Security and privacy considerations](#security-and-privacy-considerations)
- [Stakeholder feedback](#stakeholder-feedback)
- [References and acknowledgements](#references-and-acknowledgements)

## Introduction

[WebDriver BiDi](https://www.w3.org/TR/webdriver-bidi/) is a browser automation protocol. Its primary purpose is automated browser testing. The protocol assumes complete trust between the client and the browser. A client gets full access to the browser instance and can do destructive actions without security checks.

AI agents also need browser automation. Many agents use low-level protocols or test setups that give unrestricted access without user supervision.

We propose extensions to WebDriver BiDi for agentic use cases that include agentic browsing. A WebDriver BiDi session represents one long-running agentic activity in the browser. Permissions are properties of that session. The user agent enforces the permissions. The user keeps supervision and control of the agent.

A client starts a restricted session with `sessionType: "restricted"` in `session.new`. A restricted session starts with no permissions beyond any user-agent defaults. The agent requests permissions and learns the effective ACL with `session.setAcl`.

An access control list (ACL) contains the permissions of a restricted session. The ACL contains three types of permissions:

- **Origins**: the origins that the agent can access.
- **Commands**: the WebDriver BiDi commands that the agent can send.
- **Events**: the WebDriver BiDi events that the agent can receive.

> [!NOTE]
> This proposal is in incubation. All names and data shapes in this document are illustrative and can change. The goal is to move the final features into the WebDriver BiDi specification.

## Goals

1. Let the user supervise and control agent actions in the browser.
2. Keep the agent informed about user decisions.
3. Make the permission mechanism a part of the protocol, so that it is interoperable between browsers.
4. Adapt existing WebDriver BiDi commands and add new commands for agentic use cases.

## Non-goals

1. Testing-specific and debugging-specific features. Standard WebDriver BiDi sessions continue to serve these use cases.
2. A definition of the browser UI for permission prompts. Each user agent selects how it shows permissions to the user.
3. Restrictions on high-level user intent. For example, the protocol cannot know that the user does not want the agent to delete an account.

## Use cases

1. A restricted session can use only the operations that the user or the user agent allowed.
2. The user interrupts an active session. The agent receives a notification and adapts.
3. The user revokes a permission or pauses the session. The session loses the related capabilities immediately, but the connection stays open.
4. The agent requests more capabilities during a session. The user or the user agent makes the decision, and the agent receives the result.

The [original proposal](https://github.com/WICG/proposals/issues/320) contains more use cases: structured page content, feedback on input, and access to user profile data. They are in scope. See the [roadmap](#roadmap).

## Proposed solution

### Terminology

- **User agent**: the browser.
- **Agent**: the WebDriver BiDi client controlled by an AI system.
- **Restricted session**: a WebDriver BiDi session with `sessionType: "restricted"`. The user agent allows only the operations in the ACL of the session.
- **Access control list (ACL)**: a dictionary that contains the allowed origins, commands, and events for a restricted session.

### The access control list

The ACL has two parts:

- `default` contains the permissions of the full session, for all navigables.
- `perNavigable` contains more permissions for specific navigables. Each entry extends the `default` configuration. It does not replace it.

Each part contains three lists:

- `origins`: the origins that the agent can access.
- `commands`: the names of the commands that the agent can send.
- `events`: the names of the events that the agent can receive.

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

The user agent enforces the ACL:

- A command fails if the ACL does not contain the command for the target navigable.
- A command fails if the ACL does not contain the origin of the target document.
- The user agent does not send an event if the ACL does not contain the event for the source navigable.

User agents may restrict some commands and events to be valid only in a `perNavigable` entry.

The ACL does not control these commands and events. They are always available in a restricted session:

- `session.status`
- `session.end`
- `session.setAcl`
- `session.changed`

### `session.new`

A client includes `sessionType: "restricted"` in the capabilities of `session.new` to start a restricted session. The result confirms the session type.

`session.new` does not accept an ACL. A restricted session starts with an empty ACL. The agent must send [`session.setAcl`](#sessionsetacl) before it can use other commands or receive other events.

Request:

```json
{
  "id": 1,
  "method": "session.new",
  "params": {
    "capabilities": {
      "alwaysMatch": {
        "sessionType": "restricted"
      }
    }
  }
}
```

Response:

```json
{
  "id": 1,
  "type": "success",
  "result": {
    "sessionId": "3f1c6a52-9f0e-4a57-8a0b-6d1e2c7b9a10",
    "capabilities": {
      "sessionType": "restricted"
    }
  }
}
```

### `session.setAcl`

The agent sends `session.setAcl` to change the ACL of the session. This command is the only method to get permissions. There are two uses:

- **Request permissions.** The agent must give a `reason`. The user or the user agent can grant or deny the request.
- **Retract permissions.** For example, the agent removes the permissions for a navigable after it completes an interaction.

The `acl` parameter contains the full ACL that the agent requests for the session.

The result contains two fields:

- `acl`: the ACL of the session after the change. It can be smaller than the requested ACL.
- `denied`: the permissions that the user agent denied. The user agent omits this field if it granted the full request.

The `denied` field contains two fields:

- `acl`: the denied origins, commands, and events, in the ACL format.
- `reason`: an optional text from the user agent that explains the denial. The agent can use it to select a different approach.

We expect agents to request permissions only when necessary, and to retract them after use.

#### Example: first request

The agent requests its first permissions after `session.new`.

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

#### Example: more permissions for one navigable

The agent requests more permissions for navigable `A1B2C3`.

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

### `session.changed`

The user agent sends this event when the user or the user agent changes the session. There are two causes:

- The user pauses or resumes the session.
- The user changes the permissions. For example, the user revokes access to a navigable.

The event has three fields:

- `acl`: the updated ACL.
- `paused`: `true` if the user or the user agent paused the session.
- `reason`: an optional text from the user agent that explains the change, for example why it revoked a permission.

While the session is in the paused state, the agent must expect commands to fail. The connection stays open.

The user agent always sends `session.changed` to a restricted session. The ACL does not control this event.

In this example, the user pauses the session and revokes the permissions for navigable `A1B2C3`:

```json
{
  "type": "event",
  "method": "session.changed",
  "params": {
    "paused": true,
    "reason": "The user paused the session and revoked access to the tab.",
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

## Example: a full session

This sequence shows the expected permission flow for one task.

```mermaid
sequenceDiagram
    participant Agent
    participant UA as User agent
    participant User

    Agent->>UA: session.new (sessionType: restricted)
    UA-->>Agent: restricted session with an empty acl
    Agent->>UA: session.setAcl (first acl, reason)
    UA->>User: permission prompt (optional)
    User-->>UA: decision
    UA-->>Agent: granted acl
    Agent->>UA: commands in the default acl
    UA-->>Agent: results
    Agent->>UA: session.setAcl (more permissions for one navigable, reason)
    UA->>User: permission prompt (optional)
    User-->>UA: decision
    UA-->>Agent: granted and denied permissions, denial reason
    Agent->>UA: commands in the per-navigable acl
    UA-->>Agent: results and allowed events
    User->>UA: pause the session
    UA-->>Agent: session.changed (paused: true)
    User->>UA: resume the session
    UA-->>Agent: session.changed (paused: false)
    Agent->>UA: session.setAcl (retract the per-navigable permissions)
    UA-->>Agent: updated acl
```

## How this solution addresses the use cases

| Use case | Mechanism |
| :--- | :--- |
| 1. Only allowed operations | A restricted session starts with an empty ACL. The user agent enforces the `origins`, `commands`, and `events` lists of the ACL. |
| 2. User interrupts the session | `session.changed` with `paused: true`. |
| 3. User revokes permissions | `session.changed` with the updated `acl`. The connection stays open. |
| 4. Agent requests more capabilities | `session.setAcl` with a `reason`. The result contains the granted and denied permissions, and an optional reason for the denial. |

## Detailed design discussion

### Permissions are part of the protocol

An out-of-band permission mechanism is not interoperable. The agent also cannot react to it. Thus the protocol contains the request (`session.setAcl`), the result, and the change notification (`session.changed`). Each user agent selects how to make the decision. For example, it can use a user prompt, an enterprise policy, or its own analysis of the `reason`.

### Session type in `session.new`, permissions in `session.setAcl`

`session.new` only selects the restricted mode. All permission requests use `session.setAcl`. This design has these advantages:

- There is one code path for permission requests, results, and denials.
- Each permission request has a `reason` and occurs near the time of use.
- Session creation does not wait for a user decision.
- Capability matching in `session.new` does not need rules for a partial grant.

The cost is one more command before the agent can do its first operation.

### Commands and events are separate lists

Events can expose page data, for example console messages or URLs of navigations. Thus the ACL controls events as well as commands. Separate lists let the agent request an event without a related command.

### Default and per-navigable permissions

Most tasks operate on one tab or a small number of tabs. The `perNavigable` part lets the user grant more permissions for one tab only. The agent can continue to use the `default` permissions for other tabs. Additionally, user agent needs to know which tabs are being controlled by the agent.

### Open questions

1. **Session type.** `restricted` is a working name. Is `sessionType` a capability or a top-level parameter of `session.new`? Does a restricted session start with an empty ACL, or can the user agent supply a default ACL?
2. **Name of the ACL.** `acl` is a working name.
3. **Shortcut for full access.** The ACL can support a short form that requests all origins, all commands, or all events.
4. **Allowed set.** Which commands and events can an ACL of a restricted session contain? Should this be up to the user agent? Should we have a mechanism to advertise possible commands and events? Recommendation: setAcl returns current effective ACL that could be larger in scope that what the agent requests. This is a mechanism for the agent to know what can be done by default in the user agent.
5. **Entries for `perNavigable` only.** Which commands and events are valid only in a `perNavigable` entry? Recommendation: let use agent decide.
6. **Subscriptions.** How do the `events` list and `session.subscribe` interact? Recommendation: adding an event to ACL subscribes to the event. removing an event from ACL unsubscribes.
7. **Errors.** Which error code does a command return when the ACL denies it, or when the session is in the paused state?
8. **Prompt frequency.** Which permissions can a user agent grant without a user prompt? We think this should be left to the user agent discretion.
9. **Denial reason format.** Is one free-text `reason` sufficient? Alternatives are a machine-readable code, or one reason for each denied entry.

## Roadmap

We plan to explore these items after the permission model. The order does not show priority.

- **Accessibility tree.** Explore the accessibility tree as a structured text snapshot of page content for agents.
- **CSP for sandbox realms.** Explore a Content Security Policy (CSP) that applies to the sandbox realms in which the agent evaluates script.
- **`browsingContext.locateNodes` by coordinates.** Add a locator that finds nodes at an `x`, `y` position.
- **Extension and non-WebSocket transports.** Let a browser extension use the protocol, and support transports other than WebSocket.

## Considered alternatives

### Generic access control with allowlists and denylists

A more general capability can contain allowlists and denylists for modules, commands, events, and parameters. This alternative has more flexibility, but it is more complex to specify and to show to a user. This proposal uses allowlists only, and it adds origins, per-navigable scope, and in-protocol permission requests.

### ACL in `session.new`

An earlier version of this document accepted an ACL as a capability in `session.new`. The result contained the granted ACL. We replaced this design because it made two methods to request permissions. The request in `session.new` also had no `reason`, and a user prompt could delay session creation.

### Session type with permissions outside the protocol

The [original proposal](https://github.com/WICG/proposals/issues/320) also used a session type, `sessionType: "agent"`, and the session started with lifecycle commands only. That proposal did not specify how the agent gets more permissions. This document adds `session.setAcl` and `session.changed` for that purpose.

The user agent could manage all permissions in its own UI, with no protocol surface. Then the agent cannot request permissions, give a reason, or learn about a denial. The behavior is also different in each browser.

## Security and privacy considerations

- **Least privilege.** A restricted session starts with no permissions. It can use only the origins, commands, and events in its ACL. Standard WebDriver BiDi gives full access to the browser.
- **Enforcement in the user agent.** The user agent enforces the ACL. The agent and its client library are not trusted to do this.
- **Origin gating.** The agent cannot send commands to, or receive events from, documents of origins that the ACL does not contain.
- **User control.** The user can pause the session or revoke permissions at any time. The effect is immediate.
- **Limits.** Protocol permissions do not prevent all unwanted actions. An agent with permission to click can do a destructive action on an allowed origin.
- **Reason text from the agent.** The `reason` in `session.setAcl` comes from the agent and is not trusted. The user agent must not show it as a statement of fact.
- **Reason text from the user agent.** A denial reason can disclose information about the user or a policy. The user agent selects the content and can omit the reason.

## Stakeholder feedback

- **@jgraham**: supportive of the direction in the [proposal discussion](https://github.com/WICG/proposals/issues/320). The feedback includes these points:
  - The permission system must be part of the protocol.
  - Commands that match normal user actions must not need a prompt each time.
  - Permission requests must be near the time of use and must include a reason.
  - Script evaluation and network inspection are a concern for user-facing sessions.
- Other implementers: no signal yet.

## References and acknowledgements

- [WebDriver BiDi](https://www.w3.org/TR/webdriver-bidi/)
- [WICG proposal: Agentic extensions for WebDriver BiDi](https://github.com/WICG/proposals/issues/320)

Thanks to the participants of the proposal discussion for their feedback.
