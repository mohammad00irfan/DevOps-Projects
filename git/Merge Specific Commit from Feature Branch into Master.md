---

## Merge Specific Commit from Feature Branch into Master

**Objective:**
The team needed to merge a single commit (`Update info.txt`) from the `feature` branch into the `master` branch without merging the entire feature branch.

**Steps Performed:**

1. **Navigate to the project repository on the storage server**

```bash
cd /usr/src/projects/official
```

2. **Check the commit history on `feature` branch**

```bash
git checkout feature
git log --oneline
```

> Identified the commit to merge: `47579d8 Update info.txt`

3. **Switch to the master branch**

```bash
git checkout master
```

4. **Cherry-pick the specific commit from `feature` branch**

```bash
git cherry-pick 47579d8
```

* This applied only the selected commit to `master`.
* No other in-progress commits from `feature` were merged.

5. **Verify the commit is on `master`**

```bash
git log --oneline
```

6. **Push the updated master branch to the remote repository**

```bash
git push origin master
```

**Result:**
The `Update info.txt` commit from the `feature` branch is now successfully merged into `master` without affecting other work in progress. ✅

---
