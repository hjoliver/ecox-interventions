EcoConnect intervention scenario 3
==================================

We need to insert a family of tasks in the past. That can happen if
the previous run succeeded but new info have arrived and we need to run
it again, and it is gone from the pool.

We use the insert task feature in Cylc 7. All tasks in the family go into
waiting state. If the first task does not start automatically (i.e. it's
parents have gone from the pool) we may have to trigger it to start
(Requires to check the prerequisites to find out which one is the first
task or look at the graph)

P1 = "a[-P1] & data => a => b1 => b1 & b3 => b4"  # bn is a family
# want to rerun "b => FAM => d" at cycle 3

Cylc 7 intervention
-------------------
 
1. Insert the family in the past, as waiting
    cylc insert ecox/thr FAM.3
2. Trigger the first task(s) in the family sub-graph
    cylc trigger ecox/thr b1.3

Cylc 8.3 intervention
---------------------

1. Just trigger a new flow at the first task(s) in the sub-graph:
   cylc trigger ecox/thr//3/b1 --flow=new

Comments:
   - the family is a side graph so the new flow naturally comes to stop without
     flowing on to subsequent cycles
    
Cylc 8.4 intervention
---------------------

1. trigger by family name - no need to identify the initial task(s)
   cylc trigger ecox/thr//3/FAM
