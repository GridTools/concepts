# 2020/10/27 GT4Py bi-weekly  
  
# Action Points  
[ ] Vulcan - does only removing sync in the stencil wrapper solve the problem.  
[ ] @Johann D Will make a GDP for looping  
[ ] @Eddie D Will request new reviews for PRs reviewed by Linus (who is away ATM)  
[ ] @Johann D Will make region temporary sizing change to regions implementation  
[ ] @Oli F @Rhea G will propose changes to the quickstart based on fv3 and student experience  
  
**Review of action points of last meeting:**  
  
[x] @Jeremy M - review last version of the Duck Storage GDP  
[ ] @Oli F ****@Rhea G **will propose changes to the quickstart based on fv3 and student experience**  
[x] @Johann D will make an issue on GT4Py for loops  
[x] @Johann D @Enrique G. P Will review and merge compile-time asserts PR  
[x] @Johann D @Enrique G. P Will review and merge regions GDP and implementation PRs  
# Discussion Points  
## GFDL Workshop  
  
3 day workshop describing gt4py  
  
- **days 1-2:** overview of gt4py features with hands-on exercises  
- **day 3:** how to port a Fortran weather model code using Serialbox, our testing infrastructure, etc.  
  
  
## Vulcan Update  
  
Python wrapper fv3gfs validates using the gt4py-based dycore!  
  
Current performance:  
  
- Fortran code < 1 minute  
- GT4Py code ~2 days  
  
Compiling stencils takes a long time  
  
- Backend-specific option to pass compiler flags, including `debug_mode`  
- Debug-mode only works for CPU backends  
  
  
## Asynchronous stencil launches  
  
`nowait=True` or `stream=something` stencil launch keywords would be a great to amortize overhead, which can significantly limit performance especially for small kernels.  
  
There is a storage sync at the end of stencils hardcoded into gt4py. The storage GDP may change this so sync is optional. This is part of the solution. Then we need to change the wrapper to remove the sync.  
  
Also need to release GIL.  
  
**Feedback: would like to have this happen, but there is still low-hanging fruit in optimizing stencil call overhead itself.**  
  
## Looping  
  
Discussed the `delnflux` example from fv3core. Will move forward and open an issue to allow compile-time loop bound unrolling in the frontend.  
  
Will create a small GDP for this.  
  
## Lower dimensional fields  
  
Updated pass based on feedback from Linus.  
We are verifying that the code generation is still correct.  
  
## Update on duck storage implementation  
  
Implementation is very close. Wanted to open a draft PR before Linus left, but seems to not be open yet.  
  
## Regions and temporary field sizing  
  
GDP has been edited and ready for review by @Enrique G. P  
  
Temporaries in regions become the wrong sizes when skipping extent analysis on these.  
  
Need to specify extents relative to VarRef splitters. This will require changes to passes and code generation.  
  
## Eve update  
  
Structured grids: understanding and moving passes to GTC from `gt4py.ir.passes`  
  
Workshop tomorrow demonstrating features of Eve.  
Plan implementation over the next two weeks, then kick off at next meeting.  
  
