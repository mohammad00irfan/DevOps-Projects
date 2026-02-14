---

## Revert Latest Commit in Demo Repository

**Repository:** `/usr/src/githubrepos/demo`
**Environment:** Storage server

### Task

The development team reported an issue with the latest commit in the `demo` repository. The DevOps team was asked to **revert the repository HEAD to the previous commit** and create a new commit with the message `revert demo`.

### Steps Performed

1. **Navigate to the repository**

```bash
cd /usr/src/githubrepos/demo
```

2. **Check the commit history**

```bash
git log --oneline
```

Output:

```
78ece94 (HEAD -> master, origin/master) add data.txt file
3f11530 initial commit
```

3. **Revert the latest commit and set custom message**

```bash
# Revert the latest commit (HEAD)
git revert 78ece94 --no-edit

# Amend the commit message to required format
git commit --amend -m "revert demo"
```

4. **Verify the changes**

```bash
git log --oneline
```

Expected output:

```
abcdefg (HEAD -> master) revert demo
78ece94 add data.txt file
3f11530 initial commit
```

**Outcome:**

* The latest commit (`add data.txt file`) has been reverted.
* A new commit `revert demo` was created as the HEAD of the repository.
* The repository is now safe and ready for further commits.

---
