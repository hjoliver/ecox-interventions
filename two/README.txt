EcoConnect intervention scenario 2
==================================

Similar to the scenario one, but 10 tasks run after an upstream task is finished,
only one of them failed.

R1 = "a => b => c<m> => d"  # m = 1..10

Task b succeeds, but one of its outputs got corrupted causing task c<3> to fail. 

Requirements:
- rerun task b, and after that c<3> should run again
- the other c<m> succeeded, don't run them again
- we don't want to have to wait for b to finish to manually trigger c<3>

Cylc 7 intervention
-------------------

1. Trigger the succeeded upstream task b
    cylc trigger ecox/two b.1
2. Reset the failed task c<3> to waiting
    cylc reset ecox/two --state=waiting c_m3.1

Cylc 8.3 intervention
---------------------

1. trigger task b to start a new flow:
   cylc trigger ecox/two//1/b --flow=new

2. set c<m!=3> to succeeded in flow 2
   for i in 01 02 04 05 06 07 08 09; do 
       cylc set ecox/two//1/c_m$i
    done

Tasks c and d will run as the legacy of both flows: d(flows=1,2).


Cylc 8.4 intervention
---------------------

1. erase the original flow in tasks to be rerun:
   cylc remove ecox/two  //1/b //1/c_m3

2. trigger task b:
   cylc trigger ecox/two//1/b

Tasks c and d will run as flow 1.
