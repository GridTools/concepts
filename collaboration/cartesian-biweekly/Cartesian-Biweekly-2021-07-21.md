# 2021/07/21 GT4Py bi-weekly  
  
# To Dos  
[ ]   
## Review To Dos from previous sync  
  
  
  
# Agenda  
  
  
# End of Cycle 3 - Next steps [@Oli F ]  
- Tal on vacation this week  
- We want a meeting to look at outcome   
- Basic structure has no strict requirement for all to all communication but for us it might make sense  
- This cycle we wrap up today and will report back to broader Vulcan audience internally  
  
  
  
## While Loops [@Eddie D ]  
  
**Status**  
  
- https://github.com/GridTools/gt4py/pull/426  
- https://github.com/GridTools/gt4py/pull/422  
- (cu-ir and gtc_cpu)  
  
**Discussion**  
  
- Loops are reconsidered  
- implementation of a new construct (eg solver) where this is encapsulated  
- since we agree that for-loops are acceptable we can transform while to for for the one stencil using this  
- There was a Vulcan internal decision taken that while loops might not be consistent with the execution model.  
  
**Outcome**  
  
- Have a discussion how for and while are different and how they fit the model  
- If we want to push this further we would need to shape it up for the next cycle  
  
  
  
## Indirect Addressing in the vertical [@Eddie D ]  
  
**Status**  
  
- https://github.com/GridTools/gt4py/pull/417  
- (cu-ir and gtc_cpu)  
  
**Discussion**  
  
- not in numpyIR yet  
- need a review  
    - Ping Rico  
- possible to do this in the next few days  
  
**Outcome**  
  
  
  
## DaCe integration [@Linus]  
  
**Status**  
  
![](https://paper-attachments.dropbox.com/s_512CC095E991CF440C159F46C2752ED9DD1E11B82D9E80A4C3B418D5AF105B53_1626745460972_Screenshot+from+2021-07-20+03-44-00.png)  
  
  
**Discussion**  
  
- project is “done” as we have an internal representation  
- any function that calls a stencil needs its own decorator  
- there are some frontend things that we need to workaround that we know are possible  
- running it from scratch takes O(45) min  
- No optimization has been done yet  
- `compute_path` can be toggled on and off via a config → no need to remove it when not using the dace backend  
- Next steps moving forward:  
    - SPCL is interested in doing a full model optimization  
    - Get rid of the workaround we can in the front-end  
    - A lot of the concepts should  be applicable to the full dycore  
    - Should we aim for a smaller scale for our performance / validation? Looking only at `c_sw` for example?  
- Branches:  
    - gt4py: `gronerl/for-fv3acoustic-gtc`  
    - dace: `linus-fixes-10`  
    - fv3core: `gronerl/for-dace`  
## Numpy backend update - [@Rico]  
  
**Status**  
  
- https://github.com/GridTools/gt4py/pull/300  
  
**Issues**  
  
- PR merged   
- everything but vertical offsets are working in FV3core  
  
  
## Functional model [@Hannes V]  
  
**Status**  
  
- Goal was finalizing a design   
- start with a prototype implementation  
  
**Discussion**  
  
- we want to work with 2 models:  
    - local view / iterator view:  
        - lower level (closer to c++ backend)  
    - field view:  
        - higher level (closer to numpy backend)  
        - every operation working on fields  
    -   
- focus on iterator view for this cycle  
    - design finished  
    - embedded execution of c++ from python  
    - generation on an IR from that  
      
- ongoing discussion of how a front-end would look  
- interest in field-view as it could interact with current codes/front-ends  
- Natural point in time to look at performance?  
    - design was done with performance in mind  
    - Anton wants to work on a cuda backend this cycle, so it should be upcoming  
  
**Next steps**  
  
- prototype not connected to c++ yet, seems like a natural next step  
- shaping the front-end further  
  
  
# Broader discussions  
  
  
## Unblocked radiation scheme in FV3Core 🔥 - [@Oli F ]  
- two missing features: higher dim fields indirect addressing and for loop over these dimensions  
    - now both are in and we are unblocked  
- radiation as the last scheme for building a GSRM  
- radiation also uses pre-computed coefficients / lookup tables. Unclear if we need them  
  
  
## Compile-Times in GTC - [@Eddie D ]  
- GTC takes significantly longer to build and generate code  
- We would like to start tasks to speed this up  
- (large stencils take up to 6x longer though python [definition function through compilation])  
- split into three components: DaCe, c++ compilation and python exeuction  
- larger stencils take O(min), full dycore takes hours from scratch  
- maybe sync up with @Enrique G. P  for idea-dump?  
  
  
## Integration of “fast” stencils into GT4Py  - [@Tobias W ]  
- how should the API look?  
    - provide origin and domain to the decorator?  
    - have a flag that returns the fast stencil object?  
      
- Who’s responsibility is it to provide this code?  
    - is this fv3core specific enough to just be in an fv3 repo?  
    - very similar to what Till and Christian are using in fvm  
- We would like to see this as a functionality of GT4Py (gives you an option to benchmark smaller stencils easier)  
- interoperate with dace work that provides layouts?  
https://github.com/VulcanClimateModeling/fv3core/blob/0187eea776b59fef47f30a51c55ab4a016855158/fv3core/decorators.py#L79  
  
  
