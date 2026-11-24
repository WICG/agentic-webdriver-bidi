# Agentic extensions for WebDriver BiDi

## Authors

- Alex Rudenko (Google)
- Maksim Sadym (Google)

## Participate

- [Issue tracker](https://github.com/WICG/agentic-webdriver-bidi/issues)
- [Original proposal and discussion](https://github.com/WICG/proposals/issues/320)

## Explainers

The proposal has four documents:

| Document | Content |
| :--- | :--- |
| [Restricted sessions](explainers/restricted-sessions.md) | The session model: terminology, `session.new`, `session.changed`, a full session example, design decisions, and security considerations. |
| [Commands and events](explainers/commands-and-events.md) | The list of commands and events that a restricted session supports. |
| [Access control lists](explainers/acl.md) | The ACL format, `session.setAcl`, enforcement, and actuation UI. |
| [Extension transport](explainers/extension-transport.md) | The `browser.driver` WebExtensions API that carries WebDriver BiDi messages without a network endpoint. |

## Introduction

[WebDriver BiDi](https://www.w3.org/TR/webdriver-bidi/) is a browser automation protocol. Its primary purpose is automated browser testing. The protocol assumes complete trust between the client and the browser. A client gets full access to the browser instance and can do destructive actions without security checks.

AI agents also need browser automation. Many agents use low-level protocols or test setups that give unrestricted access without user supervision.

We propose extensions to WebDriver BiDi for agentic use cases that include agentic browsing. A WebDriver BiDi session represents one long-running agentic activity in the browser. Access control is a property of that session. The user agent enforces the access control. The user keeps supervision and control of the agent.

A client starts a restricted session with `sessionType: "restricted"` in `session.new`. A restricted session starts with no access beyond any user-agent defaults. The agent requests access and learns the effective ACL with `session.setAcl`.

An access control list (ACL) defines the access of a restricted session. The ACL contains three types of entries:

- **Origins**: the origins that the agent can access.
- **Commands**: the WebDriver BiDi commands that the agent can send.
- **Events**: the WebDriver BiDi events that the agent can receive.

The ACL also tells the user agent which navigables the agent actuates. A navigable is actuated while its per-navigable ACL entry contains an actuation command, for example a click or a navigation. The user agent can show actuation UI for that navigable, and the user can stop the agent.

> [!NOTE]
> This proposal is in incubation. All names and data shapes in this document are illustrative and can change (see [#13](https://github.com/WICG/agentic-webdriver-bidi/issues/13) on terminology). The goal is to move the final features into the WebDriver BiDi specification.

## Goals

1. Let the user supervise and control agent actions in the browser.
2. Keep the agent informed about user decisions.
3. Make the access control mechanism a part of the protocol, so that it is interoperable between browsers.
4. Adapt existing WebDriver BiDi commands and add new commands for agentic use cases.

## Non-goals

1. Testing-specific and debugging-specific features. Standard WebDriver BiDi sessions continue to serve these use cases.
2. A definition of the browser UI for access prompts. Each user agent selects how it shows access requests to the user.
3. Restrictions on high-level user intent. For example, the protocol cannot know that the user does not want the agent to delete an account.

## Use cases

1. A restricted session can use only the operations that the user or the user agent allowed.
2. The user interrupts an active session. The agent receives a notification and adapts.
3. The user revokes access or pauses the session. The session loses the related capabilities immediately, but the connection stays open.
4. The agent requests more capabilities during a session. The user or the user agent makes the decision, and the agent receives the result.
5. The user sees which navigables the agent can actuate. The agent can observe a navigable without actuation UI, for example to read the page in a tab that the user opened.
6. The user stops the agent with one action, for example a stop button. The agent receives a notification.

The [original proposal](https://github.com/WICG/proposals/issues/320) contains more use cases: structured page content, feedback on input, and access to user profile data. They are in scope.

## Open questions

Open questions are tracked in the [issue tracker](https://github.com/WICG/agentic-webdriver-bidi/issues).

## Stakeholder feedback

- **@jgraham**: supportive of the direction in the [proposal discussion](https://github.com/WICG/proposals/issues/320). The feedback includes these points:
  - The access control system must be part of the protocol.
  - Commands that match normal user actions must not need a prompt each time.
  - Access requests must be near the time of use and must include a reason.
  - Script evaluation and network inspection are a concern for user-facing sessions.
- Other implementers: no signal yet.

## References and acknowledgements

- [WebDriver BiDi](https://w3c.github.io/webdriver-bidi/)
- [WICG proposal: Agentic extensions for WebDriver BiDi](https://github.com/WICG/proposals/issues/320)

Thanks to the participants of the proposal discussion for their feedback.
