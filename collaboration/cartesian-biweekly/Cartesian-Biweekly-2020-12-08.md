# 2020/12/08 GT4Py bi-weekly  
  
# Action Points  
[ ]   
  
**Review of action points of last meeting:**  
  
[ ] @Johann D Will make a GDP for looping  
[x] @Oli F @Rhea G will propose changes to the quickstart based on fv3 and student experience  
[ ] [Regions] Will review GDP by next Tuesday, then review frontend impl  
# Discussion Points  
## Quickstart - @Oli F   
  
PR coming soon with minor edits!  
  
## Issue: Storages + MPI4py - @Eddie D   
  
Two issues:  
  
1. FV3 dycore uses a `Quantity` class that wraps GT4Py `Storage` objects, requiring that the `Storage.data` property returns a `np.ndarray` for CPU storages or `cp.ndarray` for GPU storages.  
https://github.com/GridTools/gt4py/pull/218  
  
    [GridTools/gt4py#218](https://github.com/GridTools/gt4py/pull/218)  
    A problem persists with implicitly defined storages, e.g., the result of an arithmetic operation, `s3 = s1 + s2`, where the `CUDARuntimeError: cudaErrorInvalidValue: invalid argument` error occurs.  
2. GPU to GPU communication via MPI in Python is not working on Piz Daint. Programs running `mpi4py` (master) + `cupy` produce segmentation faults. This functionality is needed for halo exchanges. This is a known issue and a ticket (**[cscs.ch #40784]**) has been opened, and a corresponding one with Cray already exists, but just to raise awareness…  
## Regional Computation - @Johann D   
  
GDP updated: https://github.com/GridTools/gt4py/pull/24 👌 Awaiting review  
  
**PR status:**  
Update: Additional tests were added!  
  
1. New interval processing https://github.com/GridTools/gt4py/pull/225 ✅ Merged  
2. Frontend implementation https://github.com/GridTools/gt4py/pull/227 👌 Awaiting review, thoroughly tested   
3. Lower to IIR https://github.com/GridTools/gt4py/pull/228 🙈 Ready for initial review, needs a few more tests  
4. Backend implementations https://github.com/GridTools/gt4py/pull/234 🙈 Ready for initial review, needs a test_suites test, but I can’t figure out how to make validation pass  
  
**Discussion topic: how axis offset externals should be treated.**  
  
  
https://github.com/VulcanClimateModeling/fv3core/blob/master/fv3core/decorators.py#L104  
  
  
  
## Looping  
  
GDPs were promised, but we did not have time this week to finish them.  
  
Three types of loops seem to recur as motifs:  
  
1. Tracer-like loops: do computation for a set of fields: We agreed we would solve this with a `*args` like argument to stencils  
  
PR ready: https://github.com/GridTools/gt4py/pull/278  
  
  
2. **Do a set of statements a variable number of times: A simple for loop should work for this (we agreed on compile-time upper bound for now)**  
    with computation(PARALLEL), interval(...):  
       for istep in range(0, num_steps[0, 0, 0]):  
           # substep tracer advection  
  
  
3. Waterfall loops  
  
  
    for k in range(1, nk):  
        if ri[k] < ri_ref:  
            h0 = m[k] * (q[k] - q[k-1])  
            q[k-1] += h0 / w[k-1]  
            q[k]   -= h0 / w[k]  
  
This type of loop is particularly problematic because it accesses an element again after already setting it. Hence we call this a *waterfall loop.*  
  
  
## GTC migration  
- GridTools 2 (cpu_ifirst) backend close to finished  
- NumPy in progress (Rico was on vacation)  
- No optimizations implemented yet  
- (Note to @Johann D: I was working on the auto type deduction today as this was a missing feature for finishing the GridTools backend, @Johann D - I’m so sorry I have not been present on this work. I only yesterday got our develop branch working again 😀)  
  
