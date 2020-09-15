# Participants  
  
@Tobias W @Rhea G Linus, Enrique, @Eddie D, @Oliver E  
  
# Action Points  
  
@Enrique G. P Start collecting ideas for explicit indexing and directional offsets for fields in [doc](https://docs.google.com/document/d/1J0i89ZqITf-s27CrE215OsVI1MWSXx_aC2TZErxbEI8/edit#)  
@Johann D Will send around poll to find time for a gt4py mini-workshop  
@Johann D Will setup a meeting to review GDP implementation with gt4py team  
  
# Review action points from last meeting  
  
Team: discuss solution to pass FV3 stencils with validation tests to partners  
  
- Second dawn PR is making progress  
- At some point FV3 dycore will be an open repo, but may not be advertised until more digestible  
  
Team: members should review storage GDP and comment — [GridTools/gt4py#28](https://github.com/GridTools/gt4py/pull/28)  
  
- Quantity work gave some interface ideas  
- Want to ensure that unstructured grids are naturally supported in storages  
  
All: will write README explaining domain and origin, with diagrams  
@Johann D will document specific issues that arose with automatic domain and origin in the region GDP discussion — [GridTools/gt4py#24](https://github.com/GridTools/gt4py/pull/24)  
  
- Workaround: specify domain and origin at all times  
# Discussion Points  
## Storages  
- API feedback  
- Lower dimensional fields are high priority for fv3 and other codes. Is there a way that Johann could help get a prototype working sooner?  
  
  
## Dawn backend merge into master  
  
Dawn PR is in gt4py repo, will be reviewed soon  
Parallel model merge in dawn should not affect gt4py interface  
Create 0.0.3 tag in dawn after merge  
  
  
## Math Functions  
  
Tobias is on it!!!  
Splitting intrinsic and non-intrinsic functions  
  
  
## FV3 Update  
  
Quantity?  
Plan to test fv3 stencils with gt4py after dawn merge of the parallel model  
  
  
## Regions  
  
Would like to get review in the next week  
Clarify domain and origin  
  
- Domain could be required unless output fields all have the same size  
  
  
## Runtime vertical splits  
  
Make grid-data available  
Runtime values for vertical splits?  
  
  
## Features  
  
Directional offsets  
Lookup tables  
Absolute indexing in k  
