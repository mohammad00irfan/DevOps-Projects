
## 📦 Git Repository Cloning on Storage Server

### 📌 Task Overview

The Nautilus DevOps team had previously created a bare Git repository on the Storage Server. The application development team requested a working copy of this repository without altering the original repository or system permissions.

### 📂 Repository Details

* **Source repository:** `/opt/demo.git`
* **Target directory:** `/usr/src/kodekloudrepos`
* **User used:** `natasha`
* **Repository type:** Bare repository (cloned as a working copy)

### ⚙️ Steps Performed

1. Switched to the required user:

   ```bash
   su - natasha
   ```

2. Navigated to the destination directory:

   ```bash
   cd /usr/src/kodekloudrepos
   ```

3. Cloned the repository from the local path:

   ```bash
   git clone /opt/demo.git
   ```

### ✅ Result

* The repository was successfully cloned into `/usr/src/kodekloudrepos/demo`
* No changes were made to:

  * Repository content
  * File permissions
  * Ownership of existing directories
* Task completed strictly using the **natasha** user as required

### 📝 Notes

* Since the source was a bare repository, cloning created a proper working directory.
* Local path cloning was used to avoid unnecessary network or permission changes.


