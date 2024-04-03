# RFC 0190 - Queue change task run priority
* Comments: [#0190](https://github.com/taskcluster/taskcluster-rfcs/pull/190)
* Proposed by: @lotas

## Summary

Since we refactored queue internals it became possible to see what tasks are pending for a particular
worker pool / task queue.

This RFC proposes new API method that would allow to change the priority of the existing task run.

## Motivation

There might be several use-cases when some worker pool has a lot of pending tasks and there is a need
to move some more important task to the top of the waiting list, or also move down less important ones.

Instead of cancelling existing tasks we can instead choose to change task run priorities.

## Details

I propose we implement this on a task run level and not the task itself:

* tasks are supposed to be immutable
* changing the priority makes more sense in the context of a specific run, not the whole task

Queue service will expose new method `queue.changeTaskRunPriority(taskId, runId, newPriority)`.

This will try to do the following:

* update `task.runs[runId]` in the database to reflect priority change (possibly extra field to keep the original)
* update `priority` field in the `queue_pending_tasks` table

If any of this fails (for example task is being changed by some other process), this method will also fail.

This will also require introduction of the new scopes (anyOf):

* `queue:change-task-run-priority-in-queue:<taskQueueId>`
* `queue:change-task-run-priority:<taskId>`


Current order for picking up tasks is based on the task priority and insertion time (FIFO).
This RFC proposes to change the priority only, and leave the insertion time as is.


### Other considerations

Chain of Trust validation might be broken if we change too much of the task definition or run.
We need to make sure we don't break it with this change.




## Implementation

_pending_