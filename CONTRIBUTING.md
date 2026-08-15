# Contributing to BankGate

BankGate is a personal portfolio project, but it's built to the same standards as a real team codebase — so if you're exploring the code, reproducing the setup, or suggesting a change, here's how it's organized and what's expected.

## Project structure

Four independently deployable Mule applications, each with its own `pom.xml`, `mule-artifact.json`, and flow files:

```
studio-workspace/
├── experience-api/      # Public tier
├── process-api/         # Orchestration tier
├── system-api/          # System tier (owns the DB connection)
├── auth-service-api/    # OAuth 2.0 token service
├── docs/screenshots/    # README proof-of-work images
├── API_REFERENCE.md
└── README.md
```

## Local setup

See the [README's Setup section](README.md#setup) for cloning, configuring credentials, and running each app.

## Making a change

1. **Never commit real credentials.** `config.properties` and `application.properties` are gitignored on purpose — only their `.example` counterparts (with placeholder values) should ever be committed. Check `git status` before every commit if you've touched a config file.
2. **Keep the error envelope consistent.** Any new failure path should raise a `BANK-XXXX`-style error and route through `global-error-response-subflow`, not a one-off response shape.
3. **Write a MUnit test for new flow logic.** Follow the existing patterns in `system-api/src/test/munit/`:
   - Use `munit-tools:` namespace for `mock-when`, `with-attributes`, `then-return` — not `munit:`
   - For happy-path assertions, use the `flowSucceeded` boolean-variable pattern rather than asserting directly on `payload.status`
   - For error-path tests, wrap the `flow-ref` in a Core `try`/`error-handler`/`on-error-continue` block and assert on the caught error type
4. **Update `API_REFERENCE.md` and the RAML files** if you add or change an endpoint's contract.
5. **Run the full MUnit suite** before committing — `mvn test` inside the relevant app folder — and confirm `Errors: 0, Failures: 0`.

## Reporting an issue

Since this is a personal project, the most useful thing to include if you spot something broken:
- Which app/flow it's in
- Whether it reproduces via Postman, MUnit, or both
- The actual error response body (with any real data redacted)

## Known limitations

See [README — Assumptions and Limitations](README.md#assumptions-and-limitations) for what's intentionally out of scope or not yet implemented (retry/resilience, audit trail, KYC→Account business rules, sustained standalone runtime operation on Windows).
