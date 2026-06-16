# MongoDB User Management — DBA Guide

A comprehensive, step-by-step reference for managing users in MongoDB.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Enable Authentication](#2-enable-authentication)
3. [Connect to MongoDB Shell](#3-connect-to-mongodb-shell)
4. [Built-in Roles Reference](#4-built-in-roles-reference)
5. [Create Users](#5-create-users)
6. [View Users](#6-view-users)
7. [Update Users](#7-update-users)
8. [Grant & Revoke Roles](#8-grant--revoke-roles)
9. [Change Passwords](#9-change-passwords)
10. [Delete Users](#10-delete-users)
11. [Custom Roles](#11-custom-roles)
12. [User Management Across Databases](#12-user-management-across-databases)
13. [Best Practices](#13-best-practices)

---

## 1. Prerequisites

- MongoDB installed (v4.x or higher recommended)
- Access to `mongosh` (MongoDB Shell) or legacy `mongo` shell
- An admin account or root access for initial setup
- `mongod` running with or without `--auth` flag

---

## 2. Enable Authentication

Authentication must be enabled before user management takes effect.

### Option A — Command-line flag

```bash
mongod --auth --port 27017 --dbpath /var/lib/mongo
```

### Option B — Configuration file (`mongod.conf`)

```yaml
security:
  authorization: enabled
```

Then restart the service:

```bash
sudo systemctl restart mongod
```

> **Note:** On a brand-new installation with no users, MongoDB allows one localhost connection without credentials (the *localhost exception*) so you can create the first admin user.

---

## 3. Connect to MongoDB Shell

### Without authentication (initial setup only)

```bash
mongosh
```

### With authentication

```bash
mongosh --host localhost --port 27017 -u "adminUser" -p "password" --authenticationDatabase "admin"
```

### Switch to a database inside the shell

```js
use admin        // switch to admin database
use myDatabase   // switch to your target database
```

---

## 4. Built-in Roles Reference

### Database-level roles

| Role | Permissions |
|------|-------------|
| `read` | Read all non-system collections |
| `readWrite` | Read and write all non-system collections |
| `dbAdmin` | Schema, indexes, statistics (no user data access) |
| `userAdmin` | Create/modify users and roles in the database |
| `dbOwner` | All of the above combined |

### Admin / cluster-level roles

| Role | Permissions |
|------|-------------|
| `readAnyDatabase` | Read on all databases |
| `readWriteAnyDatabase` | Read/write on all databases |
| `userAdminAnyDatabase` | Manage users on all databases |
| `dbAdminAnyDatabase` | Admin tasks on all databases |
| `clusterAdmin` | Full cluster management |
| `root` | Superuser — all privileges |
| `backup` | Perform `mongodump` backups |
| `restore` | Perform `mongorestore` restores |

---

## 5. Create Users

### 5.1 Create the first admin user

Run this while connected via the localhost exception (before auth is enabled):

```js
use admin

db.createUser({
  user: "adminUser",
  pwd: "StrongPassword123!",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```

### 5.2 Create a database-specific user

```js
use myDatabase

db.createUser({
  user: "appUser",
  pwd: "AppPassword456!",
  roles: [
    { role: "readWrite", db: "myDatabase" }
  ]
})
```

### 5.3 Create a read-only user

```js
use reporting

db.createUser({
  user: "reportUser",
  pwd: "ReadOnly789!",
  roles: [
    { role: "read", db: "reporting" }
  ]
})
```

### 5.4 Create a backup user

```js
use admin

db.createUser({
  user: "backupUser",
  pwd: "BackupPass!",
  roles: [
    { role: "backup", db: "admin" }
  ]
})
```

### 5.5 Create a user with multiple database roles

```js
use admin

db.createUser({
  user: "multiDbUser",
  pwd: "MultiPass!",
  roles: [
    { role: "readWrite", db: "database1" },
    { role: "read",      db: "database2" },
    { role: "dbAdmin",   db: "database3" }
  ]
})
```

---

## 6. View Users

### List all users in current database

```js
db.getUsers()
```

### List all users across all databases (from admin)

```js
use admin
db.system.users.find().pretty()
```

### Get a specific user

```js
db.getUser("appUser")
```

### Get user with privileges shown

```js
db.getUser("appUser", { showPrivileges: true, showAuthenticationRestrictions: true })
```

---

## 7. Update Users

### Update roles entirely (replaces existing roles)

```js
use myDatabase

db.updateUser("appUser", {
  roles: [
    { role: "readWrite", db: "myDatabase" },
    { role: "read",      db: "analytics" }
  ]
})
```

### Update custom data field

```js
db.updateUser("appUser", {
  customData: { team: "backend", env: "production" }
})
```

---

## 8. Grant & Revoke Roles

### Grant additional roles

```js
use myDatabase

db.grantRolesToUser("appUser", [
  { role: "dbAdmin", db: "myDatabase" }
])
```

### Revoke specific roles

```js
use myDatabase

db.revokeRolesFromUser("appUser", [
  { role: "dbAdmin", db: "myDatabase" }
])
```

---

## 9. Change Passwords

### Change password inside the shell

```js
use myDatabase

db.changeUserPassword("appUser", "NewSecurePassword!")
```

### Alternative using updateUser

```js
db.updateUser("appUser", {
  pwd: "NewSecurePassword!"
})
```

---

## 10. Delete Users

### Drop a specific user

```js
use myDatabase

db.dropUser("appUser")
```

### Drop all users in a database

```js
use myDatabase

db.dropAllUsers()
```

> **Warning:** `dropAllUsers()` is irreversible. Use with extreme caution in production.

---

## 11. Custom Roles

Custom roles let you define precise privilege sets beyond the built-in roles.

### 11.1 Create a custom role

```js
use myDatabase

db.createRole({
  role: "dataAnalyst",
  privileges: [
    {
      resource: { db: "myDatabase", collection: "orders" },
      actions: [ "find", "listIndexes" ]
    },
    {
      resource: { db: "myDatabase", collection: "products" },
      actions: [ "find" ]
    }
  ],
  roles: []   // can inherit from other roles here
})
```

### 11.2 View custom roles

```js
db.getRoles({ showPrivileges: true })
```

### 11.3 Assign a custom role to a user

```js
db.grantRolesToUser("reportUser", [
  { role: "dataAnalyst", db: "myDatabase" }
])
```

### 11.4 Update a custom role

```js
db.updateRole("dataAnalyst", {
  privileges: [
    {
      resource: { db: "myDatabase", collection: "orders" },
      actions: [ "find", "listIndexes", "count" ]
    }
  ],
  roles: []
})
```

### 11.5 Delete a custom role

```js
db.dropRole("dataAnalyst")
```

---

## 12. User Management Across Databases

Users are stored in the `admin` database but can be scoped to any database.

### Create a user in admin that operates on another database

```js
use admin

db.createUser({
  user: "crossDbUser",
  pwd: "CrossPass!",
  roles: [
    { role: "readWrite", db: "sales" },
    { role: "read",      db: "inventory" }
  ]
})
```

### Authenticate specifying the auth database

```bash
mongosh -u "crossDbUser" -p "CrossPass!" --authenticationDatabase "admin"
```

> Users are always authenticated against the database where they were created, regardless of which database they access.

---

## 13. Best Practices

### Security

- **Enable auth from day one.** Never run production MongoDB without `--auth`.
- **Use the principle of least privilege.** Grant only the roles a user actually needs.
- **Avoid the `root` role** for application users. Use scoped roles instead.
- **Rotate passwords regularly** and use strong, unique passwords (16+ characters).
- **Disable the localhost exception** after creating the initial admin user.

### Account hygiene

- **Separate service accounts per application.** Never share credentials between apps.
- **Create read-only users for reporting tools** and BI dashboards.
- **Use `customData`** to store metadata (team, owner, created date) on user documents.
- **Audit users periodically** with `db.system.users.find()` and remove stale accounts.

### Backup & monitoring

- **Test backup users** independently from application users.
- **Enable the `auditLog`** (MongoDB Enterprise) to track authentication and user changes.
- **Monitor failed login attempts** via `mongod` logs or MongoDB Atlas alerts.

### Connection string template

```
mongodb://username:password@host:27017/database?authSource=admin
```

---

## Quick Command Cheatsheet

```js
// Create
db.createUser({ user, pwd, roles })

// Read
db.getUsers()
db.getUser("username")

// Update
db.updateUser("username", { roles / pwd / customData })
db.grantRolesToUser("username", [roles])
db.revokeRolesFromUser("username", [roles])
db.changeUserPassword("username", "newpwd")

// Delete
db.dropUser("username")
db.dropAllUsers()

// Custom roles
db.createRole({ role, privileges, roles })
db.getRoles({ showPrivileges: true })
db.updateRole("roleName", { privileges, roles })
db.dropRole("roleName")
```

---

*Guide covers MongoDB 5.x – 7.x. Always consult the [official MongoDB documentation](https://www.mongodb.com/docs/manual/reference/method/js-user-management/) for version-specific changes.*
