# Liquibase + Snowflake OIDC (Workload Identity Federation) with GitHub Actions

This repository shows how to run Liquibase Secure against Snowflake using **Snowflake Workload Identity Federation (WIF) with OIDC**, authenticating from GitHub Actions with a short-lived GitHub OIDC token. No passwords, key pairs, or programmatic access tokens are stored.

## Tested configuration

| Component | Version / detail |
|---|---|
| Liquibase Secure | 6.0.1 |
| Snowflake JDBC driver | 4.3.4 (bundled with Liquibase in `internal/lib`) |
| GitHub Actions runner | Self-hosted, Linux x64 |
| Snowflake auth method | Workload Identity Federation, provider type `OIDC` |
| Token issuer | GitHub Actions (`https://token.actions.githubusercontent.com`) |

WIF is handled by the Snowflake JDBC driver, not by Liquibase's own Snowflake auth parameters. Any Liquibase install whose bundled `snowflake-jdbc` is **3.26.0 or later** has the required driver support. Leave `--snowflake-auth-type` **unset** when using this approach.

## How it works

1. The GitHub Actions job requests an OIDC ID token from GitHub with the audience `snowflakecomputing.com`.
2. The workflow appends that token to the Snowflake JDBC URL.
3. The Snowflake JDBC driver sends the token to Snowflake with `authenticator=WORKLOAD_IDENTITY` and `workloadIdentityProvider=OIDC`.
4. Snowflake validates the token's issuer (`iss`), subject (`sub`), and audience (`aud`) against a Snowflake **service user** configured with a matching workload identity. If all three match, Liquibase is logged in as that user.

---

## Part 1: Snowflake setup

Run these steps as `ACCOUNTADMIN`, or as a role that can create users and roles and grant privileges.

### 1.1 Create the database objects and role

Adjust names to fit your environment.

```sql
USE ROLE ACCOUNTADMIN;

-- Database, schema, and warehouse Liquibase will use
CREATE DATABASE IF NOT EXISTS LB_WIF_TEST;
CREATE SCHEMA  IF NOT EXISTS LB_WIF_TEST.PUBLIC;
CREATE WAREHOUSE IF NOT EXISTS LB_WIF_TEST_WH WAREHOUSE_SIZE = XSMALL AUTO_SUSPEND = 60;

-- Role for Liquibase
CREATE ROLE IF NOT EXISTS LB_WIF_TEST_ROLE;
GRANT USAGE ON WAREHOUSE LB_WIF_TEST_WH TO ROLE LB_WIF_TEST_ROLE;
GRANT USAGE ON DATABASE  LB_WIF_TEST    TO ROLE LB_WIF_TEST_ROLE;
GRANT USAGE, CREATE TABLE, CREATE SEQUENCE, CREATE VIEW
      ON SCHEMA LB_WIF_TEST.PUBLIC      TO ROLE LB_WIF_TEST_ROLE;
```

### 1.2 Determine the token's subject (`sub`) claim

The Snowflake service user's `SUBJECT` must **exactly** match the `sub` claim of the token GitHub issues. Don't build this value by hand. Copy it from a real token.

The workflow in this repo prints the token's non-secret claims on every run, so the easiest approach is:

1. Complete the GitHub setup in Part 2.
2. Run the workflow once. The Liquibase step will fail at this point because the user doesn't exist yet or has a placeholder subject.
3. Copy the `sub` value from the **Get GitHub OIDC token** step's output, which looks like this:

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:daticalamy@50338582/cs_snowflake_oidc@1454462333:environment:DEV",
  "aud": "snowflakecomputing.com"
}
```

**Things that affect the subject format:**

- **GitHub environments.** When the job references a GitHub environment (`environment: DEV`), the subject ends in `:environment:<NAME>`. Without an environment, it is branch-based instead, such as `...:ref:refs/heads/main`.
- **Numeric IDs.** The subject may include GitHub's numeric owner and repository IDs, as in `daticalamy@50338582/cs_snowflake_oidc@1454462333`, not just `owner/repo`. This is why copying the real value matters.
- **Case sensitivity.** The match is exact and case-sensitive.

Using GitHub environments is a useful pattern. Each environment (DEV, QA, PROD) produces a distinct subject, so each can map to its own Snowflake service user with its own privileges.

### 1.3 Create the service user with an OIDC workload identity

Use the `sub` value from step 1.2:

```sql
CREATE USER IF NOT EXISTS LB_WIF_TEST_USER
  TYPE = SERVICE
  DEFAULT_ROLE = LB_WIF_TEST_ROLE
  DEFAULT_WAREHOUSE = LB_WIF_TEST_WH
  WORKLOAD_IDENTITY = (
    TYPE = OIDC
    ISSUER = 'https://token.actions.githubusercontent.com'
    SUBJECT = '<paste the sub claim here>'
    OIDC_AUDIENCE_LIST = ('snowflakecomputing.com')
  );

GRANT ROLE LB_WIF_TEST_ROLE TO USER LB_WIF_TEST_USER;
```

If the user already exists and you need to change the subject, for example after adding a GitHub environment:

```sql
ALTER USER LB_WIF_TEST_USER SET WORKLOAD_IDENTITY = (
  TYPE = OIDC
  ISSUER = 'https://token.actions.githubusercontent.com'
  SUBJECT = '<paste the sub claim here>'
  OIDC_AUDIENCE_LIST = ('snowflakecomputing.com')
);
```

Verify the configuration:

```sql
SHOW USER WORKLOAD IDENTITY AUTHENTICATION METHODS FOR USER LB_WIF_TEST_USER;
```

### 1.4 Get your account identifier for the JDBC URL

```sql
SELECT CURRENT_ORGANIZATION_NAME() || '-' || CURRENT_ACCOUNT_NAME() AS account_identifier;
```

You can also read it from your Snowsight URL, which has the form `https://app.snowflake.com/<org>/<account>/...`. The identifier is `<org>-<account>`.

### 1.5 (Optional) Audit WIF logins

```sql
SELECT EVENT_TIMESTAMP, USER_NAME, FIRST_AUTHENTICATION_FACTOR, IS_SUCCESS, ERROR_MESSAGE
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE USER_NAME = 'LB_WIF_TEST_USER'
ORDER BY EVENT_TIMESTAMP DESC;
```

`ACCOUNT_USAGE` views can lag by up to a couple of hours.

---

## Part 2: GitHub Actions setup

### 2.1 Runner requirements

This setup was tested on a **self-hosted Linux x64 runner**. GitHub-hosted runners also support OIDC tokens. The runner needs:

- **`bash`, `curl`, and `jq`** installed. On Windows runners, `run:` steps default to PowerShell, so use a Linux or macOS runner or set `shell: bash`.
- **Outbound HTTPS access** to `<org>-<account>.snowflakecomputing.com`. If your Snowflake account has a network policy, the runner's egress IPs must be allowed.
- **Java and Liquibase Secure**, installed by the `setup-liquibase` local action in this repo, or however you normally install Liquibase on runners.

### 2.2 Create a GitHub environment

Go to **Settings → Environments** and create an environment such as `DEV`. The environment name becomes part of the token's subject (see 1.2).

If you add protection rules, such as required reviewers, runs will pause for approval before the job starts.

### 2.3 Configure the secret and variable

| Name | Type | Where | Value |
|---|---|---|---|
| `LIQUIBASE_LICENSE_KEY` | Secret | Repository or environment | Your Liquibase Secure license key |
| `LIQUIBASE_URL` | Variable | `DEV` environment (or repository) | The JDBC URL **without** the token (below) |

`LIQUIBASE_URL` value:

```
jdbc:snowflake://<org>-<account>.snowflakecomputing.com/?authenticator=WORKLOAD_IDENTITY&workloadIdentityProvider=OIDC&db=LB_WIF_TEST&schema=PUBLIC&warehouse=LB_WIF_TEST_WH&role=LB_WIF_TEST_ROLE
```

Don't store the token in the variable. OIDC tokens are short-lived and issued per run, so the workflow fetches a fresh one and appends `&token=<token>` at runtime. No database username or password is needed. 

Within the **connect_action.yml** the LIQUIBASE URL looks like:
`LIQUIBASE_COMMAND_URL: ${{ vars.LIQUIBASE_URL }}&token=${{ env.OIDC_TOKEN }}`

### 2.4 The workflow

Key requirements for .github\workflows\connect_action.yml:

- **`permissions: id-token: write`.** Without this, GitHub won't issue an OIDC token.
- **`actions/checkout` first.** It must run before the local `setup-liquibase` action and before Liquibase reads `changelog.yaml`.
- **URL set at step level.** `LIQUIBASE_COMMAND_URL` is set on the Liquibase step, after the token has been fetched.

### 2.5 Run it

1. Go to **Actions → Liquibase Secure Connect → Run workflow**, choose the environment, and run it.
2. A successful run shows the Liquibase connection succeeding. The login also appears in Snowflake's `LOGIN_HISTORY`.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `Driver class was not specified and could not be determined from the url (&token=***` | `vars.LIQUIBASE_URL` is empty or not found | Create `LIQUIBASE_URL` as a **variable** (not a secret) in the repository or in the environment the job uses |
| Snowflake authentication failure after the URL is fixed | Token `sub` doesn't match the user's `SUBJECT` | Copy the exact `sub` from the workflow output and `ALTER USER ... SET WORKLOAD_IDENTITY` (1.3) |
| Token step returns `null` or empty | Missing `permissions: id-token: write` | Add it at the workflow or job level |
| Driver complains about a missing user | Driver requires a user for your configuration | Append `&user=LB_WIF_TEST_USER` to `LIQUIBASE_URL` |
| WIF authenticator not recognized | Old Snowflake JDBC driver | Confirm `internal/lib/snowflake-jdbc*.jar` is 3.26.0 or later |

## Security notes

- **No long-lived credentials.** Nothing like passwords, key pairs, or PATs is stored in GitHub. Each run uses a short-lived token that only this repository (and, with environments, only this environment) can obtain.
- **Masked token.** The token is masked in GitHub Actions output with `::add-mask::`.
- **Log file.** The full JDBC URL, including the token, can appear in Liquibase logs, such as `logs/log.json` at higher log levels. Don't upload log files as workflow artifacts or share them without removing the token. The token expires quickly, but treat it as sensitive.
- **Scoped access.** Scope each Snowflake service user's role to the minimum privileges Liquibase needs in that environment.

## References

- Snowflake: [Workload identity federation](https://docs.snowflake.com/en/user-guide/workload-identity-federation)
- Snowflake: [ALTER USER … ADD WORKLOAD IDENTITY](https://docs.snowflake.com/sql-reference/sql/alter-user-add-workload-identity)
- Snowflake: [JDBC driver connection parameters](https://docs.snowflake.com/en/developer-guide/jdbc/jdbc-parameters)
- Liquibase: [Connect Liquibase to Snowflake](https://docs.liquibase.com/secure/integration-guide-5-0/connect-liquibase-to-snowflake)
- Liquibase: [Liquibase parameters for Snowflake](https://docs.liquibase.com/secure/reference-guide-5-0/liquibase-parameters-for-snowflake)
