---
title: "Configuring PostgreSQL and Account Access Credentials"
description: ""
weight: 100
---

This document describes the configuration methods for credentials required by SonarQube instances.

## Prerequisites

- This document applies to SonarQube 9.9.5 and above versions provided by the platform. It is decoupled from the platform using technologies such as Operator.

## PostgreSQL Credentials \{#pg-credentials}

Create a Secret, select the Opaque type, and add and fill in the following fields in the configuration items:

| Field         | Description                                                                                | Required        | Example Value        |
| ------------ | ------------------------------------------------------------------------------------------- | ------------- | ------------- |
| **host**     | The connection address of the database. Ensure that the SonarQube service can connect to this database address. | No, only required when creating a SonarQube instance through a template.            | `192.168.1.1` |
| **port**     | The connection port of the database. Ensure that the SonarQube service can connect to this database port. | No, only required when creating a SonarQube instance through a template.            | `5432`        |
| **username** | The database account username.                                                                | No, only required when creating a SonarQube instance through a template.             | `postgres`    |
| **jdbc-password** | The database password.                                                                      | Yes             |               |
| **database** | The database name. This database must already exist and be empty. You can use the command `create database <database name>` to create a database. | No, only required when creating a SonarQube instance through a template.             | `sonar_db`    |


YAML example:


```yaml
apiVersion: v1
stringData:
  host: 192.168.1.1
  port: 5432
  username: postgres
  jdbc-password: pg-password
  database: sonar_db
kind: Secret
metadata:
  name: sonarqube-pg
type: Opaque
```

**How to Create a Database on a PG Instance**

Connect to the PG instance using the psql cli and execute the following command to create a database:

```bash
create database <database name>;
```

## SonarQube Account Credentials \{#sonarqube-credentials}

The default login username is `admin` and the password must meet the following requirements:

- At least 12 characters
- Include 1 uppercase letter
- Include 1 lowercase letter
- Include 1 number
- Include 1 special character

Create a Secret, using the Opaque type, and add a `password` field in the configuration items:

```yaml
apiVersion: v1
data:
  password: <base64 encode password>
kind: Secret
metadata:
  name: sonarqube-root-password
type: Opaque
```
