# Hotel Management System

Backend-only hotel management platform built with .NET 10, ASP.NET Core, SQL Server, EF Core, JWT authentication, role-based authorization, and an Ocelot API gateway.

The solution is split into four independently deployable business services and one gateway:

| Component | Port | Database | Responsibility |
|---|---:|---|---|
| Gateway | 5110 | None | Public entry point, routing, aggregated Swagger |
| Auth API | 5111 | `HotelAuth` | Login, JWT issuance, users, roles |
| Rooms API | 5112 | `HotelRooms` | Room types, rooms, availability blocks, occupancy |
| Guests API | 5113 | `HotelGuests` | Guest profiles and active/inactive lifecycle |
| Reservations API | 5114 | `HotelReservations` | Reservations, check-in/out, cancel, no-show, reschedule |

Public traffic enters through the gateway. The Reservations service also calls the Rooms and Guests services over private `/internal/*` endpoints protected by a shared internal API key.

## Documentation

- [Low-level design](docs/LLD.md)
- [API reference](docs/API.md)
- [Testing guide](docs/TESTING.md)
- [Documentation index](docs/README.md)

## Technology Stack

- .NET SDK `10.0.101`
- ASP.NET Core `net10.0`
- Entity Framework Core `10.0.1`
- SQL Server (containerized for local development)
- ASP.NET Core Identity
- JWT bearer authentication
- FluentValidation `12.2.0`
- Ocelot `25.0.2`
- MMLib.SwaggerForOcelot `11.0.0`
- Swashbuckle.AspNetCore `10.3.1`
- xUnit `2.9.3`, Microsoft.AspNetCore.Mvc.Testing `10.0.1`

## Repository Layout

```text
.
├── src/
│   ├── BuildingBlocks/
│   │   ├── HotelManagement.SharedKernel/
│   │   └── HotelManagement.SharedInfrastructure/
│   ├── Gateway/HotelManagement.Gateway.Api/
│   └── Services/
│       ├── Auth/
│       ├── Rooms/
│       ├── Guests/
│       └── Reservations/
├── tests/
│   ├── HotelManagement.Auth.UnitTests/
│   ├── HotelManagement.Auth.IntegrationTests/
│   ├── HotelManagement.Rooms.UnitTests/
│   ├── HotelManagement.Rooms.IntegrationTests/
│   ├── HotelManagement.Guests.UnitTests/
│   ├── HotelManagement.Guests.IntegrationTests/
│   ├── HotelManagement.Reservations.UnitTests/
│   ├── HotelManagement.Reservations.IntegrationTests/
│   └── HotelManagement.Gateway.IntegrationTests/
├── docs/
├── .env.example
├── Directory.Build.props
├── global.json
└── HotelManagement.sln
```

Each service follows the same four-project structure:

```text
HotelManagement.<Service>.Domain
HotelManagement.<Service>.Application
HotelManagement.<Service>.Infrastructure
HotelManagement.<Service>.Api
```

## Prerequisites

Install the following:

1. .NET SDK `10.0.101` or a compatible `10.0.x` feature band.
2. Docker Desktop or another Docker runtime.
3. `dotnet-ef` 10.x:

   ```bash
   dotnet tool install --global dotnet-ef --version 10.0.5
   ```

4. Optional: `sqlcmd` for database checks and cleanup.
5. `openssl` for generating secrets.
6. `python3` for the curl examples that extract token and entity IDs.

Verify the SDK:

```bash
dotnet --version
dotnet ef --version
```

## Local Configuration

Copy the example environment file:

```bash
cp .env.example .env.local
```

Edit `.env.local` and replace every placeholder. At minimum, set:

- A strong SQL Server SA password.
- A random JWT signing key of at least 32 bytes.
- A random internal API key of at least 32 bytes.
- The bootstrap Owner email and password.
- The four SQL Server connection strings.

Generate suitable secrets with:

```bash
openssl rand -base64 48
```

The application reads configuration from environment variables, so nested settings use double underscores:

| Environment variable | Purpose |
|---|---|
| `ConnectionStrings__AuthDb` | Auth database connection |
| `ConnectionStrings__RoomsDb` | Rooms database connection |
| `ConnectionStrings__GuestsDb` | Guests database connection |
| `ConnectionStrings__ReservationsDb` | Reservations database connection |
| `Jwt__Issuer` | JWT issuer |
| `Jwt__Audience` | JWT audience |
| `Jwt__SigningKey` | HMAC signing key, at least 32 bytes |
| `Jwt__ExpiryMinutes` | Access-token lifetime, default `60` |
| `Bootstrap__OwnerEmail` | Bootstrap Owner email |
| `Bootstrap__OwnerPassword` | Bootstrap Owner password |
| `InternalApi__Key` | Shared key for `/internal/*` endpoints |
| `Services__RoomsBaseUrl` | Rooms base URL used by Reservations |
| `Services__GuestsBaseUrl` | Guests base URL used by Reservations |
| `Services__TimeoutSeconds` | Downstream HTTP timeout, default `15` |

Load the file into the current shell before any migration, build, run, or test command:

```bash
set -a
source .env.local
set +a
```

`.env.local` is ignored by Git. Never commit real secrets.

## Start SQL Server

If a container named `hmnext-sql` does not already exist:

```bash
set -a
source .env.local
set +a

docker run \
  --name hmnext-sql \
  -e "ACCEPT_EULA=Y" \
  -e "MSSQL_SA_PASSWORD=${HOTEL_SQL_SA_PASSWORD}" \
  -p "${HOTEL_SQL_PORT:-1434}:1433" \
  -d mcr.microsoft.com/mssql/server:2022-latest
```

If it already exists, start it:

```bash
docker start hmnext-sql
```

Wait until SQL Server is ready, then verify connectivity:

```bash
sqlcmd \
  -S "localhost,${HOTEL_SQL_PORT:-1434}" \
  -U sa \
  -P "$HOTEL_SQL_SA_PASSWORD" \
  -C \
  -Q "SELECT @@VERSION;"
```

The services also run `Database.MigrateAsync()` at startup. You can apply migrations automatically by starting the APIs, or apply them explicitly with the commands below.

## Apply Migrations Manually

Load the environment first:

```bash
set -a
source .env.local
set +a
```

Apply each service migration:

```bash
dotnet ef database update \
  --project src/Services/Auth/HotelManagement.Auth.Infrastructure \
  --startup-project src/Services/Auth/HotelManagement.Auth.Api

dotnet ef database update \
  --project src/Services/Rooms/HotelManagement.Rooms.Infrastructure \
  --startup-project src/Services/Rooms/HotelManagement.Rooms.Api

dotnet ef database update \
  --project src/Services/Guests/HotelManagement.Guests.Infrastructure \
  --startup-project src/Services/Guests/HotelManagement.Guests.Api

dotnet ef database update \
  --project src/Services/Reservations/HotelManagement.Reservations.Infrastructure \
  --startup-project src/Services/Reservations/HotelManagement.Reservations.Api
```

Verify the databases:

```bash
sqlcmd \
  -S "localhost,${HOTEL_SQL_PORT:-1434}" \
  -U sa \
  -P "$HOTEL_SQL_SA_PASSWORD" \
  -C \
  -d master \
  -Q "SELECT name FROM sys.databases WHERE name LIKE 'Hotel%' ORDER BY name;"
```

Expected databases:

```text
HotelAuth
HotelGuests
HotelReservations
HotelRooms
```

## Run the Full Stack Manually

Use five terminals. In every terminal, load `.env.local` first:

```bash
set -a
source .env.local
set +a
```

### Terminal 1 — Auth API

```bash
ASPNETCORE_URLS=http://localhost:5111 \
dotnet run --project src/Services/Auth/HotelManagement.Auth.Api
```

### Terminal 2 — Rooms API

```bash
ASPNETCORE_URLS=http://localhost:5112 \
dotnet run --project src/Services/Rooms/HotelManagement.Rooms.Api
```

### Terminal 3 — Guests API

```bash
ASPNETCORE_URLS=http://localhost:5113 \
dotnet run --project src/Services/Guests/HotelManagement.Guests.Api
```

### Terminal 4 — Reservations API

```bash
ASPNETCORE_URLS=http://localhost:5114 \
dotnet run --project src/Services/Reservations/HotelManagement.Reservations.Api
```

### Terminal 5 — Gateway

```bash
ASPNETCORE_URLS=http://localhost:5110 \
dotnet run --project src/Gateway/HotelManagement.Gateway.Api
```

The gateway reads `ocelot.json` from its content root. To use another file:

```bash
OCELOT_CONFIG_PATH=/absolute/path/to/ocelot.json \
ASPNETCORE_URLS=http://localhost:5110 \
dotnet run --project src/Gateway/HotelManagement.Gateway.Api
```

## Verify Startup

Check all health endpoints:

```bash
for port in 5110 5111 5112 5113 5114; do
  printf "%s=" "$port"
  curl -sS -o /dev/null -w "%{http_code}\n" "http://localhost:${port}/health"
done
```

Expected:

```text
5110=200
5111=200
5112=200
5113=200
5114=200
```

Swagger is available in `Development`:

| API | URL |
|---|---|
| Aggregated gateway Swagger UI | `http://localhost:5110/swagger` |
| Auth | `http://localhost:5111/swagger` |
| Rooms | `http://localhost:5112/swagger` |
| Guests | `http://localhost:5113/swagger` |
| Reservations | `http://localhost:5114/swagger` |

OpenAPI documents:

```text
http://localhost:5110/swagger/docs/v1/auth
http://localhost:5110/swagger/docs/v1/rooms
http://localhost:5110/swagger/docs/v1/guests
http://localhost:5110/swagger/docs/v1/reservations
```

## Manual End-to-End Test Through the Gateway

The following flow uses only the gateway.

### 1. Log in as the bootstrap Owner

```bash
TOKEN=$(curl -sS -X POST http://localhost:5110/api/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"$Bootstrap__OwnerEmail\",\"password\":\"$Bootstrap__OwnerPassword\"}" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['accessToken'])")

printf "token length=%s\n" "${#TOKEN}"
```

### 2. Read the current user

```bash
curl -sS http://localhost:5110/api/auth/me \
  -H "Authorization: Bearer $TOKEN"
```

Expected: HTTP 200 with the Owner identity and the `Owner` role.

### 3. Create a room type

```bash
TYPE_ID=$(curl -sS -X POST http://localhost:5110/api/room-types \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Manual Test Deluxe",
    "description": "Created during manual verification",
    "basePrice": 125.50,
    "maxAdults": 2,
    "maxChildren": 1
  }' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")

printf "room type=%s\n" "$TYPE_ID"
```

### 4. Create a room

```bash
ROOM_ID=$(curl -sS -X POST http://localhost:5110/api/rooms \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"roomNumber\": \"MANUAL101\",
    \"floor\": 1,
    \"roomTypeId\": \"$TYPE_ID\",
    \"status\": \"Available\",
    \"notes\": \"Manual verification room\"
  }" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")

printf "room=%s\n" "$ROOM_ID"
```

### 5. Create a guest

```bash
GUEST_ID=$(curl -sS -X POST http://localhost:5110/api/guests \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "firstName": "Manual",
    "lastName": "Tester",
    "email": "manual.tester@example.com",
    "phone": "+1 555 0101",
    "dateOfBirth": "1990-05-15",
    "addressLine": "1 Test Street",
    "city": "Testville",
    "country": "Testland",
    "nationality": "Testlish",
    "identityDocumentType": "Passport",
    "identityDocumentNumber": "TEST-PASSPORT-001",
    "notes": "Manual verification guest"
  }' \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")

printf "guest=%s\n" "$GUEST_ID"
```

### 6. Create a reservation

```bash
RES_ID=$(curl -sS -X POST http://localhost:5110/api/reservations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"guestId\": \"$GUEST_ID\",
    \"roomId\": \"$ROOM_ID\",
    \"checkIn\": \"2027-08-10\",
    \"checkOut\": \"2027-08-12\",
    \"adults\": 2,
    \"children\": 0,
    \"specialRequests\": \"High floor\"
  }" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")

printf "reservation=%s\n" "$RES_ID"
```

Expected: HTTP 201; status `Confirmed`; `nights` = 2; `totalAmount` = `251.00`.

### 7. Verify the room is held for those dates

```bash
curl -sS \
  "http://localhost:5110/api/rooms/available?checkIn=2027-08-10&checkOut=2027-08-12" \
  -H "Authorization: Bearer $TOKEN"
```

Expected: `MANUAL101` is absent.

### 8. Reschedule the reservation

```bash
curl -sS -X PUT "http://localhost:5110/api/reservations/$RES_ID/reschedule" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "checkIn": "2027-08-20",
    "checkOut": "2027-08-23"
  }'
```

Expected: HTTP 200, 3 nights, total `376.50`, and the old dates become available while the new dates are held.

### 9. Check in

```bash
curl -sS -X POST "http://localhost:5110/api/reservations/$RES_ID/check-in" \
  -H "Authorization: Bearer $TOKEN"
```

Expected: HTTP 200 and status `CheckedIn`. The room status becomes `Occupied`.

To verify the internal room state directly:

```bash
curl -sS "http://localhost:5112/internal/rooms/$ROOM_ID" \
  -H "X-Internal-Api-Key: $InternalApi__Key"
```

### 10. Check out

```bash
curl -sS -X POST "http://localhost:5110/api/reservations/$RES_ID/check-out" \
  -H "Authorization: Bearer $TOKEN"
```

Expected: HTTP 200 and status `CheckedOut`. The availability block is released, and the room returns to `Available`.

### 11. Cancel or no-show an alternative reservation

Create another confirmed reservation on available dates, then either cancel it or mark it no-show:

```bash
OTHER_RES_ID=$(curl -sS -X POST "http://localhost:5110/api/reservations" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"guestId\": \"$GUEST_ID\",
    \"roomId\": \"$ROOM_ID\",
    \"checkIn\": \"2027-09-10\",
    \"checkOut\": \"2027-09-12\",
    \"adults\": 2,
    \"children\": 0
  }" \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['id'])")

printf "alternative reservation=%s\n" "$OTHER_RES_ID"
```

Cancel it:

```bash
curl -sS -X PUT "http://localhost:5110/api/reservations/$OTHER_RES_ID/cancel" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"reason":"Manual test cancellation"}'
```

or:

```bash
curl -sS -X POST "http://localhost:5110/api/reservations/$OTHER_RES_ID/no-show" \
  -H "Authorization: Bearer $TOKEN"
```

Both release the room hold.

### 12. Verify internal routes are not public

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  "http://localhost:5110/internal/rooms/$ROOM_ID"
```

Expected: `404` or another non-success response. Internal routes are only available directly on the service and require `X-Internal-Api-Key`.

## Role-Based Authorization Checks

The bootstrap Owner can create Manager and Receptionist users:

```bash
curl -sS -X POST http://localhost:5110/api/auth/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "email": "receptionist@example.com",
    "displayName": "Front Desk",
    "password": "StrongPassword123!",
    "roles": ["Receptionist"]
  }'
```

Expected permission model:

| Operation | Owner | Manager | Receptionist |
|---|:---:|:---:|:---:|
| Manage users and roles | Yes | No | No |
| Manage room types and rooms | Yes | Yes | No |
| Change room status | Yes | Yes | Yes |
| Manage guests | Yes | Yes | No |
| Read guests | Yes | Yes | Yes |
| Create/read/cancel/check-in/out reservations | Yes | Yes | Yes |
| Reschedule reservations | Yes | Yes | Yes |
| Access `/internal/*` | Requires internal key | Requires internal key | Requires internal key |

## Run Tests

Integration tests require a reachable SQL Server with permission to create and drop databases. They use the same `ConnectionStrings__*` templates but replace the database name with a unique `Hotel<Service>Tests_<Guid>` database.

Load the environment:

```bash
set -a
source .env.local
set +a
```

Run the full suite:

```bash
dotnet test HotelManagement.sln
```

Expected current result: **138 passing tests**.

Run only one project:

```bash
dotnet test tests/HotelManagement.Rooms.IntegrationTests
dotnet test tests/HotelManagement.Reservations.UnitTests
```

Run one test by name:

```bash
dotnet test tests/HotelManagement.Reservations.IntegrationTests \
  --filter "FullyQualifiedName~Reschedule_moves_the_stay"
```

Test distribution:

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

Integration tests create and drop their own databases. They must not delete local development databases. If tests fail before cleanup, stale `Hotel*Tests_*` databases may remain and can be removed manually.

## Build and Formatting

```bash
set -a
source .env.local
set +a

dotnet build HotelManagement.sln
dotnet format HotelManagement.sln --verify-no-changes
```

Apply formatting automatically:

```bash
dotnet format HotelManagement.sln
```

## Database Maintenance

List databases:

```bash
sqlcmd -S "localhost,${HOTEL_SQL_PORT:-1434}" -U sa -P "$HOTEL_SQL_SA_PASSWORD" -C \
  -d master -Q "SELECT name FROM sys.databases ORDER BY name;"
```

Remove stale test databases:

```bash
sqlcmd -S "localhost,${HOTEL_SQL_PORT:-1434}" -U sa -P "$HOTEL_SQL_SA_PASSWORD" -C \
  -d master -Q "
    DECLARE @sql nvarchar(max) = N'';
    SELECT @sql = @sql + N'ALTER DATABASE [' + name + N'] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [' + name + N'];'
    FROM sys.databases
    WHERE name LIKE 'Hotel%Tests[_]%';
    EXEC sp_executesql @sql;"
```

Reset one local development database:

```bash
sqlcmd -S "localhost,${HOTEL_SQL_PORT:-1434}" -U sa -P "$HOTEL_SQL_SA_PASSWORD" -C \
  -d master -Q "
    ALTER DATABASE [HotelRooms] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
    DROP DATABASE [HotelRooms];"
```

Then reapply migrations or restart the Rooms API.

## Stop the Services

Stop `dotnet run` processes with `Ctrl+C` in each terminal.

If needed, stop the SQL container:

```bash
docker stop hmnext-sql
```

## Troubleshooting

### `ConnectionStrings:<Service>Db is required`

The environment file was not loaded in the current shell. Run:

```bash
set -a
source .env.local
set +a
```

### Login returns 401

Check:

- The bootstrap Owner environment variables are set before the Auth API starts.
- The Owner was created during startup. Look for `Bootstrap Owner ... created` or `... is ready` in the Auth API log.
- The password meets the Identity policy: at least 12 characters, uppercase, lowercase, digit, and non-alphanumeric character.

### `dotnet ef` cannot connect to SQL Server

- Confirm the container is running: `docker ps`.
- Confirm the host port matches `HOTEL_SQL_PORT` (default `1434`).
- Confirm all connection strings use `Server=localhost,1434`.
- Confirm `TrustServerCertificate=True` and `Encrypt=False` for local development.

### Internal calls return 401 or 503

- `InternalApi__Key` must be identical for Rooms, Guests, and Reservations.
- The Reservations service must use `Services__RoomsBaseUrl=http://localhost:5112` and `Services__GuestsBaseUrl=http://localhost:5113` for local execution.
- If a downstream service is down, Reservations returns `503 Service Unavailable`.

### Swagger is not available

Swagger is enabled only when `ASPNETCORE_ENVIRONMENT=Development`. The aggregated gateway Swagger also requires all downstream Swagger documents to be reachable.

### Integration tests cannot create databases

The SQL login must have `dbcreator` or equivalent permissions. The test factories create databases named `Hotel<Service>Tests_<Guid>` and drop them during disposal.

## Related Documentation

- [Low-level design](docs/LLD.md) — architecture, diagrams, data model, workflows, security, operations
- [API reference](docs/API.md) — endpoints, DTOs, authorization, status codes
- [Testing guide](docs/TESTING.md) — test strategy, isolation, matrices, commands
