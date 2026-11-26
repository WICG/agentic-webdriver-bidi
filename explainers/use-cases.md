# Use cases

> [!NOTE]
> This document is a part of the [Agentic extensions for WebDriver BiDi](../README.md#introduction) proposal. The proposal is in incubation.

This document describes the use cases of the proposal.

Each use case contains a user task and the behavior that the user expects. The tasks and the sites are examples.

The use cases do not describe a protocol design. These documents describe how the proposal supports the use cases:

- [Restricted sessions](restricted-sessions.md)
- [Commands and events](commands-and-events.md)
- [Access control lists](acl.md)

## U1. The agent uses only the access that the task needs

**User task:** "Find a blue backpack on shop.example."

**Expected behavior:**

- The agent starts with no access to the tabs and sites of the user.
- The agent asks for access to shop.example and to the tab that it intends to use. It gives a reason.
- The user or the user agent makes the decision on the access request.
- The agent can use only the access that it received. It cannot read or change other sites or other tabs.
- The agent decides when the task is complete. It then tells the user and retracts its access.
- The user agent does not rely on the agent to retract the access. The user or the user agent can revoke the access at any time.

## U2. The user interrupts the agent and changes the page

**User task:** "Book a table for two at restaurant.example for Friday." During the task, the user decides to change the time in the form.

**Expected behavior:**

- The user can pause the agent at any time.
- The agent knows that the user paused it. The user agent does not let the agent act on the page during the pause.
- The user changes the time in the form and resumes the agent.
- The agent knows that the user resumed it. It continues the task from the current state of the page.

## U3. The user revokes access during the task

**User task:** "Compare the price of this laptop on shop-a.example and shop-b.example." During the task, the user decides that the agent must not use shop-b.example.

**Expected behavior:**

- The agent has previously gained access to two tabs, one for each shop, and to the site of each shop.
- The user can revoke the access to the shop-b.example tab at any time. The agent loses that access immediately including inflight requests whenever possible.
- The agent knows which access the user revoked.
- The agent keeps its access to the shop-a.example tab and continues the task there.
- The agent tells the user that the comparison is not complete.

## U4. The task needs more access than the agent has

### U4.1. The task needs access to a new tab on an allowed site

**User task:** "Compare two hotels in Berlin for the weekend on travel.example."

**Expected behavior:**

- The agent has access to travel.example in one tab only.
- During the task, clicking a hotel listing opens a second hotel page in a new tab on travel.example. The agent has no access to the new tab.
- The agent can ask for access to the new tab at the time of use. The request names the site and the tab, and gives a reason.
- The user or the user agent grants or denies the request.
- The agent knows the result of the request and the reason for a denial.
- If the user or the user agent grants the request, the agent compares both tabs. After a denial, the agent continues with the first tab and tells the user that the comparison is not complete.

### U4.2. The task needs access to a new site in the same tab

**User task:** "Find a hotel in Berlin for the weekend on travel.example and tell me about its reviews and prices."

**Expected behavior:**

- The agent has access to travel.example in one tab only.
- During the task, the agent clicks links in that same tab to reviews.example and booking.example. It has no access to these sites.
- The agent can ask for access to each new site in that tab at the time of use. The request names the site and the tab, and gives a reason.
- The user or the user agent grants or denies each request. In this example, it grants reviews.example and denies booking.example.
- The agent knows the result of each request and the reason for a denial.
- After a denial, the agent continues with the access that it has. It tells the user that the prices are missing.

## U5. The agent reads one tab and acts in a different tab

**User task:** "Read the recipe in my current tab and order the ingredients on grocery.example."

**Expected behavior:**

- The agent asks for access to the recipe tab and to the grocery.example tab. Each request names the site and the tab. The user agent can show different UI for tabs that the agent can read and tabs in which the agent can act.
- The agent reads the recipe tab but does not act in it.
- The agent acts in the grocery.example tab.
- The user always sees in which tabs the agent can act.
- When the agent completes its actions in a tab, the user agent no longer shows that tab as controlled.

## U6. The user terminates the agent session with one action

**User task:** "Unsubscribe me from the newsletters in mail.example." During the task, the user sees that the agent opens the wrong message.

**Expected behavior:**

- The user can terminate the agent session with one action, for example a stop button.
- The agent loses all access immediately, including inflight requests whenever possible.
- This is different from a pause (U2): the agent session terminates.
- The agent knows that the user terminated the session. It can tell the user which steps it did not complete.
- A new session starts with no access.

## U7. The user adds a specific tab to the agent session

**User task:** "Summarize the report in this tab." The user adds the tab with the report to the agent session in the user agent UI.

**Expected behavior:**

- The user can give the agent access to a specific tab. The agent does not have to ask for it.
- The agent knows which tab the user added and which access it has there.
- The agent has no access to the other tabs of the user.
- If the task needs more access, for example a different site or tab, the agent can ask for it (U4).

## U8. The user confirms the submission of a form that the agent filled out

**User task:** "Fill out this long form, but ask me to review it before you submit it."

**Expected behavior:**

- The user grants the agent input access to the site and the tab of the form.
- The agent fills out the form. Then, the agent UI asks the user to confirm the submission.
- The user confirms the submission in the agent UI. The agent then submits the form with WebDriver BiDi input commands.
- WebDriver BiDi cannot know if an input command causes a form submission or a different state change in the site backend. Thus, the agent decides when to ask the user for confirmation.
    - For specific cases, where the browser knows that a state change is going to happen, the user agent may as for a confirmation in the browser UI.

## U9. The user completes part of a task manually instead of granting access

**User task:** "Buy the running shoes in my cart on shop.example and save the order number." During checkout, the site redirects to pay.example. The user does not want to grant the agent access to pay.example. Instead, the user wants to complete the payment manually without leaving the browser.

**Expected behavior:**

- The agent has access to shop.example in the tab. It starts checkout on shop.example.
- The site redirects the tab to pay.example, where the agent has no access.
- The agent asks for access to pay.example to complete the payment.
- The user denies the access request and pauses the agent session to take over ([U2](#u2-the-user-interrupts-the-agent-and-changes-the-page)).
- The agent knows that the user denied the request and paused the session. It cannot read or act on pay.example.
- The user completes the payment manually in the same tab without leaving the browser.
- After payment, the tab returns to shop.example, and the user resumes the agent.
- The agent continues the task on shop.example from the current state of the tab. It reads the order number and uses only the access that it has.
