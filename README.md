# Agentic extensions for WebDriver BiDi

## Authors

- Alex Rudenko (Google)
- Maksim Sadym (Google)

## Participate

- [Issue tracker](https://github.com/WICG/agentic-webdriver-bidi/issues)
- [Original proposal and discussion](https://github.com/WICG/proposals/issues/320)

## Introduction

[WebDriver BiDi](https://www.w3.org/TR/webdriver-bidi/) is a browser automation protocol. Its primary purpose is automated browser testing. The protocol assumes complete trust between the client and the browser. A client gets full access to the browser instance and can do destructive actions without security checks.

AI agents also need browser automation. Many agents use low-level protocols or test setups that give unrestricted access without user supervision.

We propose extensions to WebDriver BiDi for agentic use cases that include agentic browsing. A WebDriver BiDi session represents one agentic task in the browser. Access control is a property of that session. The user agent enforces the access control. The user keeps supervision and control of the agent.

> [!NOTE]
> This proposal is in incubation. All names and data shapes in documents in this repo are illustrative and can change. The goal is to move the final features into the WebDriver BiDi specification.

## Goals

1. Let the user supervise and control agent actions in the browser.
2. Keep the agent informed about user decisions.
3. Make the access control mechanism a part of the protocol, so that it is interoperable between browsers.
4. Adapt existing WebDriver BiDi commands and add new commands for agentic use cases.

## Non-goals

1. Testing-specific and debugging-specific features. Standard WebDriver BiDi sessions continue to serve these use cases.
2. A definition of the browser UI for access prompts. Each user agent selects how it shows access requests to the user.
3. Restrictions on high-level user intent. For example, the protocol cannot know that the user does not want the agent to delete an account.

## Explainers

The explainers describe the design:

| Document | Content |
| :--- | :--- |
| [Use cases](explainers/use-cases.md) | The user tasks that the proposal supports and the behavior that the user expects. |
| [Security model](explainers/security-model.md) | The security principles and the required configuration capabilities. |
| [Restricted sessions](explainers/restricted-sessions.md) | The session model: terminology, `session.new`, `session.changed`, a full session example, design decisions, and security considerations. |
| [Access control lists](explainers/acl.md) | The ACL format, the algorithm that checks a command or event against the ACL, and `session.setAcl`. |
| [Commands and events](explainers/commands-and-events.md) | The list of commands and events that a restricted session supports. |
| [Actuation UI](explainers/actuation-ui.md) | The UI that the user agent shows while an agent actuates a navigable, and how the ACL controls that UI. |
| [Extension transport](explainers/extension-transport.md) | The `browser.driver` WebExtensions API that carries WebDriver BiDi messages without a network endpoint. |

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
