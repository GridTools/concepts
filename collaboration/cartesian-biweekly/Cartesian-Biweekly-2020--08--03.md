---
title: 2020-08-03
---

# Action Points

**Review of action points of last meeting:**

[x] Complete Doodle poll:
    - [https://doodle.com/poll/zn2nhrv46et9hiem](https://doodle.com/poll/zn2nhrv46et9hiem)
[ ] Submit items for mini-workshop in this document:
    - [https://docs.google.com/document/d/1hFQ0TfO7fAiz7Xh-LE3RQucesI_k0tuCGku6pmz9qnY/edit](https://docs.google.com/document/d/1hFQ0TfO7fAiz7Xh-LE3RQucesI_k0tuCGku6pmz9qnY/edit)
[x] Review Region GDP in light of figure discussion
[x] Schedule a meeting to discuss before next instance of this meeting
[x] Characterize minimum stencil overhead
[x] Add absolute indexing and offset-center writes to the list of items in the GTScript syntax workshop
[x] Select examples for off-center (vertically) writes
[x] Check whether no-check functionality already in StencilObject
[x] Determine best way to share meeting notes — prefer a single link
[x] Convention for GT4Py issue submission — will iterate
[x] Present results of student projects that @Oli F and @Tobias W are working on — deadline is Aug ~~7~~ 16 - present on 18th

Rhea will file an issue to share off-center write examples
Johann will review the two ideas for regions and resolve differences
Enrique will add ideas to the syntax workshop

# Discussion Points
## Review PRs, please!
1. Keep PRs as drafts until they are ready for review
2. Assign someone - may be re-assigned as necessary

**Ready for review**

- ~~Reduce overhead - https://github.com/GridTools/gt4py/pull/117~~
- ~~Add BOOL - https://github.com/GridTools/gt4py/pull/129~~
- Math functions - https://github.com/GridTools/gt4py/pull/125

**Needs a test**

- Dawn extent fix - https://github.com/GridTools/gt4py/pull/101


## Interesting new issues
- Scalars become temporary fields - https://github.com/GridTools/gt4py/issues/103
- Dimensions of temporaries - https://github.com/GridTools/gt4py/issues/107
- Determining type inside an if statement - https://github.com/GridTools/gt4py/issues/106
- Functions calls with scalar literals - https://github.com/GridTools/gt4py/issues/104


## Dialect Workshop
- **18, 19,** 21st August are best, but Matthias cannot join
- Reminder: add issues to the document
    - Directional offsets
    - Absolute k indexing
    - Named regions


## Backend Compatibility
- Race condition PR in progress - https://github.com/GridTools/gt4py/pull/121
- Numpy backend warns about division by zero


    with computation(PARALLEL), interval(…):
        tmp = field[0,0,1]
        tmp2 = tmp + 1
        field = tmp2

        if __INLINED(flag):
            tmp = field[1,0,0]
            tmp2 = tmp + 1
            field = tmp2


## End to End Tests
- Discuss ideas for how this could be structured
- Increase number of integration tests, or put them elsewhere?
- Existing types of tests: unit tests, code generation


- Do not make gt4py repo dependent on regression tests
- Eventually: mergeable/merged PRs should trigger dycore builds, and tests should be visible
- If we find problems, add reproducers as regressions tests


## Regions
- Agreement on a lightweight splitter-based approach. Example:
    @gtscript.stencil(splitters={I: ['i0', 'ie'], J: ['j0', 'je']})
    def divergence_corner(...):
      from __splitters__ import i0, ie, j0, je
      with computation(FORWARD), interval(...):
        uf = (u - 0.25*(va[0, -1, 0] + va)*(cos_sg4[0, -1, 0] + cos_sg2))  \
                                   *dyc*0.5*(sin_sg4[0, -1, 0] + sin_sg2)
        with region[i0 : i0 + 1, :], region[ie - 1 : ie, :]:
          uf = u*dyc*0.5*(sin_sg4[0, -1, 0] + sin_sg2)

        vf = (v - 0.25*(ua[-1, 0, 0] + ua)*(cos_sg3[-1, 0, 0] + cos_sg1))  \
                                   *dxc*0.5*(sin_sg3[-1, 0, 0] + sin_sg1)
        with region[:, j0 : j0 + 1], region[:, je - 1 : je]:
          vf = v*dxc*0.5*(sin_sg3[-1, 0, 0] + sin_sg1)

        divg_d = rarea_c * (vf[0, -1, 0] - vf + uf[-1, 0, 0] - uf)
        with region[i0 : i0 + 1, j0 : j0 + 1], region[ie - 1 : ie, j0 : j0 + 1]:
          divg_d = rarea_c * (-vf + uf[-1, 0, 0] - uf)
        with region[i0 : i0 + 1, je - 1 : je], region[ie - 1 : ie, je - 1 : je]:
          divg_d = rarea_c * (vf[0, -1, 0] + uf[-1, 0, 0] - uf)


## Review status of GDP3-Duck Storages
- Status update on the state of the GDP3 implementation including 1D/2D fields
