---
title: Backup and Recovery
weight: 100
---

## Terminology

| Term | Description                                                 |
| ---- | ----------------------------------------------------------- |
| PG   | PostgreSQL database, used by SonarQube to store core data    |
| ES   | Elasticsearch search engine, used by SonarQube for searching |

## Scope

This document applies to SonarQube 9.9.5 and above versions.

:::warning
This document has been verified with SonarQube 9.9.5. While the procedures should theoretically work with version 25.1.0, they have not yet been practically validated with this version. Please test in a non-production environment first if you're using SonarQube 25.1.0.
:::

## Overview

SonarQube data primarily consists of two parts:

1. PostgreSQL database data (core data)
2. Elasticsearch data (search data)

Since Elasticsearch data can be regenerated from the database, we **only need to backup the PostgreSQL database** to ensure data security.

## Backup

Based on the type of PostgreSQL database you use, there are two backup methods:

- If using the PG instance provided by the platform (recommended when deploying SonarQube instances):
  - You can directly use the platform's built-in database backup feature
  - Supports manual and scheduled automatic backups
  - Simple operation with good user experience
- If using a self-built PG instance, you need to complete the backup yourself

### Method 1: Backing up Platform-Provided PostgreSQL Instance (Recommended)

If you are using a PostgreSQL instance provided by the platform, you can directly use the platform's backup feature:

1. Go to the Data Services view
2. Find your PostgreSQL instance
3. Open the `Backup and Recovery` tab
4. Follow the page prompts to configure the backup

The platform PG instance supports automatic backup, which can be enabled during the backup configuration process. For specific operation methods, refer to the [Platform PG Backup and Recovery](#how-to-view-platform-postgresql-backup-and-recovery-documentation) documentation.

After completing the backup configuration, you can:
1. Click `Create Backup` to perform an immediate backup
2. Check the backup status in the backup records


### Method 2: Backing up Self-Built PostgreSQL Instance

If using a self-built database, you need to use the `pg_dump` tool for backup. Here are the basic operation steps:

```bash
pg_dump -U postgres -h 127.0.0.1 -p 5432 -d sonar -f sonar.dump
```

Parameter description:
| Parameter       | Description      |
| --------------- | ---------------- |
| `-U postgres`   | Database username|
| `-h 127.0.0.1`  | Database address |
| `-p 5432`       | Database port    |
| `-d sonar`      | Database name    |
| `-f sonar.dump` | Backup filename  |

Tip: You can use `crontab` to implement automatic scheduled backups.

## Recovery

Data recovery requires two steps:

1. Restore the PostgreSQL database
2. Deploy a new SonarQube instance and connect it to the restored database

> Important note: It is recommended to create a new SonarQube instance rather than modifying the database configuration of the original instance, which can avoid data loss due to misoperation.

### Database Recovery

#### Restoring Platform PostgreSQL Instance

1. Go to the Data Services view
2. Find the target PostgreSQL instance
3. Open the `Backup and Recovery` tab
4. Follow the `Database Recovery` wizard to complete the restoration

Note: The recovery operation will create a new database instance.

#### Restoring Self-Built PostgreSQL Instance

Self-built PG instances need to complete the recovery operation on your own. The following recovery commands are for reference only:

Command parameter description:
| Parameter       | Description       |
| --------------- | ----------------- |
| `-U postgres`   | Database username |
| `-h 127.0.0.1`  | Database address  |
| `-p 5432`       | Database port     |
| `-d sonar_new`  | New database name |
| `-f sonar.dump` | Backup filename   |

1. Create a new database:

    ```bash
    psql -U postgres -h 127.0.0.1 -p 5432 -c "CREATE DATABASE sonar_new"
    ```

2. Import backup data:

    ```bash
    psql -U postgres -h 127.0.0.1 -p 5432 -d sonar_new < sonar.dump
    ```

3. Verify recovery results:

    ```bash
    psql -U postgres -h 127.0.0.1 -p 5432 -d sonar_new -c "\dt"
    ```

    If you can see the list of data tables, the recovery is successful:

    ```
                      List of relations
    Schema |           Name            | Type  |  Owner
    --------+---------------------------+-------+----------
    public | active_rule_parameters    | table | postgres
    public | active_rules              | table | postgres
    ...
    ```

### Deploying a New SonarQube Instance

Refer to the SonarQube deployment documentation to create a new SonarQube instance.

Please note the following key points:

1. The newly deployed SonarQube version must be **the same version** as the original instance
2. When deploying, you need to correctly configure the database connection information, which is the database you created in the database recovery step

For methods to configure database access credentials, please refer to the SonarQube Deployment Documentation.

