---
author: Isaac Fei
pubDatetime: 2024-07-21T08:36:26.903Z
modDatetime: null
title: "Building a Repeating Task Runner in Python"
slug: building-a-repeating-task-runner-in-python
featured: true
draft: false
tags:
- python
- multithreading
description: How to run a task repeatedly every once in a while in the background with the functionality of resetting the timer to postpone the execution of the current started task.
---

![Building a Repeating Task Runner in Python](/images/building-a-repeating-task-runner-in-python.png)

In this post, I will show how to run a task (Python function) repeatedly every once in a while in the background. 
And you may reset the timer to postpone the execution of the current started task.

> This is somehow like a [hardware watchdog](https://en.wikipedia.org/wiki/Watchdog_timer), which resets the system if the software fails to reset the timer periodically, indicating a potential system hang or crash.


## Complete Code Snippet

I mainly used classes `Thread` and `Event` from the `threading` module to achieve this.


```python
from threading import Thread, Event
from typing import Optional, Callable


class RepeatingTaskRunner:

    def __init__(
        self,
        *,
        interval: float,
        task: Callable,
        task_args: Optional[tuple] = None,
    ) -> None:

        # Time interval between each task (in seconds)
        self._interval = interval

        # The task to run
        self._task = task

        # The arguments of the task
        self._task_args = task_args

        # Events
        self._stop_event = Event()
        self._reset_event = Event()

        # Create the thread
        self._thread = Thread(
            target=self._run,
            daemon=True,
        )

    def start(self) -> None:
        """Starts the runner."""

        # Start the thread
        self._thread.start()

    def stop(self) -> None:
        """Stops the runner. This will join the internal thread.
        Once the runner is stopped, it cannot be started again.
        """

        # Set the stop event
        self._stop_event.set()

    def reset(self) -> None:
        """Resets the timer of the runner. This will postpone the next execution of the task."""

        # Trigger the reset event
        self._reset_event.set()

    def _run(self) -> None:

        # Execute repeatedly until the stop event is set
        while not self._stop_event.is_set():

            # Waiting for at most the amount of time defined by the interval...
            #
            # If the reset event is set at some point within the time interval,
            # immediately unset reset event and go to the next iteration
            # This will prevent the task from being executed, and
            # reset the timer
            #
            # If the reset event is not set at the end of the time interval,
            # execute the task
            if self._reset_event.wait(self._interval):

                # Set the event internal flag to False
                self._reset_event.clear()

                continue

            # Exit the loop if the stop event is set
            if self._stop_event.is_set():
                break

            # Execute the task
            if self._task_args is None:
                # Task without arguments
                self._task()
            else:
                # Unpack the arguments
                self._task(*self._task_args)
```

## General Idea

The `RepeatingTaskRunner` is designed to execute a repeating task in a separate thread. The task is encapsulated within a loop, ensuring it can be executed repeatedly.
The thread is running in daemon mode, meaning that it will be terminated when the main process is terminated.

Starting the runner essentially means starting the thread. The loop's execution is controlled by an event, allowing the runner to be stopped gracefully.

The main challenge lies in executing the task and then waiting for a specified interval before the next execution. This is handled using the `wait` method of the `Event` class, which facilitates the necessary timing and control logic.

## The Core Logic

The core functionality of the `RepeatingTaskRunner` class lies in the `_run` method, which is executed in a separate thread to repeatedly run the given task. Let's break down how this method works:

1. Loop Until Stopped:

```py
while not self._stop_event.is_set():
```

The loop runs continuously until the `_stop_event` is set. This allows the task to be executed repeatedly.

2. Wait for the Interval or Reset Event:

```py
if self._reset_event.wait(self._interval):
    self._reset_event.clear()
    continue
```

The wait method on `_reset_event` blocks for at most the amount of time defined by the interval. If the `reset` method is called *within the time interval*, the `_reset_event` is set, causing wait to return `True` immediately. 

The `_reset_event` is then cleared, and the loop continues without executing the task, 
which *simulates* a reset of the timer.

3. Check for Stop Event:

```py
if self._stop_event.is_set():
    break
```

If the `_stop_event` is set during the wait or at any point in the loop, the loop breaks, stopping the execution of the task.

4. Execute the Task:

```py
if self._task_args is None:
    self._task()
else:
    self._task(*self._task_args)
```

If the reset event is not set within the interval, the task is executed. The task can be run with or without arguments, depending on whether `task_args` is provided.

## Side Notes

At first, in the `stop` method, I called `self._thread.join()` at the end, 
which is what one usually does to stop a thread. 
Well, more precisely, to wait for the thread to finish.

```py
def stop(self) -> None:

    # Set the stop event
    self._stop_event.set()

    # Why not this?
    #
    # Join the thread
    self._thread.join()
```

But then, the runner will not stop immediately when this method is called
since it will wait for the thread to finish and hence wait for the reset event to be set.

```py
if self._reset_event.wait(self._interval):
```

Without joining the thread, when calling `stop`, the thread will still be running at the background.
But it will not block the main program, 
and when it finishes waiting for the reset event, 
the task will not be executed since we have this block of code:

```py
# Exit the loop if the stop event is set
if self._stop_event.is_set():
    break
```




## Usage Example

Consider the following example, we want to print `"Hello, World!"` every 1 second.

```py
import time

# Create the runner
runner = RepeatingTaskRunner(
    interval=1.0,
    task=lambda: print("Hello, World!"),
)

# Start the runner
runner.start()

# Wait for 0.5 seconds
time.sleep(0.5)

# Reset the runner to postpone the current execution
runner.reset()

# Keep running for 2.5 seconds
time.sleep(2.5)

# Stop the runner
runner.stop()
```

The code snippet will take 3 seconds, and
the `"Hello, World!"` string will be printed 2 times.
This is because `reset` is called after 0.5 seconds since the start of the runner, the first `"Hello, World!"` will be printed at 1.5 seconds.
