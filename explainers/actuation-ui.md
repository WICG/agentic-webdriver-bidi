# Actuation UI

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

User agents are encouraged to show dedicated UI while an agent actuates a navigable in a [restricted session](restricted-sessions.md) ([#6](https://github.com/WICG/agentic-webdriver-bidi/issues/6)). For example, the user agent can:

- Show that the agent controls the navigable.
- Show other UI that helps the user follow the agent, for example click visualization.
- Show a pause control that lets the user pause and resume the agent ([U2](use-cases.md#u2-the-user-interrupts-the-agent-and-changes-the-page)).
- Show a stop control that lets the user terminate the agent session immediately ([U6](use-cases.md#u6-the-user-terminates-the-agent-session-with-one-action)).

## ACL as the signal

The [ACL](acl.md) is the signal for this UI. A navigable is an actuated navigable while its `perNavigable` entry in the effective ACL contains at least one actuation command. Thus:

- The actuation starts when `session.setAcl` grants an actuation command for the navigable.
- The actuation ends when the effective ACL no longer contains an actuation command for the navigable. This occurs when the agent retracts the access with `session.setAcl`, or when the user or the user agent revokes it. It also occurs when the navigable closes (see [#37](https://github.com/WICG/agentic-webdriver-bidi/issues/37)) or the session ends.

The actuation continues across navigations in the navigable, and across commands that are not actuation commands. To run an actuation command in that navigable, the user agent's origin rules must also allow the command on the target document origin. The agent can request access to more than one navigable at the same time. Each of these navigables is an actuated navigable.

Actuation commands always target a navigable, so they are not valid in `global`. A navigable is an actuated navigable only while its `perNavigable` entry in the effective ACL contains at least one actuation command. Thus the user agent always knows which navigables the agent can actuate.

## Actuation commands

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

An agent that only observes a navigable does not cause actuation UI. For example, a `perNavigable` entry with only `browsingContext.captureScreenshot` does not make the navigable an actuated navigable (see [U5](use-cases.md#u5-the-agent-reads-one-tab-and-acts-in-a-different-tab)).

## User controls and retraction

When the user activates the pause control, the user agent pauses the session and sends [`session.changed`](restricted-sessions.md#sessionchanged) with `paused: true`. The user agent can also revoke access from the ACL. When the user activates the stop control, the user agent terminates the session immediately and revokes all access, including inflight requests whenever possible.

The agent must retract the actuation commands for a navigable when it completes the actuation. Otherwise the user agent continues to show the actuation UI. The protocol cannot enforce this. The user agent can revoke access at any time, for example after a period of inactivity (see [#34](https://github.com/WICG/agentic-webdriver-bidi/issues/34)), and send `session.changed`.

## Detailed design discussion

### The ACL is the signal for actuation UI

The user agent must know when to show actuation UI. If the user agent finds this out from the commands that it receives, the UI is not reliable. For example, it can show the UI after an `input.performActions` command and remove it after a timeout. Then the UI shows after the first action, not before it. The UI can also flicker between two actions of one task.

The per-navigable ACL does not have these problems. The agent requests actuation commands for a navigable before the first action, and retracts them when the task is complete. The actuation continues across more than one `session.setAcl` call, and across commands that are not actuation commands. Thus the user agent can show the UI before the first action and keep it until the agent retracts the access.

This design does not add commands to the protocol. The ACL already contains the necessary information.

The cost is that the actuation UI depends on the agent: the agent must retract the actuation commands after use. The protocol cannot enforce this, but the user agent can revoke access at any time.
