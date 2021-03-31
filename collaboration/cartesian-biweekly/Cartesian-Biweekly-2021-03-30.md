# 2021/03/30 GT4Py bi-weekly  
  
# To Dos  
## Review To Dos from previous sync  
[ ]  Will ensure CSCS can compile microphysics code — working on it  
# Agenda  
## Problems found with horizontal regions  
- Main problem: extents cannot be generalized to the entire IJ plane for horizontal regions in all cases  
- [+The regions problem](https://paper.dropbox.com/doc/The-regions-problem-qC9ZdCMsRdFvZmVbM73fH)   
- Differences between GridTools execution model and GT4Py parallel model  
    - Tobias presented the summary ([link](https://paper.dropbox.com/doc/qC9ZdCMsRdFvZmVbM73fH))  
    - Enrique: Open discussion with Felix — present in sync meeting with MeteoSwiss?  
    - Multiple GT computations was always intended to be implemented in GT4Py.  
    - These are frontend-independent changes.  
- Open PRs:  
    - #367, multi-stage merging changes  
  
  
## Progress on horizontal regions  
- Show `transportdelp` ([link](https://github.com/VulcanClimateModeling/fv3core/blob/c8ae3061721776bb54a6212ea01a9edc39f79eed/fv3core/stencils/c_sw.py#L76))  
  
  
## Status update: FV3core [@Eddie D]  
- Show performance plot ([link](https://jenkins.ginko.ch/job/fv3core-performance-plots/))  
  
  
## CSCS: Status of sprint  
- @Johann D - do we need higher dimensional fields PR finished?  
    - Yes, needed for for loops.  
    - Enrique has been working on stencil call_run.  
    - Hopefully finished this week.  
- Anton and Felix working on parallel model proposal — perhaps tomorrow  
- Rico working on wrapper generator for GTC toolchain.  
- Linus working on DaCe backend in GT4Py — naive version.  
- Til is developing Python bindings for GHex, also unstructured.  
- Hannes looking at performance, and also unstructured backend.  
- Langwen has been working a bit with Felix on the GTC CUDA backend.  
  
  
## Status update on GT C++ v1 compilation times [@Tobias W]  
- Investigation with Anton  
    - Discrepancies between what students and Anton report  
    - Need for another study?  
- [+Compile time issues](https://paper.dropbox.com/doc/Compile-time-issues-DouZarAKNQ5t93VIR6kXn)   
  
