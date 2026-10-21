# SQL Injection: Defensive DVWA Case Study

A privacy-sanitized, defensive case study of SQL injection concepts observed in a deliberately vulnerable, isolated training application.

> This repository is for secure-coding education and remediation. It does not publish exploit strings, retrieved account data, password hashes, credentials, or instructions for extracting or cracking them. The accompanying image is a cropped contextual illustration, not proof of a successful test.

## Summary

The source exercise used Damn Vulnerable Web Application (DVWA) to study how unsafe SQL query construction can let untrusted input change query behavior. In a deliberately weak training configuration, a user lookup exposed more records than intended; the exercise also highlighted the risk of storing passwords with obsolete, fast hashes.

The issue is in how an application constructs and executes database queries—not in SQL itself. Similar bugs in real applications can expose sensitive data or enable unauthorized changes. The lab results are specific to the training setup and should not be treated as a severity assessment for another system.

## Scope and privacy

- The source material described an intentionally vulnerable, locally hosted training application.
- Personal identifiers, account names, hostnames, addresses, timestamps, environment fingerprints, and third-party references to individuals have been excluded.
- Injection inputs, extracted records, password hashes, plaintext passwords, and attack or cracking procedures have been excluded.
- The original screenshot is not included. A small crop showing only the application's SQL Injection heading is provided for context.
- No real system was tested or is authorized by this report.

## Security lessons

1. Build queries with parameterized statements so input is always data, never SQL syntax.
2. Validate input against the expected type and business rules. Validation supplements parameterization; it does not replace it.
3. Use a database account with only the permissions the application needs, and avoid exposing sensitive columns to unrelated code paths.
4. Store passwords with a password-hashing function designed for that purpose, such as Argon2id or bcrypt. Do not use general-purpose hashes such as MD5.
5. Return generic errors to users and keep detailed diagnostics in access-controlled logs.
6. Verify the fix with code review and automated tests that confirm unusual input cannot change the query's structure or reveal unrelated records.

## Safer query pattern

Use parameters supported by the database driver:

```php
$stmt = $pdo->prepare(
    'SELECT first_name, last_name FROM users WHERE user_id = :id'
);
$stmt->execute(['id' => $id]);
$user = $stmt->fetch();
```

If the identifier must be an integer, validate that requirement separately and handle invalid input using the application's normal error path. Do not interpolate user input into the SQL string.

## Verification checklist

- [ ] All data values in SQL statements use driver-supported parameters.
- [ ] Invalid identifiers are rejected according to application requirements.
- [ ] The database account follows least privilege.
- [ ] Passwords use a modern password-hashing function with appropriate parameters.
- [ ] Errors do not disclose SQL statements, schema details, or database diagnostics to end users.
- [ ] Tests confirm that malformed input cannot broaden a query or expose unrelated records.

## References

- [OWASP: SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP Cheat Sheet Series: SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Cheat Sheet Series: Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [CWE-89: Improper Neutralization of Special Elements used in an SQL Command](https://cwe.mitre.org/data/definitions/89.html)
- [DVWA project](https://github.com/digininja/DVWA)

## Image

[`evidence/dvwa-sqli-context.png`](evidence/dvwa-sqli-context.png) is a cropped contextual image. It intentionally excludes the browser address bar, submitted input, displayed records, and account information.
