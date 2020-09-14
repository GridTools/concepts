# Action Points  
  
**Review of action points of last meeting.**  
  
[ ] @Enrique G. P Start collecting ideas for explicit indexing and directional offsets for fields in [doc](https://docs.google.com/document/d/1J0i89ZqITf-s27CrE215OsVI1MWSXx_aC2TZErxbEI8/edit#)  
- @Enrique G. P will help prioritize the items  
[ ] @Johann D Will send around poll to find time for a gt4py mini-workshop  
- Purpose of the meeting will be to decide syntax and other details related to the workshop topics  
- Focus on 1-2 issues  
[ ] @Johann D Will setup a meeting to review GDP implementation with gt4py team  
  
**Action points of this meeting:**  
  
[x] Complete Doodle poll:  
    - [https://doodle.com/poll/zn2nhrv46et9hiem](https://doodle.com/poll/zn2nhrv46et9hiem)  
****[ ] Submit items for mini-workshop in this document:  
    - [https://docs.google.com/document/d/1hFQ0TfO7fAiz7Xh-LE3RQucesI_k0tuCGku6pmz9qnY/edit](https://docs.google.com/document/d/1hFQ0TfO7fAiz7Xh-LE3RQucesI_k0tuCGku6pmz9qnY/edit)  
[x] Review Region GDP in light of figure discussion  
[ ] Schedule a meeting to discuss before next instance of this meeting  
[ ] Characterize minimum stencil overhead  
[ ] Add absolute indexing and offset-center writes to the list of items in the GTScript syntax workshop  
[ ] Select examples for off-center writes  
[ ] Check whether no-check functionality already in StencilObject  
[ ] Determine best way to share meeting notes — prefer a single link  
[ ] Convention for GT4Py issue submission — will iterate  
[ ] Present results of student projects that @Oli F and @Tobias W are working on — deadline is Aug 7  
# Discussion Points  
## Math functions  
- @Tobias W is working on this from the SIR side in Dawn and from the GT4Py side  
  
  
## Stencil overhead  
- Observed ~5ms overhead  
    - Known issue  
    - Experiment with how the overhead scales with number of input fields  
    - Python code generated in modules will be removed  
    - Characterize pybind11 overhead — this will be the minimum overhead  
    - What is the minimum acceptable overhead (upper bound)? — within a few percent of the dycore runtime  
    - Not currently a high priority for GT4py team  
    - @Johann D  is planning to submit PR for debug mode  
    - Functionality might already be there — need to be check  
  
  
## Regions  
- @Enrique G. P and @Johann D had a meeting to discuss the Regions GDP  
- Discussion of @Enrique G. P ‘s figure:  
![](https://paper-attachments.dropbox.com/s_29BD4C5C5F9BC0EB23379F0306EA720B6B6E8B4D3E55F6D0538BC148364F1815_1595348347809_domain_regions_ink.svg)  
  
    - Fields are aligned for the local compute domain via origins  
    - Local compute domains need to be mapped to global application domain and its halos  
    - Halos can be used to locate corner and edge points  
    - Temporaries are generated on the fly — compute temporary origins from those of input fields  
    - Need a global frame of reference — each field could store its location on the global grid, maybe only one field needs this, others can compute from origin  
    - Information will also be needed for halo updates  
    - Halo and origin are separate concepts  
    - What does the region object need to know or compute?  
    - Should the generated code answer these questions? — will be more efficient than Python  
- @Johann D  plans to proceed with reference implementation  
  
  
## Vertical treatment  
- Absolute indexing  
- Off-center writes  
    - Always in the vertical  
    - Field adapters to shift the fields  
    - Shift the output field and write into the shifted field — will appear as centered write  
    - Can one pass the same field with different origins?  
    - Separate the vertical computations from the stencil  
    - Could be done in the GT4Py frontend  
    - How will analysis work in Dawn, e.g., field versioning?  
- These to be added to the GTScript syntax workshop  
- Increasing in priority as more FV3 code is ported  
