# Quality Engineering Report: Spring PetClinic REST

## 1. Scope & System Understanding
* **Target Area:** `OwnerRestControllerV1` (`POST /api/owners` and `PUT /api/owners/{ownerId}`) and related DTO validation (`OwnerDto`, `OwnerFieldsDto`).
* **System Boundaries:** The REST API acts as the public boundary for client applications. In this architecture, OpenAPI 3.0 specifications (`openapi.yml`) generate server contract interfaces and DTOs with Jakarta Bean Validation annotations (`@NotNull`, `@Pattern`, `@Size`). The controller is expected to enforce these validation constraints at the HTTP boundary and reject invalid payloads with `400 Bad Request` before invoking internal business services.

## 2. Risks Considered and Ranked
1. **[Rank 1 - Selected] Incomplete Negative Input Validation on Owner Endpoints:** While positive paths and `null` names were tested, edge-case invalid payloads—specifically non-numeric telephone values and blank string names—were not explicitly verified against the REST boundary to ensure consistent HTTP 400 rejection.
2. **[Rank 2] Contract vs. Persistence Divergence on Telephone Length:** In `openapi.yml`, telephone is defined with `maxLength: 20` and `pattern: '^[0-9]*$'`. However, the JPA entity `Owner.java` uses `@Digits(fraction = 0, integer = 10)`. Inputs exceeding 10 digits could cause JPA ConstraintViolationException (HTTP 500) instead of clean HTTP 400.
3. **[Rank 3] ID Path Variable vs. Payload Mismatch in PUT Operations:** During `PUT /api/owners/{ownerId}`, if a client supplies an ID in the request body that contradicts the URL path variable, potential state inconsistencies could arise.
4. **[Rank 4] Error Payload Schema Consistency:** Ensuring that all validation failures return standard RFC 7807 `ProblemDetail` structures conforming to the OpenAPI contract.

*Rationale for Rank 1:* Input validation is the primary line of defense protecting downstream services and the database from malformed data and unexpected 500 runtime errors. It provides the highest immediate quality gain within the 3-hour timebox.

## 3. Changes and Rationale
* **Added Negative Validation Tests:** Implemented targeted tests in `OwnerRestControllerV1Tests.java`:
  * `shouldReturnBadRequestWhenAddOwnerWithInvalidTelephone`: Verifies that submitting an alphanumeric telephone string (e.g., `"invalid-phone"`) fails pattern validation and results in HTTP `400 Bad Request`.
  * `shouldReturnBadRequestWhenAddOwnerWithBlankNames`: Verifies that empty strings for `firstName` fail `@Size(min = 1)` and pattern checks, returning HTTP `400 Bad Request`.
* **Rationale:** Establishes regression guards proving that OpenAPI-generated Bean Validation annotations (`@Pattern`, `@Size`) are actively enforced by Spring's MockMvc pipeline.

## 4. Reproducible Verification
Run the following single command from the project root:

```bash
# Windows (PowerShell / Command Prompt):
.\mvnw.cmd test -Dtest=OwnerRestControllerV1Tests

# Linux / macOS / Git Bash:
./mvnw test -Dtest=OwnerRestControllerV1Tests
```

**Actual Result:**
```text
[INFO] Tests run: 31, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 20.57 s -- in org.springframework.samples.petclinic.rest.controller.OwnerRestControllerV1Tests
[INFO] BUILD SUCCESS
[INFO] Total time: 34.207 s
```

## 5. Confidence Limits
* **What the evidence proves:** Confirms that `OwnerRestControllerV1` properly enforces HTTP 400 rejection for malformed telephone numbers and blank names when payloads arrive at the REST boundary.
* **What the evidence does NOT prove:** Does not cover other REST controllers (`PetRestControllerV1`, `VetRestControllerV1`), nor does it verify database constraint enforcement under concurrent multi-user load.

## 6. Work Left Out
* Expanding validation tests to Pet and Visit controllers.
* Adding contract verification tests (e.g., using Spectral or OpenAPI Validator) to ensure `openapi.yml` never drifts from runtime behavior.

## 7. Next Steps (With One More Day)
* Set up automated schema linting in the Maven build to validate `openapi.yml` changes against RFC and domain standards.
* Implement end-to-end integration tests using Testcontainers against PostgreSQL and MySQL to verify persistence-layer behavior with boundary values.

## 8. Time Spent
* Approximate counted time: **2 hours 15 minutes** (Architecture review & OpenAPI schema analysis: 40m, Test design & implementation: 45m, Verification & documentation: 50m).
