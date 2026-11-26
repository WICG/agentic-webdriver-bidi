# Security model

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

This document describes the security model for agentic browsing in [restricted sessions](restricted-sessions.md). It defines the security principles, the required configuration capabilities, and related use cases.

## Principles

### Constrain what the agent can see and do

The user agent must constrain what the agent can observe and what it can actuate. An agent can transfer data and trigger actions across all security origins that it can access through its context window. This capability bypasses the Same-Origin Policy of the web platform. Therefore, the user agent restricts observations and actuations by security origin and by top-level traversable.

### Origin-level isolation

The web platform isolates distinct security origins from each other by default. Users, enterprise administrators, and agent providers must be able to restrict agent observations and actuations for any security origin (see [#49](https://github.com/WICG/agentic-webdriver-bidi/issues/49) on site-level scoping and opaque origins).

### Least privilege by default

A restricted session starts with no access beyond user-agent defaults. If neither the configuration nor user-agent defaults allow a security origin and top-level traversable, the user agent blocks all observations and actuations.

### Shared data boundary across controlled tabs

All top-level traversables (for example, browser tabs) under the control of an agent share the same data-access boundary. The agent can transfer data from one top-level traversable to another through its context window. Therefore, enforcing different access rules on different top-level traversables does not prevent cross-tab data leaks. Per-traversable restrictions still limit which observations and actuations the agent can perform in each top-level traversable (see [U5](use-cases.md#u5-the-agent-reads-one-tab-and-acts-in-a-different-tab)). The user agent allows an observation or actuation only if both the top-level traversable and the security origin of the target document are allowed.

### User and enterprise configurability

Users and enterprise administrators must be able to configure agent access controls to protect sensitive data. The user agent enforces these policies locally and does not rely on the agent to honor them.

### Focus on top-level page origins

Top-level pages often depend on cross-origin subresources, such as images, stylesheets, and APIs, to function. In normal browsing, web-platform boundaries (such as the Same-Origin Policy, CORS, and `SameSite` cookies) already govern what an allowed top-level document can embed or read from cross-origin subresources, even though rendered subresources such as images appear in screenshots. Therefore, the access control configuration does not need to list subresource origins loaded by the page itself, provided the session cannot execute arbitrary agent-authored script that initiates outbound subresource requests to unlisted origins (see [#45](https://github.com/WICG/agentic-webdriver-bidi/issues/45)).

Cross-origin iframes are an exception. Embedded iframes are separate documents with their own origin boundary: user-like actions, such as clicks, can act directly on a cross-origin iframe, and screenshots can capture cross-origin iframe documents that the top-level page itself cannot inspect. The user agent must enforce origin restrictions on cross-origin iframes as well (see [#36](https://github.com/WICG/agentic-webdriver-bidi/issues/36) and [#50](https://github.com/WICG/agentic-webdriver-bidi/issues/50)).

## Required configuration capabilities

The access control configuration must allow users and administrators to enforce the following restrictions:

- **Block navigation to origins that are not allowlisted.** Navigations to an agent-chosen origin can exfiltrate sensitive data from the agent context, for example through URL query parameters. The user agent blocks such navigations, including redirects, before it sends the request.
- **Block all access to origins that are not allowlisted.** The agent cannot observe or actuate documents from those origins.
- **Block a specific browser-provided action globally.** The agent cannot invoke that action on any origin.
- **Block a specific browser-provided action on origins that are not allowlisted for that action.** The agent can invoke that action on allowlisted origins, but not on other origins.
- **Block tab takeover on origins that are not allowlisted.** The agent cannot take control of an existing tab that displays a document from an origin that is not allowlisted.
- **Block observations or actuations on top-level traversables that are not allowlisted.** The agent cannot observe or actuate a top-level traversable (for example, a browser tab) unless that traversable is allowlisted for that access, even if it displays an allowlisted origin. This allows configuring observation-only access or both observation and actuation per traversable.

## Related use cases

The [use cases](use-cases.md) document describes concrete user tasks that rely on this security model:

- **Least privilege and blocked tab takeover:** [U1](use-cases.md#u1-the-agent-uses-only-the-access-that-the-task-needs) and [U7](use-cases.md#u7-the-user-adds-a-specific-tab-to-the-agent-session) limit the agent to approved origins and tabs and block access to other user tabs.
- **Scoped multi-origin access and navigation blocking:** [U3](use-cases.md#u3-the-user-revokes-access-during-the-task) and [U4](use-cases.md#u4-the-task-needs-more-access-than-the-agent-has) allow access to approved origins and block access to denied or revoked origins.
- **Per-origin action restrictions:** [U5](use-cases.md#u5-the-agent-reads-one-tab-and-acts-in-a-different-tab) allows read-only access in one tab while allowing actions on another origin. [U8](use-cases.md#u8-the-user-confirms-the-submission-of-a-form-that-the-agent-filled-out) scopes input access to a specific site and tab.
