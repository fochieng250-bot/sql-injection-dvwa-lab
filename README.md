# SQL Injection: DVWA Defensive Case Study

A privacy-sanitized report on a SQL injection lesson in Damn Vulnerable Web Application (DVWA), with findings, impact, limitations, and remediation.

> **Purpose and limits:** This is a defensive training write-up. It preserves the original exercise's key observations without publishing reusable injection strings, extraction instructions, real account data, password hashes, or credentials. Numeric identifiers below are randomized examples, not original evidence.

## Executive summary

The exercise examined a user-lookup feature in an intentionally vulnerable DVWA instance running at the application's low security setting. A normal lookup displayed one record. A controlled lab test showed that unsafe handling of the lookup value could change query behavior and expose five records instead of one.

The exercise also documented a second risk: password-hash fields could be exposed through the vulnerable lookup path. The original material described the stored format as MD5, which is unsuitable for password storage. Actual hashes and passwords are intentionally omitted here.

This demonstrates two related but distinct control failures:

1. **SQL injection:** untrusted input can affect the structure or logic of a database query when queries are assembled unsafely.
2. **Excessive data exposure:** the application/database path made account information available beyond the intended single-record lookup.

The findings apply to the deliberately vulnerable lab only. They do not establish that another system is vulnerable or support a severity rating for any real service.

## Lab scope

| Item | Sanitized description |
|---|---|
| Application | Damn Vulnerable Web Application (DVWA), an intentionally vulnerable training app |
| Security setting | Low |
| Deployment | Isolated local container lab; loopback web service on port `8080` |
| Application stack | Web server, PHP, and MariaDB-compatible database |
| Normal lookup | One record returned for the ordinary test case |
| Altered-query observation | Five records displayed in the lab |
| User/account values | Excluded; all identifying and credential values removed |

No public or third-party target was tested. Personal names, handles, hostnames, network addresses, container identifiers, machine details, and mentoring or employment references from the source notes are not reproduced.

## Lab setup and troubleshooting notes

The training app was launched in a local container, its database was initialized through the app's setup page, and the exercise was performed at the intentionally weak security setting. The exact machine fingerprint, container ID, login details, and shell history are excluded.

During setup, the database service initially failed to start because the lab machine had insufficient free disk space. Reviewing the container's service logs identified the storage problem. The lab operator reclaimed space using routine container and package-cache maintenance, restarted the service, and confirmed that the database initialized successfully.

This setup issue is an operational lesson rather than an application vulnerability: inspect service logs, check available storage, make a bounded cleanup, and verify the affected service before resuming testing.

## Findings

### V-07 — User lookup returned more records than intended

| Attribute | Summary |
|---|---|
| Category | SQL injection, CWE-89 |
| Component | DVWA user lookup |
| Expected behavior | A single requested record is returned |
| Observed behavior | A lab input altered query logic; five records were displayed |
| Data handling | Original names and account details removed |
| Risk | Depending on the application's data and permissions, unauthorized query broadening can expose records outside the user's intended scope |

To illustrate the output shape without repeating the source data, the records can be represented by randomized placeholders:

| Example row | Synthetic record ID | Displayed fields |
|---:|---:|---|
| 1 | `482731` | first-name field, surname field |
| 2 | `915204` | first-name field, surname field |
| 3 | `367810` | first-name field, surname field |
| 4 | `704529` | first-name field, surname field |
| 5 | `128643` | first-name field, surname field |

These IDs are invented for this report. They do not correspond to the accounts or identifiers in the original notes.

### V-07b — Password-hash fields were exposed

| Attribute | Summary |
|---|---|
| Category | Sensitive data exposure associated with SQL injection |
| Component | DVWA user lookup and database query permissions |
| Observed behavior | The lab exercise reported that a query result could include user and password-hash columns for five accounts |
| Stored format described in source notes | MD5 |
| Data handling | Usernames, actual hash values, and passwords are not included |
| Risk | Fast general-purpose hashes can be guessed offline if disclosed; password reuse may increase consequences beyond the lab |

The evidence is described at the finding level only. There are no real or synthetic credential hashes in this repository, and no cracking procedure is provided.

### V-07c — Login-bypass checks did not succeed

The source notes also recorded three unsuccessful attempts against DVWA's separate login form. The report's conclusion was that the login flow did not exhibit the same behavior in the tested configuration. This negative result is retained to distinguish the affected lookup feature from the separately tested authentication flow.

This is not proof that other login versions, configurations, or routes are safe. Each relevant code path should be assessed and tested independently.

## Severity and assessment caveat

The source notes labeled the two positive findings “Critical” and quoted CVSS 9.8. That score is not repeated as a validated rating: the original vector and score were inconsistent, and a lab result cannot by itself establish the exposure, privileges, or impact of a real deployment. Assess severity only after confirming the affected system, reachable attack surface, prerequisites, data sensitivity, and integrity/availability impact.

## Root cause

SQL injection occurs when an application lets input influence SQL syntax rather than binding that input as a value. Escaping and input filtering alone are fragile defenses: they vary across database engines and contexts and can be implemented inconsistently.

The impact can be amplified when:

- the application database account has broader permissions than required;
- a lookup query can access authentication or other sensitive columns;
- database errors or query details are returned to users; or
- passwords use a fast, general-purpose hash rather than a password-hashing function.

## Remediation

### 1. Parameterize all data values

Use the database driver's parameter-binding API. Keep the SQL statement separate from user-supplied values:

```php
$stmt = $pdo->prepare(
    'SELECT first_name, last_name FROM users WHERE user_id = :id'
);
$stmt->execute(['id' => $id]);
$user = $stmt->fetch();
```

Do not concatenate request values into SQL text. Validate the identifier's expected type and application constraints separately; validation supplements, but does not replace, parameterization.

### 2. Limit database permissions

- Give each application component only the database permissions it needs.
- Avoid granting a user-lookup feature access to password-hash fields.
- Separate read and write roles when practical.
- Review permissions periodically and test with the actual deployed database account.

### 3. Protect stored passwords

- Replace MD5 and other general-purpose hashes with a dedicated password-hashing function such as Argon2id or bcrypt.
- Use the library's current recommended work factor and migration strategy.
- Never log, publish, or return password hashes or plaintext passwords.
- If hashes may have been exposed, follow the incident-response process: assess scope, invalidate affected credentials as appropriate, and notify responsible owners.

### 4. Handle errors and verify fixes

- Return generic errors to end users; keep necessary diagnostics in access-controlled logs.
- Add regression tests for expected input, invalid input, and attempts to change query behavior.
- Verify that a lookup returns only the authorized record and cannot return unrelated columns.
- Re-test each relevant route and security configuration after remediation.

## Safe verification checklist

- [ ] Query values are bound through prepared statements.
- [ ] Identifiers are validated against the application's type and business rules.
- [ ] Queries return only the fields needed for the feature.
- [ ] The database account has least-privilege permissions.
- [ ] Passwords use a dedicated password-hashing function.
- [ ] User-visible errors do not expose SQL or database internals.
- [ ] Regression tests confirm that unexpected input cannot broaden results.
- [ ] The fix is verified in a controlled, authorized test environment.

## Evidence and privacy

The original source contained a screenshot showing a local DVWA page and returned demo records. The original image and source document are not included. [`evidence/dvwa-sqli-context.png`](evidence/dvwa-sqli-context.png) is a small, cropped illustration showing only the application heading; it is not evidence of the query result.

The row counts above preserve the reported scale of the lab observation. The example numeric IDs are randomly generated placeholders. Account names, actual hashes, plaintext passwords, attack inputs, and machine-identifying details have been removed rather than replaced with values that could be mistaken for real credentials or evidence.

## References

- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP Cheat Sheet Series: SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Cheat Sheet Series: Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [CWE-89: Improper Neutralization of Special Elements used in an SQL Command](https://cwe.mitre.org/data/definitions/89.html)
- [Damn Vulnerable Web Application (DVWA)](https://github.com/digininja/DVWA)
