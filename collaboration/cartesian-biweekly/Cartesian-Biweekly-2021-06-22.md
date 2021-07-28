# 2021/06/22 GT4Py bi-weekly  
  
# To Dos  
[ ]   
## Review To Dos from previous sync  
[x] [@Johann D ] Organize a time to work on the variable vertical offsets and while/for loops  
  
  
# Agenda  
## Distributed Caching Strategy Demo - [@Eddie D]  
  
Current approach is generic - tells application to give gt4py groups of processors  
Each processor group has a generator rank, and waits for this rank to generate the stencil  
  
Discussed a different approach with a global orchestrator  
  
  
## Numpy backend update - [Rico]  
  
**Status**  
Working on existing features  
Once features work, needs cleanup and master merge  
  
**Issues**  
Horizontal regions introduces a new iteration space  
  
  
## Horizontal regions update - [@Johann D]  
  
**Status**  
Frontend, gtir, and oir ready  
Currently working on extent analysis in OIR so that some of this can be shared between the GTC backends  
  
**Issues**  
Does not fit well into GTC model - access extents are inseparable from sizing extents  
Backends each do their own transformations and extent analysis, so infrastructure is somewhat duplicated (but for good reasons)  
  
  
## Nested control-flow - [All]  
  
**While loops**  
Merged into legacy backends  
PRs exist for GTC  
****  
**For loops**  
Implementation exists for legacy backends  
Started in GTC backends  
  
  
    for va in range(1, 9):  
      field += va  
  
**Update**  
Shows the need for foreign concepts in the IR  
Ideas for reductions  
  
  
## DaCe-related work update - [@Tobias W]  
  
**Status**  
  
- Standalone application for the acoustic time-step working with GTC master  
- Still in the FV3Core regression suite so verification can be done  
  
**Issues**  
  
- Next steps unclear?  
  
  
# Broader discussions  
## What is a stencil? What should be allowed inside a `gtscript.stencil`  
  
There seem to be opposing viewpoints on this  
  
1. (As a first step) everything that is needed to port a weather model from Fortran  
    1. Includes some control flow that could be done differently  
2. Focused DSL from start  
    1. Downsides: difficult-to-sell solutions to partners/users  
  
Agreement? If there are patterns that are not included, have solutions for these. Do not try to design for all.  
  
Are we supporting variable neighborhoods? Maybe?  
Are we supporting variable weights? Yes.  
  
  
## Who is our target user?  
  
Ideas:  
  
- Domain scientists porting codes from Fortran  
    - Do they want usability first, or performance, or both?  
- Research groups re-implementing models from mathematical equations?  
  
Most users are and will be porting from Fortran, but models should be able to be written from scratch with gt4py.  
Fortran ports should be a temporary v1 step in the process.  
Being able to interoperate with Fortran is important for modeling centers.  
  
What are our users now, 1 year, 5 years?  
What do we want to design today versus delay?  
e.g. halo exchanges  
   
  
## If we need to escape the DSL, how can we do this cleanly?  
  
Can we introduce a set of “foreign” IR nodes that are eliminated from certain analyses?  
  
Do-anything-you-want IR nodes - **is this possible with optimizations**?  
  
Debugging/exporting data within a timestep as a single stencil is difficult  
  
Lagrangian remapping (maybe a stencil?) is the ideal example because it’s “non-dense” in the model.  
  
  
## How should point-based control flow look?  
  
Ideas:  
  
- Free-form in DSL  
- Folds/reductions  
  
