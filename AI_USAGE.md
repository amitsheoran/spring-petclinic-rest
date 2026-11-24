# AI Usage and Independent Verification Report

## 1. AI Tools and Models Used
* **Tool / Environment:** Google Antigravity CLI
* **Model:** Gemini 3.8 Flash (Medium)
* **Scope of Delegation:** Architecture exploration, risk ranking, test fixture generation, and input validation constraint analysis.

## 2. Meaningful Work Delegated to AI
* **Codebase Exploration:** Tasked the AI with analyzing `OwnerRestControllerV1.java`, `openapi.yml`, and `OwnerRestControllerV1Tests.java` to discover gaps in negative input validation coverage.
* **Risk Identification & Ranking:** Delegated generating 4 quality risks concerning HTTP boundary validation vs. persistence layer constraints and ranking them by impact vs. verification effort.
* **Test Fixture Drafting:** Delegated scaffolding the MockMvc negative test request structures for `POST /api/owners` verifying `status().isBadRequest()` on malformed payloads.

## 3. Independent Verification Strategy
* Did not rely on AI claims or assume generated tests would compile and pass.
* Verified independently on the local machine by executing:
  ```bash
  # Test compilation check:
  .\mvnw.cmd test-compile -DskipTests

  # Targeted test suite execution:
  .\mvnw.cmd test -Dtest=OwnerRestControllerV1Tests
  ```
* Confirmed the test results independently from Maven logs:
  `[INFO] Tests run: 31, Failures: 0, Errors: 0, Skipped: 0`
  `[INFO] BUILD SUCCESS`
* Verified that HTTP status assertions (`isBadRequest()`) accurately match Spring Boot's `MethodArgumentNotValidException` mapped via `ExceptionControllerAdvice`.

## 4. Rejected or Changed AI Suggestion (Critical Review)
* **What was considered / suggested:**
  Applying a strict international E.164 regular expression (`^\+?[1-9]\d{1,14}$`) to the telephone field validation.
* **Why it was independently rejected:**
  1. **Violates Contract-First Architecture:** DTO classes (`OwnerDto`, `OwnerFieldsDto`) reside in `target/generated-sources/openapi/` and are generated from `src/main/resources/openapi.yml`. Directly modifying Java DTOs is an anti-pattern as changes get wiped during `mvn clean compile`.
  2. **Persistence Constraint Conflict:** The underlying JPA entity `Owner.java` enforces `@Digits(fraction = 0, integer = 10)`. The `+` character is rejected by `@Digits`, which would cause an unhandled JPA `ConstraintViolationException` (HTTP 500) during `clinicService.saveOwner()`.
  3. **Seed Data Incompatibility:** Sample data files across all database engines (`src/main/resources/db/*/data.sql`) use 10-digit plain numeric strings (e.g., `'6085551023'`). Enforcing E.164 with mandatory international prefixes would break existing database records upon update.
* **Action Taken:**
  Rejected the custom E.164 regex. Instead, focused engineering effort on adding regression tests verifying that the existing contract (`pattern: '^[0-9]*$'`, `@Size(max = 20)`) is strictly enforced at the HTTP boundary.

## 5. Items Requiring Further Verification Before Production Merge
* **Client-Side UX Validation:** Verify how the Angular and React front-end clients render the `ProblemDetail` RFC 7807 error payload when HTTP 400 is returned.
* **Database Constraint Synchronization:** Align the OpenAPI specification `maxLength: 20` with the entity-level `@Digits(integer = 10)` constraint in a future database schema migration to avoid 11-to-20 digit payloads passing the REST boundary but failing at the database level.
