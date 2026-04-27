# Liquibase Secure 5.1.1 — Hello World (Maven + PostgreSQL + AWS Secrets Manager + AWS S3)

A minimal Maven project demonstrating Liquibase Secure 5.1.1 connected to PostgreSQL,
with all sensitive values (license key, DB URL, username, password) stored in AWS Secrets Manager,
and Liquibase Secure reports written directly to an AWS S3 bucket.

## Project layout

```
liquibase-hello-world/
├── pom.xml
└── src/main/resources/
    ├── liquibase.properties        ← Secrets Manager refs (no plain-text secrets)
    └── db/changelog/
        ├── db.changelog-master.xml ← master changelog
        └── 001-hello-world.xml     ← creates greeting table + seeds a row
```

## Prerequisites

| Tool                      | Version  |
|---------------------------|----------|
| Java                      | 17+      |
| Maven                     | 3.8+     |
| Liquibase Secure license  | required |
| AWS credentials configured| required |

## Step 1 — Create your AWS Secrets Manager secret

Create a single secret named `liquibase-credentials` with these JSON keys:

```json
{
  "license_key":  "<your-liquibase-secure-license-key>",
  "db_url":       "jdbc:postgresql://<host>:5432/<database>",
  "db_username":  "<your-db-user>",
  "db_password":  "<your-db-password>"
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

## Step 2 — Create your AWS S3 bucket

Create an S3 bucket to store Liquibase Secure HTML reports.

AWS CLI example:
```bash
aws s3api create-bucket \
  --bucket your-bucket \
  --region us-east-1
```

Then update the `liquibase.reports.path` system property in `pom.xml` to match:

```xml
<property>
  <name>liquibase.reports.path</name>
  <value>s3://your-bucket/liquibase-reports</value>
</property>
```

## Step 3 — Configure AWS credentials

Any standard AWS credential method works (CLI profile, environment variables, or IAM role).

The same identity needs permissions for both Secrets Manager and S3:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue",
    "secretsmanager:DescribeSecret",
    "s3:PutObject",
    "s3:GetObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:secretsmanager:<region>:<account-id>:secret:liquibase-credentials*",
    "arn:aws:s3:::your-bucket",
    "arn:aws:s3:::your-bucket/liquibase-reports/*"
  ]
}
```

## Step 4 — Run Liquibase

```bash
mvn liquibase:update                                  # apply all pending changesets
mvn liquibase:status                                  # preview pending changesets (read-only)
mvn liquibase:updateSQL                               # preview SQL without executing
mvn liquibase:rollback -Dliquibase.rollbackCount=1    # roll back the last changeset
mvn liquibase:changelogSync                           # populate the DATABASECHANGELOG table for all existing changesets
```

After each `update`, a Secure HTML report is uploaded to your S3 bucket at the path
configured in `pom.xml`, e.g. `s3://your-bucket/liquibase-reports/Update-report-<timestamp>.html`.

## How the integrations work

### Secrets Manager

`liquibase.properties` uses the syntax `aws-secrets,<secret-name>,<secret-key>`:

```properties
liquibase.licenseKey = aws-secrets,liquibase-credentials,license_key
url                  = aws-secrets,liquibase-credentials,db_url
username             = aws-secrets,liquibase-credentials,db_username
password             = aws-secrets,liquibase-credentials,db_password
```

At runtime, the Liquibase AWS extension (`com.liquibase.ext:liquibase-aws-extension:5.1.1`)
fetches each value from your secret before any database connection is attempted.
No secrets are stored in the POM or on disk.

### S3 reports

The `liquibase.reports.path` system property in `pom.xml` is set to an `s3://` URL.
The same AWS extension intercepts this path and uploads the HTML report file to S3
after each command completes, instead of writing it to the local filesystem.
