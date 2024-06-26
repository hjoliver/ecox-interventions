EcoConnect intervention scenario 4
==================================

Succeeding a family or a full cycle.

A family of tasks may need to be force to succeeded as the required input data
will never arrived (e.g. Himawari, GSMapnow, PCWeather, NIWAWeather, etc.).

We normally force succeed the upstream family task from other workflow first
(e.g. Broadcasting, Ingestion), these workflow are polling the main workflow.

Then we succeed the family in the main workflow (e.g. Himawari). Forcing a
family to succeed will automatically spawn the same family in the next cycle.

P1 = "poller => FAM"
# with poller never succeeds in cycle point 2

Cylc 7 intervention
-------------------
 
1. Reset FAM.2 from waiting to succeeded, to spawn FAM.3 as waiting
    cylc reset ecox/fou FAM.2
2. Kill and remove the poller that will never succeed
    cylc kill ecox/fou poller.2
    cylc remove ecox/fou poller.2

Cylc 8.3 intervention
---------------------

1. Just kill and remove the poller that will never succeed:
   cylc kill ecox/fou//2/poller
   cylc remove ecox/fou//2/poller

Comments:
   - in Cylc 8 we don't need to pretend that 2/FAM succeeded, because there
     is nothing directly downstream of it that we want to run, and 3/FAM is
	 spawned according to the graph (by 3/poller) not by submission of the
	 previous instance of itself (i.e. not by 2/FAM)    
   - if poller is an xtrigger rather a polling task, then the initial task(s)
     of FAM will be spawned as waiting on the xtrigger. In this case, we
	 would just remove them (cylc remove ecox/fou//2/FAM) to prevent a stall.
   - if poller success is optional, it will not need removing once killed
     (i.e., if "poller? => FAM")
	
