# Security model

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

This document describes the security model for agentic browsing in [restricted sessions](restricted-sessions.md). It defines the security principles, the required configuration capabilities, and related use cases.

## Principles

### Constrain what the agent can see and do

The user agent must constrain what the agent can see and what it can do. An agent can transfer data and trigger actions across all origins that it can access through its context window. This capability bypasses the Same-Origin Policy of the web platform. Therefore, the configuration must explicitly list the origins that connect to the agent context.

### Origin-level isolation

The web platform isolates distinct origins from each other by default. Users, enterprise administrators, and agent providers must be able to protect any origin from the agent with origin-level [access control lists (ACLs)](acl.md) (see [#49](https://github.com/WICG/agentic-webdriver-bidi/issues/49) on site-level scoping and opaque origins).

### Least privilege by default

A restricted session starts with no access beyond user-agent defaults. If the configuration does not list an origin and the user-agent defaults do not allow it, the user agent blocks all actions for that origin.

### Shared data boundary across controlled tabs

All tabs under the control of an agent share the same data-access boundary. The agent can transfer data from one tab to another through its context window. Therefore, enforcing different ACLs on different tabs does not prevent cross-tab data leaks. Per-tab ACLs still limit which actions the agent can run in each tab (see [U5](use-cases.md#u5-the-agent-reads-one-tab-and-acts-in-a-different-tab)).

### User and enterprise configurability

Users and enterprise administrators must be able to configure agent access controls to protect sensitive data. The user agent enforces these policies locally and does not rely on the agent to honor them.

### Focus on top-level page origins

Top-level pages often depend on cross-origin subresources, such as images, stylesheets, and APIs, to function. In normal browsing, web-platform boundaries (such as the Same-Origin Policy, CORS, and `SameSite` cookies) already govern what an allowed top-level document can embed or read from cross-origin subresources, even though rendered subresources such as images appear in screenshots. Therefore, the ACL does not need to list subresource origins loaded by the page itself, provided the session cannot execute arbitrary agent-authored script that initiates outbound subresource requests to unlisted origins (see [#45](https://github.com/WICG/agentic-webdriver-bidi/issues/45)).

Cross-origin iframes are an exception. Embedded iframes are separate documents with their own origin boundary: user-like actions, such as clicks, can act directly on a cross-origin iframe, and screenshots can capture cross-origin iframe documents that the top-level page itself cannot inspect. The user agent must enforce origin restrictions on cross-origin iframes as well (see [#36](https://github.com/WICG/agentic-webdriver-bidi/issues/36) and [#50](https://github.com/WICG/agentic-webdriver-bidi/issues/50)).

## Required configuration capabilities

The access control configuration must allow users and administrators to enforce the following restrictions:

- **Block navigation to origins that are not allowlisted.** Navigations to an agent-chosen origin can exfiltrate sensitive data from the agent context, for example through URL query parameters. The user agent blocks such navigations, including redirects, before it sends the request.
- **Block all access to a specific origin.** The agent cannot read or act on documents from that origin.
- **Block a specific browser-provided action globally.** The agent cannot invoke that action on any origin.
- **Block a specific browser-provided action on a specific origin.** The agent can invoke that action on allowed origins, but not on restricted origins.
- **Block tab takeover on a specific origin.** The agent cannot take control of an existing tab that displays a document from that origin.

## Related use cases

The [use cases](use-cases.md) document describes concrete user tasks that rely on this security model:

- **Least privilege and blocked tab takeover:** [U1](use-cases.md#u1-the-agent-uses-only-the-access-that-the-task-needs) and [U7](use-cases.md#u7-the-user-adds-a-specific-tab-to-the-agent-session) limit the agent to approved origins and tabs and block access to other user tabs.
- **Scoped multi-origin access and navigation blocking:** [U3](use-cases.md#u3-the-user-revokes-access-during-the-task) and [U4](use-cases.md#u4-the-task-needs-more-access-than-the-agent-has) allow access to approved origins and block access to denied or revoked origins.
- **Per-origin action restrictions:** [U5](use-cases.md#u5-the-agent-reads-one-tab-and-acts-in-a-different-tab) allows read-only access in one tab while allowing actions on another origin. [U8](use-cases.md#u8-the-user-confirms-the-submission-of-a-form-that-the-agent-filled-out) scopes input access to a specific site and tab.
