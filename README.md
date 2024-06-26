# Ecox Cylc Intervention Scenarios

Cylc 7 vs Cylc 8 comparison.

--------

## Background: flows

A flow is a single self-consistent run through the graph, or part of the graph.
Cylc 8 can support multiple flows at the same time.

If you trigger a new flow, it can automatically re-traverse parts of the graph
that already ran in a previous flow.

If you retrigger a task as part of a previous flow (e.g. flow 1, the original
flow) the task itself will run, but the flow will not continue because Cylc
knows that the downstream tasks already ran in that flow.

Flow numbers are not yet exposed in the UI, but they do appear in the scheduler
log, if not just the original flow.


### Cylc 8.3 (now)

Re-running parts of the graph requires starting new flows.

### Cylc 8.4 (coming soon)

Re-running parts of the graph won't require explicitly starting a new flow,
because `cylc remove` will be extended "beyond the n=0 task pool" to erase
the previous flow - allowing a re-flow.
 
----------

## Scenario 1: rerun all tasks from upstream of a failed task

```ini
"a => b => c => d"
```
Task `b` succeeds, but one of its outputs was corrupted causing `c` to fail. 

We want to rerun `b`, then `c` without having to wait to trigger `c` after
`b` succeeds.

Demo example:
```ini
# Cylc 8
[scheduling]
    [[graph]]
        R1 = "a => b => c => d"
[runtime]
    [[a, b, d]]
    [[c]]
        script = """
            if (( CYLC_TASK_SUBMIT_NUMBER == 1 )); then
                false
            fi
        """
```

### Cylc 7

```sh
# 1. retrigger the upstream task b
$ cylc trigger ecox/one b.1

# 2. reset the failed c to waiting, to run again once b succeeds
$ cylc reset ecox/one --state=waiting c.1
```

### Cylc 8

```sh
# 1. Retrigger b to start a new flow
$ cylc trigger ecox/one//1/b --flow=new
```

#### Explanation

There's no need to "reset tasks to waiting" in Cylc 8. All future tasks (with respect
to any flow that has not run them yet) are effectively "waiting" to be spawned by
completion of the upstream outputs that they depend on.

When `b(flow=2)` succeeds and tries to spawn `c (flow=2)` it will encounter
`c:failed(flow=1)` already in the task pool. The flows will merge and 
`c(flows=1,2)` will run automatically (because it belongs to the new flow too).

Downstream tasks continue as the legacy of both flows: `d(flows=1,2)`.

------------------

## Scenario 2: rerun some tasks from upstream of a failed task

Similar to scenario 1, but 10 tasks run after an upstream task succeeds, and
only one of them fails.
```ini
# m = 1..10, with c<3> fails
"a => b => c<m> => d"
```

We want to rerun `b`, and then `c<3>`, without having to wait to trigger
`c<3>` after `b` succeeds.


```ini
# Cylc 8
[task parameters]
    m = 1..10
[scheduling]
    [[graph]]
        R1 = "a => b => c<m> => d"
[runtime]
    [[root]]
        pre-script = sleep 5
    [[a, b, d]]
    [[c<m>]]
    [[c<m=3>]]
        script = """
            if (( CYLC_TASK_SUBMIT_NUMBER == 1 )); then
                false
            fi
        """
```

### Cylc 7

```sh
# 1. Trigger the succeeded upstream task b
$ cylc trigger ecox/two b.1

# 2. Reset the failed task c<3> to waiting
$ cylc reset ecox/two --state=waiting c_m3.1
```

### Cylc 8.3 (now)

```sh
# 1. trigger task b to start a new flow
$ cylc trigger ecox/two//1/b --flow=new

# 2. set c<m!=3> to succeeded in flow 2
$ for i in 01 02 04 05 06 07 08 09; do 
     cylc set ecox/two//1/c_m$i
  done
```

#### Explanation

Setting a final task output (e.g. succeeded) in Cylc 8 changes the
task state and spawns activity downstream just as if the output had
been completed naturally.

If a task already has a final status, it won't run again in the same flow.

Tasks `c` and `d` will run as the legacy of both flows: `d(flows=1,2)`.

This is the only one of the four scenarios that requires more work in
Cylc 8: we have to manually set to succeeded, all the tasks that we don't
want to run again in the new flow. However, it'll get a lot easier at 8.4:

### Cylc 8.4 (coming soon)

```sh
# 1. erase the original flow, in tasks to be rerun
$ cylc remove ecox/two  //1/b //1/c_m3

# 2. trigger task b
$ cylc trigger ecox/two//1/b
```

#### Explanation

Erasing a flow (from 8.4) allows that part of the graph to run again in the
same flow, so all we have to do is erase flow 1 for the two tasks that we want
to rerun.

--------------------

## Scenario 3: rerun a past family of tasks

We need to re-run a past family of tasks after they and their upstream
parents already succeeded and left the task pool.

```ini
"""a[-P1] & data => a
   a => b1 => b1 & b3 => b4"""  # bn is a family
# we want to rerun "b => FAM => d" at cycle point 3, say
```

Demo example:
```ini
# Cylc 8 
[scheduling]
    cycling mode = integer
    [[graph]]
        P1 = "a[-P1] & data => a => b1 => b2 & b3 => b4"
[runtime]
    [[FAM]]
    [[data]]
    [[b1, b2, b3, b4]]
        inherit = FAM
    [[a]]
        script = """
           # cause a runahead stall at cycle 5 just to stop the action
            if (( CYLC_TASK_CYCLE_POINT == 5 )); then
                false
            fi
        """
```

### Cylc 7

```sh
# 1. Insert the family in the past, as waiting
$ cylc insert ecox/thr FAM.3

# 2. Trigger the first task(s) in the family sub-graph
$ cylc trigger ecox/thr b1.3
```

### Cylc 8.3 (now)

```sh
# 1. Trigger a new flow at the first task(s) in the sub-graph:
$ cylc trigger ecox/thr//3/b1 --flow=new
````

#### Explanation

The family forms a side graph so the new flow naturally comes to a stop
without flowing on to subsequent cycles.
    

### Cylc 8.4 (coming soon)

```sh
# 1. trigger by family name - no need to identify the initial task(s)
$ cylc trigger ecox/thr//3/FAM [--flow=new or reflow]
```

(This depends on a small development proposal that hasn't been discussed
and agreed yet, but I'm pretty sure it will go through).

---------------

## Scenario 4: skip over a family or cycle

Skip a family or a full cycle.

A family of tasks may need to be force to succeeded as the required input data
will never arrived (e.g. Himawari, GSMapnow, PCWeather, NIWAWeather, etc.).

We normally force succeed the upstream family task from other workflow first
(e.g. Broadcasting, Ingestion), these workflow are polling the main workflow.

Then we succeed the family in the main workflow (e.g. Himawari). Forcing a
family to succeed will automatically spawn the same family in the next cycle.

```ini
P1 = "poller => FAM"
# with poller never succeeds in cycle point 2
```

Demo example:
```ini
# Cylc 8
[scheduling]
    cycling mode = integer
    [[graph]]
        P1 = "poller => FAM"
[runtime]
    [[root]]
        pre-script = sleep 5
    [[FAM]]
    [[b1, b2, b3, b4]]
        inherit = FAM
    [[poller]]
        script = """
           # never succeed at cycle point 2
            if (( CYLC_TASK_CYCLE_POINT == 2 )); then
                sleep 3600
                false
            fi
        """
```

### Cylc 7
 
```sh
# 1. Reset FAM.2 from waiting to succeeded, to spawn FAM.3 as waiting
$ cylc reset ecox/fou FAM.2

# 2. Kill and remove the poller that will never succeed
$ cylc kill ecox/fou poller.2
$ cylc remove ecox/fou poller.2
```

### Cylc 8.3 (now)

```ini
# 1. Just kill and remove the poller task that will never succeed:
$ cylc kill ecox/fou//2/poller
$ cylc remove ecox/fou//2/poller
```

#### Explanation

In Cylc 8 we don't need to pretend that `2/FAM` succeeded, because there
is nothing directly downstream of it that we want to run, and `3/FAM` is
spawned according to the graph (i.e., by `3/poller`) not (as in Cylc 7) on
submission of the previous instance of itself (i.e. not by `2/FAM`).    

If `poller` success is optional (i.e. `poller? => FAM`) it will not need to
be removed once killed.

If `poller` is an xtrigger rather a polling task, then the initial task(s)
of `FAM` will be spawned as waiting on the xtrigger. In this case, we
would just remove them (`cylc remove ecox/fou//2/FAM`) to prevent a stall.


### Cylc 8.4 (coming soon)

A new "skip mode" will allow easily skipping any part of the graph by (in effect)
running through in simulation mode.
