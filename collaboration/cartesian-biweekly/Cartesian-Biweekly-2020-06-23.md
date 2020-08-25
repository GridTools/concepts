# Participants

@Jeremy M @Rhea G @Mark C @Eddie D @Tobias W @Oli F …

# Action Points

Team: discuss solution to pass FV3 stencils with validation tests to partners
Team: members should review storage GDP and comment — [GridTools/gt4py#28](https://github.com/GridTools/gt4py/pull/28)
All: will write README explaining domain and origin, with diagrams
@Johann D will document specific issues that arose with automatic domain and origin in the region GDP discussion — [GridTools/gt4py#24](https://github.com/GridTools/gt4py/pull/24)

# Discussion Points
## Vulcan Status Update
****
FV3 porting status and issues (frontend)

- Full dycore ported!
- Only a few small issues came up over the last few weeks
    - Would like to get reproducers, and open github issues

Dawn backend correctness

- CPU: Almost all FV3 stencils validate
- GPU: ?

Parallel model

- Code generated with CUDA backend fails to verify on a select few clang-gridtools stencil tests


## Dawn backend merge into master

Tests pass on Enrique’s machine
Status: two branches off master, that look different but behave similarly
Will rebase branch against master and fix carefully
Have plan to merge soon, regardless of state


## Math Functions

Dawn IR needs new nodes for these functions
Dawn could use general-purpose function nodes for now
Tobias will coordinate with Enrique in the next few days on implementation details


## Storages

Plan to split into two GDPs

1. Storage implementation interface as python objects — [GridTools/gt4py#28](https://github.com/GridTools/gt4py/pull/28)
2. **Lower-dimensional fields**

Read and comment on (1) soon!

## Application story: Quantities in FV3

Overview document: [+2020-06-22 Quantity Snapshot](https://paper.dropbox.com/doc/2020-06-22-Quantity-Snapshot-bWtgw0D91vfAwHtBjYuef)


## Regions

Clarify domain and origin

- Domain could be required unless output fields all have the same size

Discussion: [GridTools/gt4py#24](https://github.com/GridTools/gt4py/pull/24)
Implementation: [GridTools/gt4py#36](https://github.com/GridTools/gt4py/pull/36)

Status: Implemented in frontend, and IRs
Current work: Python backends, dawn backend
