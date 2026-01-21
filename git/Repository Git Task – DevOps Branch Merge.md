---

# Nautilus Repository Git Task – DevOps Branch Merge

## Objective

The Nautilus application development team required the DevOps team to:

1. Create a new branch `devops` from `master`.
2. Copy the `/tmp/index.html` file into the repository.
3. Add and commit the file in the new branch.
4. Merge the `devops` branch back into `master`.
5. Push changes to the remote repository for both branches.

---

## Steps Performed

### 1. Navigate to the repository

```bash
cd /usr/src/kodekloudrepos/news
```

### 2. Switch to `master` branch

```bash
git checkout master
```

### 3. Create a new branch `devops` from master

```bash
git checkout -b devops
```

### 4. Copy the `index.html` file into the repository

```bash
cp /tmp/index.html .
```

### 5. Add and commit the file in `devops` branch

```bash
git add index.html
git commit -m "Add index.html from /tmp to devops branch"
```

### 6. Merge `devops` branch back into `master`

```bash
git checkout master
git merge devops
```

### 7. Push both branches to remote

```bash
git push origin master
git push origin devops
```

---

## Outcome

* `devops` branch was created successfully from `master`.
* `/tmp/index.html` was added and committed in `devops`.
* Changes were merged back into `master`.
* Both branches were successfully pushed to the remote repository.

---
