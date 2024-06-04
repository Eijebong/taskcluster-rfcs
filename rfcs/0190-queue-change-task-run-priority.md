# RFC 0190 - Queue change task / task group priority
* Comments: [#0190](https://github.com/taskcluster/taskcluster-rfcs/pull/190)
* Proposed by: @lotas

## Summary

Since we refactored queue internals it became possible to see what tasks are pending for a particular
worker pool / task queue.

This RFC proposes new API method that would allow to change the priority of the existing task run / task group.

## Motivation

There might be several use-cases when some worker pool has a lot of pending tasks and there is a need
to move some more important tasks to the top of the waiting list, or also move down less important ones.

Instead of cancelling some existing tasks we can change task run priorities.

Use-case examples:

* We have a high-priority patch running tasks on Try. We want to bump the priority of all tasks in this task group
* We have a chemspill release or high priority nightly to get out. We want to bump the priority of all tasks in one or more groups
* We are waiting for a specific task in a low resource pool (eg: mac hardware). We want to bump the priority of that specific task
* There are several tasks with the same highest priority and one of them is more important. We want to lower the priority of the less important task

## Details

Queue service will expose new methods:

* `queue.changeTaskPriority(taskId, newPriority)`
* `queue.changeTaskGroupPriority(taskGroupId, newPriority)`

New priority would be stored along the task definition for a given task or all tasks within the task group.

The process to change single task will be as follows:

* new task priority will be stored in the database as `task.priority_override` column
* if task was scheduled already, its priority in the `queue_pending_tasks` table will be updated
* if the tasks run fails and new one is created, it will check if `task.priority_override` is set and use it instead of the original priority

The process to change the whole task group will be as follows:

* all tasks in the task group will be updated to store the new priority in the `task.priority_override` column
* all currently scheduled tasks would be attempted to be updated in the `queue_pending_tasks` table
* all unscheduled tasks and tasks that will be restarted will use the new priority

### New scopes

This will also require introduction of the new scopes (anyOf):

* `queue:change-task-priority-in-queue:<taskQueueId>`
* `queue:change-task-priority:<taskId>`
* `queue:lower-task-priority:<taskId>` to only allow lowering the priority
* `queue:raise-task-priority:<taskId>` to only allow raising the priority

And for task group:

* `queue:change-task-group-priority:<schedulerId>/<taskGroupId>` (for consistency with existing task group scopes)
* `queue:lower-task-group-priority:<taskGroupId>` to only allow lowering the priority
* `queue:raise-task-group-priority:<taskGroupId>` to only allow raising the priority

### Priority calculation

Current order for picking up tasks is based on the task priority and insertion time (FIFO).
This RFC proposes to change the priority only, and leave the insertion time as is.

### Other considerations

To ensure that Chain of Trust validation is not affected, we aim to keep the original task definition
and store the new priority in a separate column.

## Implementation

_pending_