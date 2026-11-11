# Requirements — PER-234

**Reviewers can set a case's priority**

## Given/When/Then acceptance criteria

### Setting the priority

**AC-1: Allowed priority values**

- **Given** a case that is waiting for answers to flags (`status` is `needs_input`) or waiting for final review (`status` is `needs_review`)
- **When** a reviewer sets the priority to `high`, `medium`, or `low`
- **Then** the case's priority is updated to the chosen value

**AC-2: Priority requires a reason**

- **Given** a case whose priority can be set
- **When** a reviewer attempts to set the priority with a reason that is empty, or contains only spaces, tabs, or line breaks
- **Then** the request is refused with an error message, and the case's priority remains unchanged

**AC-3: Reason length limit**

- **Given** a case whose priority can be set
- **When** a reviewer sets the priority with a reason of up to 300 characters
- **Then** the priority is set and the reason is stored

**AC-4: Invalid priority values rejected**

- **Given** a case whose priority can be set
- **When** a reviewer attempts to set the priority to a value other than `high`, `medium`, or `low`
- **Then** the request is refused with a message naming the allowed values: `high`, `medium`, `low`

### After setting the priority

**AC-5: Priority display**

- **Given** a reviewer has set a case's priority
- **When** the case is displayed
- **Then** the case shows the new priority, indicates that a reviewer set it, and displays the reason

**AC-6: Reviewer priority persists**

- **Given** a case whose priority was set by a reviewer
- **When** a flag is raised later, or the case is planned again
- **Then** the priority remains as the reviewer set it — raising a flag does not raise the priority one level, and re-planning does not replace it

**AC-7: History tracking**

- **Given** a case whose priority was changed
- **When** the case's history is viewed
- **Then** the history shows each priority change: the old priority (`priority` field before the change), the new priority, and the reason

**AC-8: Updating a reviewer-set priority**

- **Given** a case whose priority was previously set by a reviewer
- **When** a reviewer sets the priority again with a new value and reason
- **Then** the priority is updated to the new value, and both changes are preserved in the history

**AC-9: Activity feed notification**

- **Given** a case whose priority was set by a reviewer
- **When** the priority change is recorded
- **Then** the console's activity feed receives a notification stating "Priority set to <priority>", with the employee's name when the service order has one

### Cases that cannot be changed

**AC-10: Running cases cannot be changed**

- **Given** a case whose `status` is `running`
- **When** a reviewer attempts to set its priority
- **Then** the request is refused with a message telling the reviewer to wait until the case pauses

**AC-11: Terminal-state cases cannot be changed**

- **Given** a case whose `status` is `done`, `rejected`, or `failed`
- **When** a reviewer attempts to set its priority
- **Then** the request is refused with a message stating the case's status

**AC-12: Non-existent cases**

- **Given** a case identifier that does not exist
- **When** a reviewer attempts to set its priority
- **Then** the request is refused with a not-found message

### General

**AC-13: Persistence across restarts**

- **Given** a case whose priority was set by a reviewer
- **When** the service restarts
- **Then** the reviewer-set priority and reason are retained

**AC-14: System-managed priority for other cases**

- **Given** a case whose priority no reviewer has set
- **When** the case is processed
- **Then** the priority works exactly as today: the planner sets the priority (`priority` field in `State`), and it is raised one level (per `_RAISED` mapping in `orchestrator_node`) when any flag is raised

## Examples

- **SO-1004** is a termination whose final pay is due today. The planner set `priority` to `medium`. Asha sets it to `high` with reason "Final pay due today". The case displays priority `high`, set by a reviewer, with the reason.
- Later a compliance flag is raised on **SO-1004**. The priority stays `high` and does not change.
- Ben sets a case waiting on the client from `medium` to `low` with reason "Waiting on client for bonus details". The next day he sets it back to `medium` with reason "Client replied". The history shows both changes.
- Asha tries to set a priority of "urgent" and is told the allowed values are `high`, `medium`, and `low`.

## Out of scope

- Changing the priority from the command line
- Changing how the planner decides priority
- Ordering cases or queues by priority
- Notifications (beyond the activity feed entry in AC-9)
