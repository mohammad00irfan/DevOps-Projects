# Git Repository Setup on Storage Server (Stratos DC)

## 📌 Overview

As part of the Nautilus application development project, the DevOps team was tasked with setting up a centralized Git repository on the **Storage Server** in **Stratos Datacenter**.
This repository will act as a shared remote repository for development teams.

---

## 🛠️ Tasks Performed

### 1️⃣ Installed Git Using `yum`

Git was installed on the Storage Server using the system package manager to ensure compatibility and stability.

```bash
yum install -y git
```

**Verification:**

```bash
git --version
```

---

### 2️⃣ Created a Bare Git Repository

A **bare repository** was created, which is the recommended setup for centralized/shared Git repositories.

```bash
cd /opt
git init --bare demo.git
```

📁 **Repository Location:**

```
/opt/demo.git
```

---

### 3️⃣ Validation

Confirmed that the repository was created successfully and contains the expected Git structure.

```bash
ls -ld /opt/demo.git
```

Expected directories include:

* `objects/`
* `refs/`
* `HEAD`
* `config`

---

## ✅ Final Outcome

* Git successfully installed on the Storage Server
* Bare Git repository created with the exact required name
* Repository ready for use as a central remote by development and DevOps teams

---

## 🧠 Key Notes

* A **bare repository** does not contain a working directory and is ideal for team collaboration.
* This setup follows standard DevOps best practices for source code management in shared environments.

---

⭐ *This task demonstrates hands-on experience with Linux package management and Git repository administration in an enterprise infrastructure setup.*


