# Use cases

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation. All names and data shapes can change.

This document describes the use cases of the proposal.

Related documents:

- [Restricted sessions](restricted-sessions.md) describes the session model, `session.new`, and `session.changed`.
- [Commands and events](commands-and-events.md) lists the commands and events that a restricted session supports.
- [Access control lists](acl.md) describes the access of a restricted session, `session.setAcl`, and the actuation UI.

Each use case contains a user task and the agent workflow that completes the task. The tasks and the origins are examples.

The workflows show how the current proposal can complete each task. They are illustrations, not requirements. A different design can complete the same task with different commands and events.

## U1. The agent uses only the access that the task needs

**User task:** "Find a blue backpack on shop.example."

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The session has no access beyond the user-agent defaults.
2. The agent sends `session.setAcl`. The request contains the `https://shop.example` origin, the necessary commands and events, and a reason.
3. The user or the user agent grants the request. The agent receives the effective ACL.
4. The agent searches the shop. The user agent rejects each operation that is not in the ACL, for example a command for a different origin.
5. The agent reports the result to the user.
6. When the task is complete, the agent sends `session.end`. The user agent ends the session and removes all access.

## U2. The user interrupts the agent and changes the page

**User task:** "Book a table for two at restaurant.example for Friday." During the task, the user decides to change the time in the form.

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The session has no access beyond the user-agent defaults.
2. The agent requests access to `https://restaurant.example` with `session.setAcl`. The user or the user agent grants the request.
3. The agent fills in the booking form.
4. The user pauses the session. The agent receives `session.changed` with `paused: true`.
5. The agent stops sending commands. The user changes the time in the form.
6. The user resumes the session. The agent receives `session.changed` with `paused: false`.
7. The agent reads the page again and completes the booking from the new page state.
8. When the task is complete, the agent sends `session.end`. The user agent ends the session and removes all access.

## U3. The user revokes access during the task

**User task:** "Compare the price of this laptop on shop-a.example and shop-b.example." During the task, the user decides that the agent must not use shop-b.example.

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The session has no access beyond the user-agent defaults.
2. The agent requests access to the two origins with `session.setAcl`. The user or the user agent grants the request.
3. The agent reads the price on shop-a.example.
4. The user revokes the access to `https://shop-b.example` in the user agent UI.
5. The agent receives `session.changed` with the updated ACL. The connection and stays open.
6. The agent no longer sends commands for shop-b.example. The user agent rejects such commands.
7. The agent reports the price from shop-a.example and tells the user that the comparison is not complete.
8. When the task is complete, the agent sends `session.end`. The user agent ends the session and removes all access.

## U4. The task needs more access than the agent has

**User task:** "Find a hotel in Berlin for the weekend on travel.example and tell me about its reviews and prices."

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The session has no access beyond the user-agent defaults.
2. The agent sends `session.setAcl`. The request contains the `https://travel.example` origin and input commands for the tab. The user grants the request.
3. The agent clicks links on travel.example and finds a hotel.
4. The reviews link goes to `https://reviews.example`. The ACL does not contain this origin, so commands for its documents fail.
5. The agent sends `session.setAcl` and adds `https://reviews.example`. The reason says that the task needs the hotel reviews.
6. The user or the user agent grants the request. The agent clicks the link and reads the reviews.
7. The prices link goes to `https://booking.example`. The agent requests this origin in the same way.
8. The user or the user agent denies the request. The agent receives the denied access and the reason for the denial.
9. The agent does not use booking.example. It reports the hotel and the reviews, and tells the user that the prices are missing.
10. When the task is complete, the agent sends `session.end`. The user agent ends the session and removes all access.

## U5. The agent reads one tab and acts in a different tab

**User task:** "Read the recipe in my current tab and order the ingredients on grocery.example."

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The session has no access beyond the user-agent defaults.
2. The agent requests observation commands for the recipe tab with `session.setAcl`. The user agent shows no actuation UI for that tab.
3. The agent reads the list of ingredients.
4. The agent requests actuation commands for the grocery.example tab. The user agent shows actuation UI for that tab only.
5. The user sees which tab the agent can actuate. The agent adds the ingredients to the cart.
6. The agent retracts the actuation commands with `session.setAcl`. The user agent removes the actuation UI.
7. When the task is complete, the agent sends `session.end`. The user agent ends the session and removes all access.

## U6. The user stops the agent with one action

**User task:** "Unsubscribe me from the newsletters in mail.example." During the task, the user sees that the agent opens the wrong message.

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The session has no access beyond the user-agent defaults.
2. The agent requests actuation commands for the mail.example tab with `session.setAcl`. The user or the user agent grants the request.
3. The agent actuates the tab. The user agent shows actuation UI with a stop button.
4. The user clicks the stop button. The user agent ends the WebDriver BiDi session and closes the connection.
5. The user agent removes all access of the session and the actuation UI.
6. The agent detects that the connection is closed. 
7. The agent reports the aborted steps to the user and can start a new session if needed.

## U7. The user adds a specific tab to the agent session

**User task:** "Summarize the report in this tab." The user adds the tab with the report to the agent session in the user agent UI.

**Agent workflow:**

1. The agent starts a restricted session with `session.new`. The result confirms the session type.
2. Immediately after the `session.new` result, the agent receives `session.changed`. The agent did not send `session.setAcl`.
3. The ACL in the event contains a per-navigable entry for the tab that the user added.
4. The agent reads the report with the commands in that entry and gives the summary to the user.
5. If the task needs more access, the agent requests it with `session.setAcl`.
6. When the task is complete, the agent sends `session.end`. The user agent ends the session and removes all access.
