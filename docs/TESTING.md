# Testing Guide

This document describes the automated test strategy, suite layout, database isolation model, and commands for the Hotel Management backend.

## Test Strategy

The solution uses a two-level automated strategy:

1. **Unit tests** validate domain invariants, request validators, and authorization/administration rules without infrastructure.
2. **Integration tests** start each real ASP.NET Core application with `WebApplicationFactory`, use a real SQL Server database, generate real JWTs, and exercise HTTP endpoints.

The gateway has a dedicated integration suite that starts the gateway against an in-process stub downstream server and verifies routing, Swagger aggregation, health, and internal-route isolation.

There are no browser end-to-end tests. The manual end-to-end flow in the root `README.md` covers the full gateway-to-service path.

## Suite Layout

| Project | Type | Focus |
|---|---|---|
| `HotelManagement.Auth.UnitTests` | Unit | Owner-role safeguards, user request validation |
| `HotelManagement.Auth.IntegrationTests` | Integration | Login, JWT, RBAC, user administration, health, Swagger |
| `HotelManagement.Rooms.UnitTests` | Unit | Room, room type, date range, block domain rules and validators |
| `HotelManagement.Rooms.IntegrationTests` | Integration | Inventory CRUD, availability, internal key, serializable holds, occupancy |
| `HotelManagement.Guests.UnitTests` | Unit | Guest normalization, document rules, validators |
| `HotelManagement.Guests.IntegrationTests` | Integration | Guest CRUD, deactivation, internal summary, RBAC |
| `HotelManagement.Reservations.UnitTests` | Unit | Stay calculations, reservation snapshots, status transitions, validators |
| `HotelManagement.Reservations.IntegrationTests` | Integration | Create, reschedule, cancel, no-show, check-in/out, compensation paths |
| `HotelManagement.Gateway.IntegrationTests` | Integration | Routing, Swagger aggregation, health, internal-route blocking |

## Current Test Count

The full suite contains **138 test cases**:

| Project | Tests |
|---|---:|
| Auth unit | 14 |
| Auth integration | 18 |
| Rooms unit | 18 |
| Rooms integration | 19 |
| Guests unit | 15 |
| Guests integration | 9 |
| Reservations unit | 18 |
| Reservations integration | 16 |
| Gateway integration | 11 |
| **Total** | **138** |

A test case count includes each data row of a parameterized test. For example, the three anonymous-request paths in Auth are three test cases.

## Prerequisites

- .NET SDK 10.0.101 or compatible 10.0.x.
- SQL Server reachable through the configured connection strings.
- SQL login with permission to create and drop databases.
- Environment loaded:

```bash
set -a
source .env.local
set +a
```

Integration test factories resolve a connection template from:

- Auth: `HOTEL_SQL_CONNECTION` or `ConnectionStrings__AuthDb`
- Rooms: `ConnectionStrings__RoomsDb` or `HOTEL_SQL_CONNECTION`
- Guests: `ConnectionStrings__GuestsDb` or `HOTEL_SQL_CONNECTION`
- Reservations: `ConnectionStrings__ReservationsDb` or `HOTEL_SQL_CONNECTION`

The factory clones the template and replaces the database name. The original local development databases are never used by tests.

## Commands

### Full suite

```bash
set -a
source .env.local
set +a

dotnet test HotelManagement.sln
```

### One project

```bash
dotnet test tests/HotelManagement.Rooms.IntegrationTests
dotnet test tests/HotelManagement.Auth.UnitTests
dotnet test tests/HotelManagement.Gateway.IntegrationTests
```

### One test name

```bash
dotnet test tests/HotelManagement.Reservations.IntegrationTests \
  --filter "FullyQualifiedName~Reschedule_moves_the_stay_and_records_the_new_total"
```

### One class

```bash
dotnet test tests/HotelManagement.Rooms.UnitTests \
  --filter "FullyQualifiedName~DomainTests"
```

### Unit tests only

Unit tests do not need SQL:

```bash
dotnet test tests/HotelManagement.Auth.UnitTests
dotnet test tests/HotelManagement.Rooms.UnitTests
dotnet test tests/HotelManagement.Guests.UnitTests
dotnet test tests/HotelManagement.Reservations.UnitTests
```

### List tests without executing

```bash
dotnet test HotelManagement.sln --list-tests
```

### Build and format verification

```bash
dotnet build HotelManagement.sln
dotnet format HotelManagement.sln --verify-no-changes
```

## Test Database Isolation

Each integration-test factory creates a unique database per factory instance:

```text
HotelAuthTests_<32-hex-guid>
HotelRoomsTests_<32-hex-guid>
HotelGuestsTests_<32-hex-guid>
HotelReservationsTests_<32-hex-guid>
```

The isolation mechanism is:

1. Read a connection-string template from the environment.
2. Use `SqlConnectionStringBuilder` to replace `InitialCatalog` with the unique test database name.
3. Supply that connection string through in-memory configuration.
4. In `ConfigureTestServices`, remove the application's `DbContextOptions<T>` and `DbContext` registrations.
5. Re-register the `DbContext` against the test database and keep the audit interceptor.
6. Let the application's startup initializer apply migrations.
7. On disposal, clear SQL connection pools, resolve the test `DbContext`, and call `EnsureDeleted()`.

This design was introduced to prevent IIS/TestServer configuration ordering from causing tests to migrate or delete the local development databases. It is a hard requirement: tests must never resolve `HotelAuth`, `HotelRooms`, `HotelGuests`, or `HotelReservations` as their target database.

### Evidence of isolation

- Development databases persist with the same names after the suite.
- Test databases use the `*Tests_*` naming convention.
- Test disposal removes the unique databases.
- The test suite can run while local services are stopped.
- The local seeded Owner remains intact after tests.

### Stale test databases

If a test process is killed before disposal, a `Hotel*Tests_*` database can remain. Clean them with:

```bash
sqlcmd -S "localhost,${HOTEL_SQL_PORT:-1434}" -U sa -P "$HOTEL_SQL_SA_PASSWORD" -C \
  -d master -Q "
    DECLARE @sql nvarchar(max) = N'';
    SELECT @sql = @sql + N'ALTER DATABASE [' + name + N'] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [' + name + N'];'
    FROM sys.databases
    WHERE name LIKE 'Hotel%Tests[_]%';
    EXEC sp_executesql @sql;"
```

## What Each Suite Covers

### Auth unit

- Creating a user with known roles, including case-insensitive role names.
- Rejecting empty, unknown, or weak input.
- Preventing removal of one's own Owner role.
- Preventing demotion or deactivation of the last active Owner.
- Allowing Owner demotion/deactivation when another active Owner exists.
- Allowing non-owner deactivation.

### Auth integration

- Bootstrap Owner login and `/api/auth/me`.
- Anonymous access returns `401`.
- Manager and Receptionist cannot administer users.
- Owner can create, list, read, update roles, activate, and deactivate users.
- Duplicate email returns `409`.
- Unknown role and weak password return `400`.
- Invalid credentials return `401`.
- Unknown user returns `404`.
- Deactivated users cannot log in, while an already issued JWT remains valid until expiry.
- Self-deactivation and self-demotion are blocked.
- Health endpoint, Swagger exposure, and correlation ID echo.

### Rooms unit

- Room type name/description normalization and price rounding.
- Negative price and invalid capacity rejection.
- Room number normalization, floor bounds, room type requirement, status validation.
- Half-open date-range overlap semantics.
- Adjacent stays are allowed.
- Block rescheduling and invalid ranges.
- Room occupancy transitions, including refusal to occupy maintenance rooms.
- Request validator coverage.

### Rooms integration

- Anonymous access returns `401`.
- Receptionist read access and status changes; inventory writes return `403`.
- Manager CRUD for room types and rooms.
- Case-insensitive duplicate room type and room number rejection.
- Unknown room type returns `404`.
- Availability blocks remove and release rooms from availability.
- Block reschedule preserves the hold and updates dates.
- Reschedule overlap rejection leaves the original block intact.
- Internal API key required for internal reads, blocks, occupancy, and availability.
- Occupancy endpoint marks rooms `Occupied` and back to `Available`.
- Concurrent overlapping block requests produce exactly one `201` and one `409`.
- Out-of-service and maintenance rooms are unavailable and cannot receive blocks.
- Invalid date ranges and statuses return `400`.
- Health and Swagger are public; internal routes are absent from Swagger.

### Guests unit

- Name, email, and optional text normalization.
- Empty required fields rejected.
- Future date of birth and invalid document type rejected.
- Document type and number must be provided together.
- The parameterized email test covers malformed addresses.
- Validators mirror domain rules.

### Guests integration

- Anonymous access returns `401`.
- Receptionist can read but cannot manage guests.
- Manager can create, update, search, and deactivate guests.
- Duplicate email is rejected case-insensitively.
- Invalid email and mismatched document fields return `400`.
- Unknown guest returns `404`.
- Internal summary requires the API key.
- Health and Swagger are public; internal routes are hidden.

### Reservations unit

- Stay period nights calculation and invalid range rejection.
- Guest, room, and room type snapshots.
- Total calculation from nightly rate and nights.
- Occupancy and negative-rate rejection.
- Block assignment only once.
- Confirmed → CheckedIn → CheckedOut transitions.
- Confirmed → Cancelled and Confirmed → NoShow.
- Reschedule recalculates nights and total at the preserved nightly rate.
- Non-confirmed reservations cannot be rescheduled.
- Terminal statuses cannot transition again.
- Validator coverage for create, reschedule, and cancel.

### Reservations integration

- Anonymous access returns `401`.
- Receptionist can create, read, check in, and check out.
- Manager can create, search, and read.
- Inactive or unknown guests are rejected.
- Unknown, non-operational, or too-small rooms are rejected.
- Overlapping reservations are rejected.
- Cancel and no-show release the room and cannot be repeated.
- Check-out requires `CheckedIn`.
- Invalid date ranges return `400`.
- Check-in occupies the room; check-out releases it.
- Reschedule moves the stay and updates nights/total.
- Reschedule conflict leaves the original stay unchanged.
- Checked-in reservations cannot be rescheduled.
- Health and Swagger remain public.

Reservation integration tests replace `IRoomDirectory` and `IGuestDirectory` with in-memory fakes. This keeps the tests focused on reservation orchestration, local persistence, and compensation behavior without requiring live Rooms and Guests services. The real HTTP clients are covered by the manual end-to-end flow and by the downstream service's own integration tests.

### Gateway integration

- Health endpoint.
- Swagger UI and aggregated documents.
- Auth, Rooms, Room Types, Guests, and Reservations routes proxied.
- `/internal/*` routes not exposed.
- Ocelot config supplied through a temporary `OCELOT_CONFIG_PATH`.
- Downstream service simulated with an in-process stub server.

## Concurrency Testing

`Concurrent_overlapping_block_requests_result_in_a_single_hold` sends two overlapping block requests in parallel. The Rooms service uses a serializable transaction and an overlap query. The expected result is exactly:

- one `201 Created`
- one `409 Conflict`

This test protects the most important inventory invariant: a room cannot be double-held for overlapping dates.

There is no automated stress or soak test for high concurrency. The single concurrency test is a correctness check, not a throughput benchmark.

## Test Coverage Matrix

| Requirement | Unit coverage | Integration coverage |
|---|---|---|
| Login and JWT issuance | Validators | Auth login, `/me` |
| Role-based authorization | Administration rules | Manager/Receptionist/Owner role checks |
| Owner continuity | Rules tests | Self-demotion/self-deactivation tests |
| Room types and rooms | Domain + validators | CRUD and duplicate checks |
| Availability holds | Domain + validators | Create, overlap, release, concurrent hold |
| Internal API key | — | Missing/wrong key on Rooms and Guests |
| Guest lifecycle | Domain + validators | CRUD, deactivate, internal summary |
| Reservation creation | Domain + validators | Guest/room validation, overlap, RBAC |
| Reschedule | Domain + validators | Move, conflict, checked-in rejection |
| Cancel and no-show | Domain | Release, idempotency/state rejection |
| Check-in/out | Domain | Occupancy and release behavior |
| Gateway routing | — | All public routes and internal blocking |
| Health/Swagger | — | All services and gateway |

## Test Gaps and Future Work

Current automated coverage does not include:

- Load, stress, or soak testing.
- Fault injection for downstream timeouts during compensation.
- Contract tests between Reservations and Rooms/Guests clients.
- Mutation testing or a minimum coverage gate.
- Browser-based end-to-end tests.
- Multi-instance migration or startup-race tests.
- Security scanning and dependency-vulnerability gates.
- Tests against a production-like TLS/network topology.

These gaps are documented in the LLD risk section. They should be addressed before production rollout, especially end-to-end compensation failure tests and load testing of availability contention.

## Best Practices for New Tests

1. Add domain invariants to unit tests first; they are fast and deterministic.
2. Add endpoint behavior to the owning service's integration suite.
3. Never share a development database with tests.
4. Keep the unique-database factory pattern.
5. Use fixed dates far enough in the future to avoid clock coupling.
6. Generate unique room numbers, emails, and names when tests share a factory.
7. Assert status codes and response bodies, not only success.
8. Add a gateway test when adding a new public route.
9. Add an internal-key test when adding an internal route.
10. Run the full suite and `dotnet format --verify-no-changes` before merging.
