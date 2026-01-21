# Automated Git Release Tagging with Post-Update Hook

## 🎯 Overview

A server-side Git hook that automatically creates date-stamped release tags (`release-YYYY-MM-DD`) whenever code is pushed to the master branch.

## 🚀 Quick Start

### Setup Hook

```bash
# Create post-update hook in bare repository
cat > /opt/games.git/hooks/post-update << 'EOF'
#!/bin/bash

for refname in "$@"
do
  if [[ "$refname" == "refs/heads/master" ]]; then
    TODAY=$(date +%F)
    TAG_NAME="release-$TODAY"
    GIT_DIR=/opt/games.git
    
    if git --git-dir="$GIT_DIR" rev-parse "$TAG_NAME" >/dev/null 2>&1; then
      echo "Tag $TAG_NAME already exists."
    else
      git --git-dir="$GIT_DIR" tag -a "$TAG_NAME" -m "Release for $TODAY"
      echo "Tag $TAG_NAME created successfully."
    fi
  fi
done
EOF

chmod +x /opt/games.git/hooks/post-update
```

### Configure Git User

```bash
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

### Test

```bash
# Push to master triggers automatic tagging
git push origin master

# Verify
git fetch --tags
git tag -l
```

## 📊 Results

```bash
$ git push origin master
remote: Tag release-2026-01-15 created successfully.

$ git tag -l
release-2026-01-15
```

## 💡 Key Features

- ✅ Automatic release tagging on master branch pushes
- ✅ Date-based naming (`release-YYYY-MM-DD`)
- ✅ Prevents duplicate tags
- ✅ Server-side execution (bare repository)

## 🔑 Key Learnings

1. **post-update** hook uses command-line args (`$@`), not stdin
2. Hooks go in **bare repository** (`/opt/games.git/hooks/`)
3. Git user config required for tag creation
4. Hook must be executable (`chmod +x`)

## 🎓 Use Cases

- Daily release cycles
- CI/CD integration
- Automated version tracking
- Audit trails

## 🛠️ Tech Stack

Git · Bash · Linux

---

**Status**: ✅ Production Ready
