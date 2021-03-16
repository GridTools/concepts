# 2020/11/10 GT4Py bi-weekly  
  
# Action Points  
[ ] @Johann D Will make a GDP for looping  
[x] @Johann D Will update Regions GDP with new implementation  
[ ] @Oli F @Rhea G will propose changes to the quickstart based on fv3 and student experience  
    - Update 10.11.2020: Unfortunately no progress here. Currently all of our attention is on preparing the 3-day DSL workshop next week.  
  
**Review of action points of last meeting:**  
  
[x] Vulcan - does only removing sync in the stencil wrapper solve the problem?  
    Some tests passed, others did not (where numpy and cupy code linked stencils). Is a viable approach.  
[x] @Eddie D Will request new reviews for PRs reviewed by Linus (who is away ATM)  
[x] @Johann D Will make region temporary sizing change to regions implementation  
# Discussion Points  
## Vulcan Update  
  
Workshop been the Vulcan focus, so not much has happened in the last week, nor will happen in the next week.  
  
  
## Asynchronous stencil launches follow-up  
  
Requires first removing or adding syncs for numpy and cupy code between stencil calls.  
  
  
## Looping  
  
Expanded parameters - @Eddie D will write a GDP for this  
  
    def stencil(field_in: Field, fields: Tuple[Field, ...], *tracers: Field):  
      pass  
      
    gtscript.stencil(stencil, )  
  
Local iterations - @Johann D will write a GDP for this (only compile-time int range for now)  
  
    def stencil(field_in: Field, fields: Tuple[Field, ...], *tracers: Field):  
      from __externals__ import nord  
      with computation(PARALLEL), interval(...):  
        for n in range(nord):  
          # do something in a loop  
          pass  
  
  
## Lower dimensional fields  
  
Putting in the dycore. PR is here: https://github.com/GridTools/gt4py/pull/203  
  
  
## Update on duck storage implementation  
  
GDP is finalized. Implementation will finish up in the next few weeks.  
  
  
## Horizontal Restricted Computation  
  
Walkthrough series of PRs.  
Demo.  
Need to update GDP.  
  
  
## Eve update  
  
Kick-off meeting on Monday.  
Goal: GTIR to code-generation for numpy and GT C++ backends (no CUDA)  
  
Notebook is here:  
  
https://hackmd.io/Un720B_SSbazsk4-FYkBPQ  
  
  
