# Liquibase Secure 5.1.1 — Hello World (Maven + PostgreSQL + AWS Secrets Manager)

A minimal Maven project demonstrating Liquibase Secure 5.1.1 connected to PostgreSQL,
with all sensitive values (license key, DB URL, username, password) stored in AWS Secrets Manager.

## Project layout

```
liquibase-hello-world/
├── pom.xml
└── src/main/resources/
    ├── liquibase.properties             ← Secrets Manager refs (no plain-text secrets)
    └── db/changelog/
        ├── db.changelog-master.xml      ← master changelog
        └── 001-hello-world.xml          ← creates greeting table + seeds a row
```

## Prerequisites

| Tool | Version |
|------|---------|
| Java | 17+ |
| Maven | 3.8+ |
| Liquibase Secure license | required |
| AWS credentials configured | required |

## Step 1 — Create your AWS Secrets Manager secret

Create a single secret named `liquibase-credentials` with these JSON keys:

```json
{
  "license_key":   "<your-liquibase-secure-license-key>",
  "db_url":        "jdbc:postgresql://<host>:5432/<database>",
  "db_username":   "<your-db-user>",
  "db_password":   "<your-db-password>"
}
```

AWS CLI example:
```bash
aws secretsmanager create-secret \
  --name liquibase-credentials \
  --secret-string '{
    "license_key":  "YOUR_KEY",
    "db_url":       "jdbc:postgresql://myhost:5432/mydb",
    "db_username":  "myuser",
    "db_password":  "mypassword"
  }'
```

## Step 2 — Configure AWS credentials

Any standard AWS credential method works (CLI profile, env vars, or IAM role).

Minimum IAM permissions required:
```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "secretsmanager:DescribeSecret"
  ],
  "Resource": "arn:aws:secretsmanager:<region>:<account-id>:secret:liquibase-credentials*"
}
```

## Step 3 — Run Liquibase

```bash
mvn liquibase:update       # apply all pending changesets
mvn liquibase:status       # preview pending changesets (read-only)
mvn liquibase:updateSQL    # preview SQL without executing
mvn liquibase:rollback -Dliquibase.rollbackCount=1
```

## How the Secrets Manager integration works

`liquibase.properties` uses the syntax `aws-secrets,<secret-name>,<secret-key>`:

```properties
liquibase.licenseKey = aws-secrets,asmith-postgres,license_key
url                  = aws-secrets,asmith-postgres,db_url
username             = aws-secrets,asmith-postgres,db_username
password             = aws-secrets,asmith-postgres,db_password
```

At runtime, the Liquibase AWS extension (org.liquibase:liquibase-AWS-extension:1.1.3)
fetches each value from your secret before any database connection is attempted.
No secrets are stored in the POM or on disk.
