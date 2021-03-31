# GridTools Concepts

This repository is wiki-only. The [wiki](https://github.com/GridTools/concepts/wiki) is managed via PRs to this repository.

## How to edit the Wiki?

I see 3 main work-flows:

1. Edit locally
   - `git clone git@github.com:GridTools/concepts.git`
   - Edit file, commit and push to a new branch (in this repository or your fork).
   - Open a PR

2. Edit online in the main repository
   - Go to [https://github.com/GridTools/concepts](https://github.com/GridTools/concepts)
   - Click on a file to edit
   - Click the edit file button at the top right
   - Edit the file and commit changes into a new branch
   - Open a PR

3. Edit directly in the wiki (use with care)
   - Go to [https://github.com/GridTools/concepts/wiki](https://github.com/GridTools/concepts/wiki)
   - Select a page
   - Click on "Edit"
   - Save your changes
   - This will open a PR on the main repository, but doesn't apply your changes directly. (Actually they are first applied directly, but then reverted a few seconds later.)


## How does it work?

The source of truth is the main repository `github.com:GridTools/concepts.git`. On GitHub, the wiki is kept in a repository at `github.com:GridTools/concepts.wiki.git`. On each merge to master, the main repository is mirrored to the wiki repository (overwriting all changes in the wiki repository) by a GitHub action.

Additionally, direct changes to the wiki (workflow 3 above) are intercepted by a GitHub action. The changing commit is added on top the main master in a new branch and a PR will be opened. Then the changes to the wiki repository are reverted to the state of the main repository.
There are about 10-20 seconds, while the action is running, in which the wiki is in an inconsistent state. Additionally there is a race condition if the main repository is updated at the same time as the wiki edit is committed. The main repository will always win and potentially the wiki edit is gone forever...
