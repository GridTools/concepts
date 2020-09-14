# Action Points  
[ ] @Tobias W, @Oli F Present outcome of HPC4WC student projects (once they are graded)  
[ ] @Enrique G. P will report on how the Daint testing for GPU backends is working  
[ ] @Eddie D Will work on multistage issue with gt4py devs  
[ ] @Enrique G. P, @Johann D Will review regions toolchain implementation  
[ ] @Johann D will create a doodle poll to setup the next gtscript workshop (after Sept. 20)  
[ ] @Johann D will add “Change `computation` to `vertical`" to gtscript workshop  
[ ] @Enrique G. P will add language to the parallel model about conditions  
[ ] @Hannes will push on finishing parallel model conditional language given example from @Tobias W (disallow pattern only for point-based if?)  
[ ] @Enrique G. P will ask Mauro about documentation and report back  
[ ] @Rhea G What would useful documentation for gt4py look like?  
  
**Review of action points of last meeting:**  
  
[x] @Johann D Will update regions GDP language to match implementation  
[x] @Rhea G + @Oliver E read through https://github.com/GridTools/concepts/wiki/GTScript-Parallel-model and give feedback from implementation-land  
  
  
[x] Submit items for mini-workshop in this document:  
    - [https://docs.google.com/document/d/1hFQ0TfO7fAiz7Xh-LE3RQucesI_k0tuCGku6pmz9qnY/edit](https://docs.google.com/document/d/1hFQ0TfO7fAiz7Xh-LE3RQucesI_k0tuCGku6pmz9qnY/edit) (outdated)  
    - https://hackmd.io/@egparedes/Bkf7qxibv  
[ ] @Tobias W, @Oli F Present outcome of HPC4WC student projects (once they are graded) — carried over to this time  
# Discussion Points  
## Testing of gt4py [@Oli F]  
  
Currently only internal backend are tested on CPU on github.com/gridtools/gt4py.  
VCM would like to extend testing to GPU and Piz Daint.  
How to proceed?  
  
Proof of concept working   
  
  
## Closure: make stage with extents issue [@Eddie D]  
https://github.com/GridTools/gt4py/issues/143  
  
  
Meant to resolve [GridTools/gt4py#143](https://github.com/GridTools/gt4py/issues/143)  
  
We need to make separate multistages for fields written back to themselves with a horizontal extent. This check used to exist in an older version of the pipeline, but was since removed. We need to put back a fix to not merge multistages when this pattern exists.  
  
Assignee: @Eddie D   
  
  
## Reduced field dimensions [@Eddie D]  
  
Reduce temporary field dimensions if not used between multistages and k extent is 0.  
Changed axes in fielddecl to IJ  
Workaround: Changing shape to 1 in these dimensions (thus index to 0)  
  
  
  
  
## Regions [@Johann D]  
  
Update: python backends working, need gridtools support  
Initial review — @Enrique G. P   
Settled on using `with horizontal(region[:, js:je])`  
  
We’re removing the PARALLEL keyword in `computation(...)`!  
  
  
## Stencil Workshop Follow-up [@Enrique]  
  
@Enrique G. P cleaned up the workshop document  
Tasks will be created and planned there  
  
Point to add: Change `computation` to `vertical`  
  
  
## Discuss: temporaries in conditionals [@Tobias W]  
  
Specifically `if` conditions make us split computations into multiple stencils  
Problem: run-time conditionals   
  
  
## Parallel model feedback [@Rhea G, @Oliver E]  
  
See [+GT4Py Parallel Model thoughts](https://paper.dropbox.com/doc/GT4Py-Parallel-Model-thoughts-xjUHfsc2SV0D0nqCEnozV) .  
  
  
## Discuss: definition of conditionals in the parallel model [@Enrique G. P @Hannes]  
  
2 way of defining this: whole block or each statement is wrapped in a parallel for loop  
  
  
## Documentation [@Oli F]  
  
Where are new features (e.g. regions) documented? GDP? Should we plan a Hackathon where we aim for more documentation? Priority? Format?  
  
  
