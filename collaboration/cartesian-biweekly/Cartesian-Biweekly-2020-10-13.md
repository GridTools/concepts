# 2020/10/13 GT4Py bi-weekly  
  
# Action Points  
[x] @Jeremy M - review last version of the Duck Storage GDP  
[ ] @Oli F @Rhea G will propose changes to the quickstart based on fv3 and student experience  
[ ] @Johann D will make an issue on GT4Py for loops  
[x] @Johann D @Enrique G. P Will review and merge compile-time asserts PR  
[x] @Johann D @Enrique G. P Will review and merge regions GDP and implementation PRs  
  
**Review of action points of last meeting:**  
  
[x] @Tobias W Ensure conditional behavior together with extended reads and writes makes sense for the user in the parallel model  
[x] @Hannes V Continue refining gtscript parallel model document  
[x] @Rhea G Will review https://hackmd.io/@k0ziESjjQxS2xHLFwGPOyw/HJLOUL0NP/edit  
[ ] ~~Vulcanites will review and propose an expansion to the Quickstart based on experience from fv3~~ ~~**[We will meet before Tuesday to resolve]**~~  
~~~~[x] @Mauro B Ask Stefano about priority for Dawn backends and prioritize issue accordingly  
[x] @Jeremy M Review Duck Storage PR update  
# Discussion Points  
## Update on discussion in GridTools/concepts [CSCS]  
  
Recent changes:  
  
- [Off-center writes](https://github.com/GridTools/concepts/pull/34)  
- [Conditionals 6: NumPy assignments with Vulcan conditionals](https://github.com/GridTools/concepts/pull/31)  
  
Discussion question: Is the frontend parallel model stable now?  
  
- Definition of the conditionals is stable  
- Off-center writes is not merged yet (but should be merged soon) — Vulcan can give feedback here  
  
Todo:  
  
- Functions  
    - Revise the current implementation  
- Extended computation domain -- how this is computed?  
- **Nonlocals? Discuss in the issue and PR**  
  
  
## GTScript Workshop Review [@Enrique GP]  
https://hackmd.io/9MHp3vuPRVGy1Zf7VILBvw  
  
  
There were open issues at the end of the workshop that we said would be discussed via GDPs.  
  
Could add a follow-up workshop for a few open issues, including the change to vertical/horizontal  
  
## Vulcan “Fork” [@Johann D]  
https://github.com/vulcanclimatemodeling/gt4py  
  
- `branches.cfg`  
- `make_develop.py`  
  
Not ideal workflow, but able to move quickly. Is there a better solution?  
  
  
## Lower Dimensional Fields  
  
We are creating a typing system right now for fv3. Would like to use notation like `gtscript.Field[float, IJK]` if this is supported.  
  
- How does mask work?  
- Can we currently create fields of lower dimension?  
    def stencil(field_in: gtscript.Field[float, IJ])  
  
Typing support in the frontend is already implemented  
Storage creation with lower dimensional fields is not yet complete - may not work  
  
  
## Update on duck storage implementation [@Linus G, @Jeremy M ]  
  
Implementation is advancing!  
  
  
## Eve Preview [@Hannes V, @Rico H]  
  
Will allow using eve pipeline in a couple weeks  
Goal: IR should be bijection with code generation  
  
  
## Compile-Time Assertions [@Johann D]  
  
Should move forward with this.  
  
  
## Loops in GT4Py [@Rhea G]  
  
Discussed the `delnflux` example from fv3core. Will move forward and open an issue to allow compile-time loop bound unrolling in the frontend.  
  
  
