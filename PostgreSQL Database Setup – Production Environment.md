## PostgreSQL Database Setup – Production Environment

As part of an application deployment requirement, I configured a PostgreSQL database server to support a newly developed application..

### ✅ Work Completed

- Confirmed that the PostgreSQL server was already installed and running.
- Created a dedicated PostgreSQL database user for the application:
  - **Username:** `app_db_user`
  - **Password:** `xxxxx`
- Created a new PostgreSQL database for application usage:
  - **Database Name:** `app_database`
- Granted full privileges on the database to the application user.
- Ensured proper access control without restarting the PostgreSQL service, as per operational constraints.

### 🛠️ Technologies Used
- PostgreSQL
- Linux Command Line
- SQL

### 🎯 Outcome
The database environment was successfully prepared according to application prerequisites, enabling secure and uninterrupted deployment while following best practices for access management.
```
