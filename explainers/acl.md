# Access control lists

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

An access control list (ACL) defines the access of a [restricted session](restricted-sessions.md). This document describes the ACL format, the algorithm that checks whether the ACL allows a command or event, and the `session.setAcl` command. See [Actuation UI](actuation-ui.md) for how the ACL controls actuation UI, [Security model](security-model.md) for the security principles, and [Use cases](use-cases.md) for the user tasks behind this design.

The user agent sends the [`session.changed`](restricted-sessions.md#sessionchanged) event when the user or the user agent changes the ACL.

## The access control list

The ACL has two parts:

- `global` defines commands and events that do not target any navigable, such as `browsingContext.create`. Commands and events that target a navigable are not valid in `global`.
- `perNavigable` lists the allowed top-level traversables (navigables) and the commands and events allowed in each navigable.

Each part or entry contains two lists (see [#26](https://github.com/WICG/agentic-webdriver-bidi/issues/26) for full-access shortcuts):

- `commands`: the names of the commands that the agent can send.
- `events`: the names of the events that the agent can receive.

The `commands` and `events` lists can contain only the commands and events in [Commands and events in a restricted session](commands-and-events.md#commands-and-events-in-a-restricted-session).

```js
{
  global: {
    commands: [...commandName],
    events: [...eventName]
  },
  perNavigable: {
    "<navigable id>": {
      commands: [...commandName],
      events: [...eventName]
    }
  }
}
```

Access control is a property of the session. A navigable without a `perNavigable` entry has no access beyond user-agent defaults. An empty entry has the same effect as no entry.

The ACL does not list origins. The user agent enforces origin restrictions locally according to the [security model](security-model.md) (see [#49](https://github.com/WICG/agentic-webdriver-bidi/issues/49) on site-level scoping and opaque origins).

Event subscriptions are independent of the ACL `events` lists. The agent uses `session.subscribe` and `session.unsubscribe` as usual. The user agent filters out events that the effective ACL or origin rules do not allow before it emits them (see [#29](https://github.com/WICG/agentic-webdriver-bidi/issues/29) for subscriptions).

The ACL does not control these commands and events. They are always available in a restricted session:

- `session.status`
- `session.end`
- `session.subscribe`
- `session.unsubscribe`
- `session.setAcl`
- `session.changed`

## Checking if the ACL allows a command or event

When the agent sends a command or the user agent prepares an event, the user agent checks the session ACL and its origin rules (see [#30](https://github.com/WICG/agentic-webdriver-bidi/issues/30) for error codes, [#8](https://github.com/WICG/agentic-webdriver-bidi/issues/8) for pausing instead of aborting, and [#35](https://github.com/WICG/agentic-webdriver-bidi/issues/35) for pause/resume):

1. **Always-available session operations.** If the operation is `session.status`, `session.end`, `session.subscribe`, `session.unsubscribe`, `session.setAcl`, or `session.changed`, allow it.
2. **`browsingContext.getTree` without `root`.** Return only navigables `N` where `browsingContext.getTree` is in `acl.perNavigable[N].commands` and the user agent allows it for `N`'s document origin. To discover tabs outside `perNavigable` without extension APIs, the agent needs a dedicated tab-overview command and event or action feedback ([#5](https://github.com/WICG/agentic-webdriver-bidi/issues/5)).
3. **Global operations.** If the command or event does not target any navigable (such as `browsingContext.create`), allow it only if `acl.global` contains it.
4. **Navigable and origin check.** For a command targeting navigable `N` with document origin `O`, or an event from `(N, O)`:
   - Let `T` be the top-level traversable of `N`.
   - Require the name in `acl.perNavigable[T]`.
   - Require the user agent's origin rules to allow the operation on `O` (this also covers cross-origin iframes; see [#36](https://github.com/WICG/agentic-webdriver-bidi/issues/36) and [#50](https://github.com/WICG/agentic-webdriver-bidi/issues/50)). An initial `about:blank` top-level traversable with an opaque origin (such as a tab from `browsingContext.create`) passes this check. Step 5 still checks any navigation target origin.
5. **Navigation target check.** For a navigation or redirect to target origin `T`, require the user agent's origin rules to allow `T`. Otherwise block the navigation or redirect before sending the request.
6. **Subresources.** Subresource origins loaded by an allowed top-level document do not need separate origin approval, provided the session cannot run agent-authored script that sends requests to disallowed origins (see [#45](https://github.com/WICG/agentic-webdriver-bidi/issues/45)).

### Origin changes on a controlled navigable

When a controlled navigable `N` attempts or undergoes an origin change:

- **Navigation blocked before sending the request:** `N` stays on its current allowed document and remains in `perNavigable`. The user agent fails the command (or sends `session.changed` with a `reason`).
- **Navigable transitions to a disallowed origin** (for example, user navigation, an origin revocation, or manual checkout in [U9](use-cases.md#u9-the-user-completes-part-of-a-task-manually-instead-of-granting-access)): the user agent removes `N` from `perNavigable` and sends [`session.changed`](restricted-sessions.md#sessionchanged) with the updated `acl` and a `reason`.
- **Navigable transitions to an origin with narrower allowed operations:** the user agent removes the disallowed commands and events from `perNavigable[N]` and sends `session.changed` with the updated `acl` and a `reason`.

## `session.setAcl`

The agent sends `session.setAcl` to request or retract access. This command is the only method to change the session ACL.

Parameters:

- `acl`: the full ACL that the agent requests for the session.
- `reason`: an optional text from the agent that explains the request. The user agent may deny a request without a `reason`.

To process `session.setAcl`, the user agent:

1. Rejects invalid entries (such as navigable commands in `global`, global-only commands in `perNavigable`, or entries that the user agent's origin rules disallow for a navigable's current document origin). The user agent allows an initial `about:blank` top-level traversable with an opaque origin (such as a tab from `browsingContext.create`) in `perNavigable`. The agent can then grant commands such as `browsingContext.navigate` before step 5 checks the destination origin.
2. Requests user or user-agent approval for any new access in the diff between the current ACL and the requested `acl` (using `params.reason` if provided).
3. Sets the session ACL to the granted entries plus any user-agent-default access, removes empty entries, and updates the [actuation UI](actuation-ui.md).
4. Returns `{ acl }`. If any requested item was not granted, includes `denied: { acl, reason }` with the denied entries in ACL format and an optional `reason`.

Agents must request access only when necessary and retract it after use, especially for actuation commands that control the [actuation UI](actuation-ui.md).

See [#48](https://github.com/WICG/agentic-webdriver-bidi/issues/48) for ordering of concurrent `session.setAcl` calls.

## Detailed design discussion

### Commands and events are separate lists

Events can expose page data, for example console messages or URLs of navigations. Thus the ACL controls events as well as commands. Separate lists let the agent request an event without a related command.

### `perNavigable` ACL with user-agent origin enforcement

The [security model](security-model.md) requires restrictions by both top-level traversable and security origin:

- `global` holds only commands and events that do not target any navigable (such as `browsingContext.create`). It never grants access to any navigable.
- `perNavigable` scopes access to specific tabs. It blocks access to other user tabs ([U1](use-cases.md#u1-the-agent-uses-only-the-access-that-the-task-needs), [U7](use-cases.md#u7-the-user-adds-a-specific-tab-to-the-agent-session)), lets the user revoke one tab without affecting another ([U3](use-cases.md#u3-the-user-revokes-access-during-the-task)), supports different access levels per tab ([U5](use-cases.md#u5-the-agent-reads-one-tab-and-acts-in-a-different-tab)), and signals which tabs need [actuation UI](actuation-ui.md).
- The user agent enforces origin rules locally and communicates origin denials or revocations through `denied.reason` in `session.setAcl` or by updating `perNavigable` and sending `session.changed` with a `reason`.

Keeping origin allowlists and blocklists in the user agent avoids duplicating origin policies in the protocol, avoids disclosing origin lists to the agent, and keeps the ACL format simple. All controlled tabs still share the agent context window, so `perNavigable` limits operations per tab rather than isolating data between controlled tabs.

## Considered alternatives

### Per-origin entries in the ACL

Earlier designs placed origins in the ACL in two ways:

- **A `perOrigin` map alongside `perNavigable` (or an `origins` list inside each entry):** This design added cross-map resolution rules, inheritance and narrowing semantics, and edge cases on temporary navigations to unlisted origins.
- **Origin maps nested under each navigable (`perNavigable[navigableId][origin]`):** This design avoids cross-map ambiguity and lets an agent request a destination origin with a `reason` before navigation. It also ties the protocol to origin-or-site syntax ([#49](https://github.com/WICG/agentic-webdriver-bidi/issues/49)) and repeats command lists across `(navigable, origin)` pairs.

Keeping origin rules in the user agent and reporting denials via `session.changed` (with `reason`) and `denied.reason` keeps the protocol simple and enforces the security model.

### Generic access control with allowlists and denylists

A more general capability can contain allowlists and denylists for modules, commands, events, and parameters. This alternative has more flexibility, but it is more complex to specify and to show to a user. This proposal uses allowlists only, with `global` and `perNavigable` scopes and in-protocol access requests.

### ACL in `session.new`

An earlier version of this document accepted an ACL as a capability in `session.new`. The result contained the granted ACL. We replaced this design because it made two methods to request access. The request in `session.new` also had no `reason` parameter, and a user prompt could delay session creation.
