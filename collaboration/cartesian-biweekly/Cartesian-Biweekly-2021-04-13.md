# 2021/04/13 GT4Py bi-weekly  
  
# To Dos  
## Review To Dos from previous sync  
[x]  Will ensure CSCS can compile microphysics code — working on it  
# Agenda  
## Progress on horizontal regions [@Johann D]  
  
Detected via loop in MergeBlocksPass. See https://github.com/jdahm/gt4py/blob/horizontal-regions-gtstencil/src/gt4py/analysis/passes.py#L1028  
  
Example:  
  
    def example(out: Field[int], out2: Field[int]):  
        with computation(FORWARD), interval(...):  
            tmp = 1  
            tmp2 = 2  
            out = tmp2[1, 0, 0]  
            with horizontal(region[I[0], :]):  
                out = tmp[1, 0, 0]  
            out2 = out + tmp2  
  
allocated_fields = {out, out2, tmp}  
  
    def example(out: Field[int], out2: Field[int]):  
        with computation(FORWARD), interval(...):  
            tmp = 1  
            tmp2 = 2  
    # --- sync ---  
            out = tmp2[1, 0, 0]  
    # --- sync ---  
            with horizontal(region[I[0], :]):  
                out = tmp[1, 0, 0]  
            out2 = out + tmp2  
  
allocated_fields = {out, out2, tmp, tmp2}  
  
Re-run algorithm → same as last allocated_fields, so loop exits.  
  
  
## Understanding of corner and edge problem [Felix]  
- Present solution in new parallel model. Does it fix all problems or was the simplified testcase too simple?  
- Present plain single-stencil-GT4Py implementation  
## Status update: FV3core [@Eddie D]  
- Show performance plot ([link](https://jenkins.ginko.ch/job/fv3core-performance-plots/))  
  
  
## Profiling GT4Py [@Oli F]  
  
Had to use `add_profile_info` backend option, nsight systems, and latest CUDA version (11).  
Would like share some insights next meeting.  
  
  
## Small PRs / performance related  
- `gtscript.external_assert` function instead of `assert` keyword  
- `StencilObject` changes  
- Make `AccessIntent` an `IntFlag` and add `READ,` `WRITE`  
- Add `device_sync` backend option  
  
  
## Lagrangian Remapping  
  
Algorithm in initial FV3 mapz GridTools port used a while loop and a read with a variable vertical offset. See: https://github.com/VulcanClimateModeling/GT_FV3/blob/master/fv_mapz/fv3_map_stencils.hpp#L136  
  
Our new plan is to try this route instead of writing to an absolute index.  
  
Simplified frontend example:  
  
    def stencil(k_offset: Field[int, IJ], pe1: Field[float], pe2: Field[float], ...):  
      with computation(FORWARD), interval(...):  
        l = k_offset + 1  
        qsum = pe1[0, 0, l+1] - pe2 + ...  
        while pe1[0, 0, l+1] < pe2[0, 0, 1]:  
          qsum += dp1[0, 0, l] * q[0, 0, l]  
          l = l + 1  
  
Changes to `Ref` nodes:  
  
        Ref         = VarRef(name: str, [index: int])  
                    | FieldRef(name: str, offset: Dict[str, int | Expr])  
                    # Horizontal indices must be ints  
  
