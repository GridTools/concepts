# 2021/03/16 GT4Py bi-weekly  
  
# To Dos  
[ ]  Will ensure CSCS can compile microphysics code  
## Review To Dos from previous sync  
# Agenda  
## GTC Work Cycle  
  
Projects look useful!  
Plan to finish up higher dimensional fields to unblock work in sprint https://github.com/GridTools/gt4py/pull/338  
  
  
## For Loops  
  
Ensure work is coordinated with GTC cycle project  
  
    def example_stencil(field: Field[float]):  
      for i in range(ITER):  
        with computation(PARALLEL), interval(...):  
          ...  
  
  
## Parallel multistages fusing  
  
[GridTools/gt4py#246](https://github.com/GridTools/gt4py/issues/246)  
[GridTools/gt4py#375](https://github.com/GridTools/gt4py/pull/375)  
  
Problem addressed in GridTools v2 using [GridTools/gridtools#1612](https://github.com/GridTools/gridtools/pull/1612)  
  
  
## GridTools Extents and Horizontal Ifs  
  
[+GridTools Extents and Horizontal Ifs](https://paper.dropbox.com/doc/GridTools-Extents-and-Horizontal-Ifs-2QUNSLQtMjtWx3a1YiiNu)   
We have a workaround  
  
https://github.com/jdahm/gt4py/commit/b06456188ad93d88950d863d28657103974f233b  
  
  
