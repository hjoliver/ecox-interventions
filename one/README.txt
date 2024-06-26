EcoConnect intervention scenario 1
==================================

R1 = "a => b => c => d"

Task b succeeds, but one of its outputs got corrupted causing task c to fail. 

Requirements:
- rerun task b, and after that c should run again
- we don't want to have to wait for b to finish to manually trigger c

Cylc 7 intervention
-------------------

1. Trigger the succeeded upstream task b
    cylc trigger ecox/one b.1
2. Reset the failed task c to waiting, so that it will run again once b succeeds
    cylc reset ecox/one --state=waiting c.1

Cylc 8 intervention
-------------------

1. trigger task b to start a new flow:
   cylc trigger ecox/one//1/b --flow=new

A new flow can traverse parts of the graph that already ran.

When b(flow=2) succeeds and tries to spawn c (flow=2), it will merge with
the incomplete failed task c(flow=1) in the task pool. c(flows=1,2) will
run automatically (because it belongs to the new flow too). Downstream
tasks continue as the legacy of both flows: d(flows=1,2).

