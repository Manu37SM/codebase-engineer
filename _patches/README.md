# Patches (delete this folder after applying)

Each folder is the commit series for one repo, based on current origin/master.
For each repo (backend/, frontend/, and the project root):

    git checkout master && git pull
    git checkout -b <branch-name>
    git am /path/to/_patches/<repo>/*.patch
    git push -u origin <branch-name>

Frontend: the first patch (react 19.x) should be applied together with the rest;
it supersedes Dependabot PRs #35 #36 #34 #37 #38 #39 (close them after merging).
Backend: supersedes Dependabot PRs #31 #32 #33 #35 #36 #37 #38.
