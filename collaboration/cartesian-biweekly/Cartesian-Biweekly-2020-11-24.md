# 2020/11/24 GT4Py bi-weekly  
  
# Action Points  
[ ] @Johann D Will make a GDP for looping  
[ ] @Oli F @Rhea G will propose changes to the quickstart based on fv3 and student experience  
  
**Review of action points of last meeting:**  
  
[ ] @Johann D Will make a GDP for looping  
[x] @Johann D Will update Regions GDP with new implementation  
[ ] @Oli F @Rhea G will propose changes to the quickstart based on fv3 and student experience  
    - Update 10.11.2020: Unfortunately no progress here. Currently all of our attention is on preparing the 3-day DSL workshop next week.  
# Discussion Points  
## Workshop Recap  
  
Materials: https://github.com/VulcanClimateModeling/dsl_workshop  
  
  
![](https://paper-attachments.dropbox.com/s_1E38AE7606C02B120FDA14152ECD1C23AE134BEF94A2B1D00548A1286FE3D952_1606171198744_Unknown-1.png)  
  
![](https://paper-attachments.dropbox.com/s_1E38AE7606C02B120FDA14152ECD1C23AE134BEF94A2B1D00548A1286FE3D952_1606171144329_Unknown.png)  
  
  
  
## Regional Computation  
  
GDP updated: https://github.com/GridTools/gt4py/pull/24 👌 Awaiting review  
  
**PR status:**  
  
1. New interval processing https://github.com/GridTools/gt4py/pull/225 ✅ Merged  
2. Frontend implementation https://github.com/GridTools/gt4py/pull/227 👌 Awaiting review, thoroughly tested   
3. Lower to IIR https://github.com/GridTools/gt4py/pull/228 🙈 Ready for initial review, needs a few more tests  
4. Backend implementations https://github.com/GridTools/gt4py/pull/234 🙈 Ready for initial review, needs a test_suites test, but I can’t figure out how to make validation pass  
  
Get GDP finalized by next Tuesday, then review frontend impl  
  
## Lower-dimensional storages  
  
Current PR: https://github.com/GridTools/gt4py/pull/203  
  
Currently this solves:  
  
- Removes axes of temporary FieldDecls if no dependency in that direction  
- Codegen in supported backends  
  
What we still want to do:  
  
- Fix restriction that temporary fields are not reduced if they are accessed in parallel vertical loops (broadcast issues in numpy backend)  
- Make compatible with “mask” idea in frontend  
- **Look at type hint for axes**  
  
  
## Expose the Iteration Index  
https://github.com/GridTools/gt4py/pull/264  
  
  
[GridTools/gt4py#264](https://github.com/GridTools/gt4py/pull/264)  
  
Todo: check type of eval.i() in GT C++  
  
It seems common, especially in pointwise physics routines, to check if levels are within a range. Example:  
  
https://github.com/VulcanClimateModeling/physics_standalone/blob/9368e4461c7d6391a4a1eb0f77ec150cf87707d7/microph/gfdl_cloud_microphys.F#L4964  
  
https://github.com/VulcanClimateModeling/physics_standalone/blob/9368e4461c7d6391a4a1eb0f77ec150cf87707d7/microph/gfdl_cloud_microphys.F#L4995-5000  
  
## Looping  
  
Three types of loops seem to recur as motifs:  
  
1. Tracer-like loops: do computation for a set of fields: We agreed we would solve this with a `*args` like argument to stencils  
2. Do a set of statements a variable number of times: A simple for loop should work for this (we agreed on compile-time upper bound for now)  
3. Waterfall loops  
  
  
    for k in range(1, nk):  
        if ri[k] < ri_ref:  
            h0 = m[k] * (q[k] - q[k-1])  
            q[k-1] += h0 / w[k-1]  
            q[k]   -= h0 / w[k]  
  
This type of loop is particularly problematic because it accesses an element again after already setting it. Hence I call this a *waterfall loop.*  
  
I have shown with sympy that this can be mapped to double for loop: an outer for loop with an inner for loop that goes from current K→end.  
  
Example syntax that could work for this:  
  
    with interval(1, None):  
        k_ind = index(K)  
        with interval(k_ind, None):  
            ...  
  
Go ahead with GDPs for (2) then (1), delay (3)  
  
## NOAA workshop  
  
Vertical Riemann solver miniapp here:  
  
https://github.com/VulcanClimateModeling/Riem-solver-c-miniapp  
  
  
[+Initial Riem_Solver_C Mini-App Port Notes](https://paper.dropbox.com/doc/Initial-Riem_Solver_C-Mini-App-Port-Notes-GseXErqCMJH140Yv35U1b)   
  
**Main points relevant to GT4Py:**  
  
- SIM1_Solver inlining  
- Temporaries across computations  
- Need to allow empty computations due to compile-time ifs (PR submitted)  
  
  
## Strict vs Non-Strict Typing  
https://github.com/GridTools/gt4py/issues/263  
  
  
[GridTools/gt4py#263](https://github.com/GridTools/gt4py/issues/263)  
  
