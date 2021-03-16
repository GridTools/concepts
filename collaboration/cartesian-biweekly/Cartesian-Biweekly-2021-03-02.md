# 2021/03/02 GT4Py bi-weekly  
  
# To dos  
[ ] @Tobias W Will ensure CSCS can compile microphysics code  
  
  
# Agenda  
## FV3 transition to GTC toolchain [Johann]  
  
**Basic requirements**  
  
- Regions - requires extent analysis, delayed  
    - New implementation in classic toolchain  
    - 0% in GTC  
- Lower dimensional storages (https://github.com/GridTools/gt4py/pull/327)  
    - PR ready, to be reviewed by Hannes  
- Numpy and/or plain Python backend  
    - Uses extent analysis - which is currently being implemented into GTC  
    - All other tests are passing, modulo lower/higher dimensional fields  
- Regions and lower dimensional storages in numpy backend  
    - 0%  
  
**Timeline:**  
*Through April we still plan to use the “classic” toolchain, but hope to build up feature parity in the meantime and switch at that time*  
******  
- Merge basic requirements  
- Contribute representative stencil(s) to begin tracking GTC performance at CSCS  
- Create nightly fv3core CI plan utilizing GTC backends  
- Use the dycore tests to address specific issues  
  
**Open questions:**  
  
- Is `make_stage_with_extent` still needed? https://github.com/GridTools/gt4py/compare/master...eddie-c-davis:gt_stage_extents  
    - a2b_ord4 has since been refactored, so this might not be necessary any more  
    - **Limitation on re-using temporaries still exists in GT v2 - ask Hannes**  
- Has the enforcement / interpretation of the parallel model changed in GTC?  
    - No guarantee that the classic toolchain was implementing the parallel model correctly. New ones are more correct!  
    - Felix: found that in gtbench classic toolchain did some reordering that GTC does not  
  
**Update on horizontal regions PR**  
  
- Fixed issues in demotion pass - will spin off as separate PR  
    - This now properly enforces the rules in the code:  
        A field can be demoted when it is a temporary field that:  
        1. is never used with an offset  
        2. is never read before assigned to in any stage  
- Fixed issues in block merging - will spin off as separate PR  
    - Uses `networkx` to determine order of reads and writes instead of using accessors, which lack this information. This allows us to be more correct about when to allow merging  
- Numpy backend headache  
    - `numpy.where` evaluates argument everywhere, even at places where it will read out of bounds on the array  
- Merge GDP-2 → Ok!  
  
*Related issue: block merging pass does not distinguish properly between read and write offsets in vertical*  
  
  
## Performance plot of dycore [Tobias]  
  
CI tests are now tracking performance of the main loop of the dycore.  
  
![](https://paper-attachments.dropbox.com/s_9514EBD9E0DA849004C29E214C268D69257CCC4AE70137363031D4D24ED6E3E5_1614697641829_history_per_timestep.png)  
  
## Roadmap & projects [Oli]  
- Roadmap  
    - April → try to get “single node” performance of FV3 as fast as possible on Piz Daint  
    - October → scale to 1-3 km  
- Several upcoming projects  
    - Langwen Huang: ETH CSE MS thesis on asynchronous execution on GPU  
    - Safira Piasko: ETH CSE term paper on land-surface scheme porting  
    - Andrew Pauling: UW internship on radiation scheme porting (possibly with Robert Pincus)  
## Show main learnings from Mikael’s work [Tobias]  
- Potential for optimization: fields that are completely reset in later stages or multistages be renamed so that they can be demoted.  
- Potential for optimization: fusing computations  
## Control flow in stencils [Tobias]  
- Unrolling  
![](https://paper-attachments.dropbox.com/s_9514EBD9E0DA849004C29E214C268D69257CCC4AE70137363031D4D24ED6E3E5_1614698965806_Screenshot+from+2021-02-26+18-19-06.png)  
  
- Start to more control flow between stencils: how much of that do we want?  
  
  
## Absolute indexing [Tobias]  
- Issue: Control-Flow outside the stencil slices Fields to 2d and passes them in as arguments  
    - performance implication as this happens for every k-level  
![](https://paper-attachments.dropbox.com/s_9514EBD9E0DA849004C29E214C268D69257CCC4AE70137363031D4D24ED6E3E5_1614699142812_Screenshot+from+2021-03-02+16-31-56.png)  
  
  
  
## C++ compile time [Tobias]  
  
Status update of investigation of Anton & Tobias  
  
- Right now we have `O(300)` stencils in the dycore  
- Compilation takes at least 10sec on daint on compute nodes  
- To get CI to work, we are hard-copying in the `.gt_cache` folder with the compiled files to make sure we only need to compile new things. This reduces our time to ~2min  
- Anton’s findings:  
    -  number of instantiations should be linear against the number of fields and also almost linear against number of stencils -- `O(fields_no * stencils_no)`.  
  
  
