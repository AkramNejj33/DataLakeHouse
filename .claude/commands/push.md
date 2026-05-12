Run the following git commands sequentially to stage, commit and push all changes to GitHub:

1. Run `git status` to show current state
2. Run `git add .` to stage all changes
3. Run `git diff --cached --stat` to show what will be committed
4. If there are staged changes, ask the user for a commit message, then run `git commit -m "<message>"`. If the user provided arguments to this command (via $ARGUMENTS), use those as the commit message directly without asking.
5. Run `git push origin main` to push to GitHub

If there is nothing to commit, inform the user that the repo is already up to date.
