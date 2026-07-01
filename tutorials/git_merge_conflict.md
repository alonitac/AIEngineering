# Git merge conflicts in VS Code

A **merge conflict** happens when Git can't automatically combine changes from two branches. This occurs when:

- Two branches modify the **same lines** in a file
- One branch **deletes** a file the other branch modifies
- Two branches add **different content** at the same location (usually at the end of the file)

Git pauses the merge and marks the conflicts for you to resolve manually.


## create a merge conflict

```bash
# Start from dev branch
git checkout dev

# Create a new branch and make a change
git checkout -b feature-branch
echo "Hello from feature branch" > hello.txt
git add hello.txt
git commit -m "Add hello from feature branch"

# Go back to dev and make a conflicting change to the same file
git checkout dev
echo "Hello from dev branch" > hello.txt
git add hello.txt
git commit -m "Add hello from dev branch"

# Try to merge - this will cause a conflict
git merge feature-branch
```

Git will output something like:
```
CONFLICT (content): Merge conflict in hello.txt
Automatic merge failed; fix conflicts then commit the result.
```

## Recognize conflicts in VS Code

Open VS Code. You'll see:

- **Source Control** panel shows a **Merge Changes** section with the conflicted file(s)

> **Note:** If the **Merge Changes** section doesn't appear, click the **Refresh** button (↻) at the top of the Source Control panel. You can find it by opening the Source Control panel (`Ctrl+Shift+G`), then looking at the top bar of the panel next to the title "Source Control".

Open the conflicted file and you'll see **conflict markers**:

```
<<<<<<< HEAD
Hello from dev branch
=======
Hello from feature branch
>>>>>>> feature-branch
```

| Marker | Meaning |
|--------|---------|
| `<<<<<<< HEAD` | Start of **your** (current branch) changes |
| `=======` | Separator between the two versions |
| `>>>>>>> branch-name` | End of **incoming** branch changes |


Above each conflict, VS Code shows clickable actions:

| Action | Result |
|--------|--------|
| **Accept Current Change** | Keep your branch's version |
| **Accept Incoming Change** | Keep the incoming branch's version |
| **Accept Both Changes** | Keep both, one after the other |
| **Compare Changes** | Open a side-by-side diff view |

**To resolve:**
1. Click one of the CodeLens actions above the conflict
2. The conflict markers disappear and the chosen version remains
3. Save the file (`Ctrl+S`)
4. In the Source Control panel, stage the file (click `+`)
5. Commit: `git commit -m "Resolve merge conflict"`



For complex conflicts where you want to combine parts of both changes - right-click the conflicted file in **Source Control** → **Open in Merge Editor**  
   *(or click the **Resolve in Merge Editor** button at the top of the file)*

The merge editor has **three panels**:

| Panel | Shows |
|-------|-------|
| **Incoming** (left) | Changes from the branch being merged |
| **Current** (right) | Changes from your current branch |
| **Result** (bottom) | The final merged output you control |

**To use it:**
1. Review Incoming and Current panels
2. Use the checkboxes/buttons next to each conflict to:
   - Accept **Incoming** or **Current**
   - Accept a **Combination** of both
   - **Ignore** (exclude from result)
3. The Result panel updates live - you can also edit it directly
4. Watch the **conflict counter** (bottom right) until it reaches 0
5. Click **Complete Merge** to stage and close


Made a mistake or want to start over?

```bash
git merge --abort
```


