# 2020/09/15 GT4Py bi-weekly  
  
# Action Points  
[ ] @Tobias W Ensure conditional behavior together with extended reads and writes makes sense for the user in the parallel model  
[ ] @Hannes V Continue refining gtscript parallel model document  
[ ] @Rhea G Will review https://hackmd.io/@k0ziESjjQxS2xHLFwGPOyw/HJLOUL0NP/edit  
[ ] Vulcanites will review and propose an expansion to the Quickstart based on experience from fv3  
[ ] @Mauro B Ask Stefano about priority for Dawn backends and prioritize issue accordingly  
[ ] @Jeremy M Review Duck Storage PR update  
  
**Review of action points of last meeting:**  
  
[ ] @Tobias W, @Oli F Present outcome of HPC4WC student projects (once they are graded) — **Meet Monday 21/9 to review**  
[ ] @Enrique G. P will report on how the Daint testing for GPU backends is working: https://gitlab.com/cscs-ci/gridtools/gt4py-ci/-/pipelines  
[x] @Eddie D Will work on multistage issue with gt4py devs  
[x] @Enrique G. P, @Johann D Will review regions toolchain implementation  
[ ] @Johann D will create a doodle poll to setup the next gtscript workshop (after Sept. 20)  
[ ] @Johann D will add “Change `computation` to `vertical`" to gtscript workshop  
[x] @Enrique G. P will add language to the parallel model about conditions  
[x] @Hannes V will push on finishing parallel model conditional language given example from @Tobias W (disallow pattern only for point-based if?)  
[x] @Enrique G. P will ask Mauro about documentation and report back: https://hackmd.io/@k0ziESjjQxS2xHLFwGPOyw/HJLOUL0NP/edit  
[x] @Rhea G What would useful documentation for gt4py look like?  
# Discussion Points  
## Discuss conditionals5 branch in GridTools/concepts [@Mauro B]  
  
New PR https://github.com/GridTools/concepts/pull/24 with a slightly different approach to the parallel model. Designed on some principles and examples have been worked out to show that performance optimization can be achieved.  
  
Reviewed this document:  
  
https://github.com/GridTools/concepts/blob/conditionals5/GTScript-Parallel-model.md  
  
  
4 design principles for gt4py syntax and semantics  
Discussed conditionals and stage extension  
  
  
## User docs [@Rhea G]  
  
Short and easy to read  
Many examples  
Have docs right there - don’t want to build it after cloning. Use ReadTheDocs  
  
@Oli F - what balance do we want between reference and tutorial  
  
## Review Issue #160 [@Mauro B]  
  
[GridTools/gt4py#160](https://github.com/GridTools/gt4py/issues/160)  
  
  
## Update on duck storage implementation [@Linus G]  
  
Jeremy or another Vulcanite will review.  
  
## Backend Race condition [@Tobias W]  
  
Wait for #169 to be merged. If still an issue, #148 will be prioritized.  
  
