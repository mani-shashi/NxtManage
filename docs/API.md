# API Reference

This document describes the implemented HTTP API. All public routes are reachable through the gateway at `http://localhost:5110`. Internal routes are reachable only on the owning service and require `X-Internal-Api-Key`.

## Base URLs

| API | Direct base URL | Gateway base URL |
|---|---|---|
| Auth | `http://localhost:5111` | `http://localhost:5110` |
| Rooms | `http://localhost:5112` | `http://localhost:5110` |
| Guests | `http://localhost:5113` | `http://localhost:5110` |
| Reservations | `http://localhost:5114` | `http://localhost:5110` |

Public path prefixes are preserved by the gateway:

```text
/api/auth/*
/api/room-types/*
/api/rooms/*
/api/guests/*
/api/reservations/*
```

Internal routes are intentionally not routed by the gateway:

```text
/internal/*
```

## Authentication

Protected endpoints use a JWT bearer token:

```http
Authorization: Bearer <access-token>
```

The token is issued by `POST /api/auth/login`. The default lifetime is 60 minutes and is configurable with `Jwt__ExpiryMinutes`.

### Roles

| Role | Intended use |
|---|---|
| `Owner` | Full administration, including users, roles, and all operational data |
| `Manager` | Inventory and guest management plus operational reservation work |
| `Receptionist` | Read-only inventory and guest access, operational reservation work, room status changes |

### Authorization policies

| Policy | Allowed roles |
|---|---|
| `Authenticated` | Any authenticated user |
| `OwnerOnly` | `Owner` |
| `Management` | `Owner`, `Manager` |
| `Operations` | `Owner`, `Manager`, `Receptionist` |

### Internal authentication

Internal endpoints require:

```http
X-Internal-Api-Key: <internal-api-key>
```

The key must be at least 32 bytes. A missing or incorrect key returns `401`. The comparison is fixed-time.

## Common Conventions

### Content type

Requests and responses use JSON:

```http
Content-Type: application/json
```

### Dates and timestamps

- Calendar dates use ISO `YYYY-MM-DD`.
- Timestamps use ISO 8601 UTC values ending in `Z`, for example `2026-10-27T12:00:00Z`.
- Check-out must be strictly after check-in.
- Reservation nights are calculated as `CheckOut.DayNumber - CheckIn.DayNumber`.

### Enums

Enums are serialized as strings. String values are preferred in requests. Supported values:

- `RoomStatus`: `Available`, `Occupied`, `Maintenance`, `OutOfService`
- `ReservationStatus`: `Confirmed`, `CheckedIn`, `CheckedOut`, `Cancelled`, `NoShow`
- `IdentityDocumentType`: `Passport`, `NationalId`, `DriverLicense`, `Other`

### Pagination

Paged endpoints accept:

| Query parameter | Default | Limits |
|---|---:|---|
| `page` | `1` | Values below 1 are normalized to 1 |
| `pageSize` | `20` | Clamped to 1–100 |

Response shape:

```json
{
  "items": [],
  "page": 1,
  "pageSize": 20,
  "totalCount": 0,
  "totalPages": 0
}
```

### Correlation ID

Every request accepts an optional:

```http
X-Correlation-ID: <client-generated-id>
```

If omitted, the service uses the current activity ID or trace identifier. The value is echoed in the response header and included in error responses.

### Error format

Errors use RFC 7807-style `ProblemDetails`:

```json
{
  "type": "https://httpstatuses.com/409",
  "title": "Conflict.",
  "status": 409,
  "detail": "The room already has an availability block for the requested dates.",
  "instance": "/api/reservations",
  "traceId": "0HN...",
  "correlationId": "client-or-server-id"
}
```

Validation errors add an `errors` object:

```json
{
  "type": "https://httpstatuses.com/400",
  "title": "Validation failed.",
  "status": 400,
  "detail": "One or more validation errors occurred.",
  "instance": "/api/reservations",
  "errors": {
    "CheckOut": ["Check-out date must be after check-in date."]
  },
  "traceId": "...",
  "correlationId": "..."
}
```

### Status codes

| Code | Meaning |
|---:|---|
| 200 | Successful read or state transition |
| 201 | Resource created |
| 204 | Successful deletion with no body |
| 400 | Validation or domain business-rule failure |
| 401 | Missing/invalid JWT or internal API key |
| 403 | Authenticated but lacking the required role |
| 404 | Resource or route not found |
| 409 | Duplicate value, unavailable room, or conflicting state |
| 500 | Unexpected server failure |
| 503 | Downstream Rooms or Guests service unavailable or timed out |

## Auth API

### `POST /api/auth/login`

Anonymous.

Request:

```json
{
  "email": "owner@hotel.local",
  "password": "ChangeMe_OwnerPassword1!"
}
```

Response `200`:

```json
{
  "accessToken": "<jwt>",
  "expiresAtUtc": "2026-10-27T13:00:00Z",
  "user": {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "email": "owner@hotel.local",
    "displayName": "System Owner",
    "isActive": true,
    "roles": ["Owner"]
  }
}
```

Failures:

- `400` invalid email or empty password.
- `401` unknown user, inactive user, locked-out user, or wrong password.

Inactive and locked-out users cannot log in. Failed passwords increment the Identity access-failed count. The default lockout is 5 failed attempts for 15 minutes.

### `GET /api/auth/me`

Requires an authenticated user.

Response `200`: `AuthUserDto`.

```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "email": "owner@hotel.local",
  "displayName": "System Owner",
  "isActive": true,
  "roles": ["Owner"]
}
```

### `GET /api/auth/users`

Requires `OwnerOnly`.

Query parameters:

| Parameter | Type | Description |
|---|---|---|
| `page` | integer | Page number |
| `pageSize` | integer | Page size |
| `search` | string | Email or display name contains |

Response `200`: paged `UserSummaryDto`.

```json
{
  "items": [
    {
      "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "email": "owner@hotel.local",
      "displayName": "System Owner",
      "isActive": true,
      "createdAtUtc": "2026-10-26T12:00:00Z",
      "roles": ["Owner"]
    }
  ],
  "page": 1,
  "pageSize": 20,
  "totalCount": 1,
  "totalPages": 1
}
```

### `GET /api/auth/users/{id}`

Requires `OwnerOnly`.

Response `200`: `UserSummaryDto`. Returns `404` for an unknown user.

### `POST /api/auth/users`

Requires `OwnerOnly`.

Request:

```json
{
  "email": "manager@example.com",
  "displayName": "Hotel Manager",
  "password": "StrongPassword123!",
  "roles": ["Manager"]
}
```

Password rules:

- At least 12 characters.
- At least one uppercase letter.
- At least one lowercase letter.
- At least one digit.
- At least one non-alphanumeric character.
- At least 4 unique characters.
- Identity also enforces its configured complexity rules.

Role rules:

- Roles are case-insensitive on input and normalized to `Owner`, `Manager`, or `Receptionist`.
- At least one role is required.
- Unknown roles return `400`.

Responses:

- `201` with `UserSummaryDto`.
- `409` if the email already exists.
- `400` for weak password or invalid role.

### `PUT /api/auth/users/{id}/roles`

Requires `OwnerOnly`.

Request:

```json
{
  "roles": ["Manager", "Receptionist"]
}
```

Protections:

- An Owner cannot remove their own `Owner` role.
- The last active Owner cannot be demoted.
- Unknown users return `404`.
- Unknown roles return `400`.

Response `200`: `UserSummaryDto`.

### `POST /api/auth/users/{id}/activate`

Requires `OwnerOnly`.

Response `200`: `UserSummaryDto` with `isActive = true`.

### `POST /api/auth/users/{id}/deactivate`

Requires `OwnerOnly`.

Protections:

- An Owner cannot deactivate their own account.
- The last active Owner cannot be deactivated.

Response `200`: `UserSummaryDto` with `isActive = false`.

Note: deactivation prevents future logins and invalidates the security stamp, but an already issued JWT remains valid until its expiry because services validate JWTs statelessly.

### `GET /api/auth/roles`

Requires `OwnerOnly`.

Response `200`:

```json
[
  { "name": "Manager" },
  { "name": "Owner" },
  { "name": "Receptionist" }
]
```

## Rooms API

### `GET /api/room-types`

Requires `Operations`.

Response `200`: array of `RoomTypeDto`.

```json
[
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "name": "Deluxe",
    "description": "King bed",
    "basePrice": 125.50,
    "maxAdults": 2,
    "maxChildren": 1,
    "createdAtUtc": "2026-10-26T12:00:00Z"
  }
]
```

### `GET /api/room-types/{id}`

Requires `Operations`. Returns `404` if missing.

### `POST /api/room-types`

Requires `Management`.

Request:

```json
{
  "name": "Deluxe",
  "description": "King bed",
  "basePrice": 125.50,
  "maxAdults": 2,
  "maxChildren": 1
}
```

Rules:

- Name required, maximum 100 characters, unique case-insensitively.
- Base price cannot be negative.
- Adults 1–20.
- Children 0–20.

Responses: `201`, `400`, `409`.

### `PUT /api/room-types/{id}`

Requires `Management`. Same body and rules as create. Responses: `200`, `400`, `404`, `409`.

### `GET /api/rooms`

Requires `Operations`.

Query parameters:

| Parameter | Type | Description |
|---|---|---|
| `page` | integer | Page number |
| `pageSize` | integer | Page size |
| `search` | string | Room number or room type name contains |
| `status` | `RoomStatus` | Filter by status |
| `roomTypeId` | GUID | Filter by room type |
| `floor` | integer | Filter by floor |

Response `200`: paged `RoomDto`.

### `GET /api/rooms/available`

Requires `Operations`.

Query parameters:

- `checkIn` required, `YYYY-MM-DD`
- `checkOut` required, `YYYY-MM-DD`
- `roomTypeId` optional

A room is returned when:

- Its status is not `Maintenance` or `OutOfService`.
- No availability block overlaps the requested half-open interval.

Response `200`: array of `RoomDto`. Invalid dates return `400`.

### `GET /api/rooms/{id}`

Requires `Operations`. Returns `404` if missing.

### `POST /api/rooms`

Requires `Management`.

Request:

```json
{
  "roomNumber": "101",
  "floor": 1,
  "roomTypeId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "status": "Available",
  "notes": "Near lift"
}
```

Rules:

- Room number required, maximum 20 characters, letters/digits/hyphens only, unique case-insensitively.
- Floor 0–200.
- Room type must exist.
- Status optional; defaults to `Available`.

Responses: `201`, `400`, `404`, `409`.

### `PUT /api/rooms/{id}`

Requires `Management`. Updates room number, floor, room type, and notes. Does not change status. Responses: `200`, `400`, `404`, `409`.

### `PUT /api/rooms/{id}/status`

Requires `Operations`.

Request:

```json
{
  "status": "Maintenance"
}
```

Valid values: `Available`, `Occupied`, `Maintenance`, `OutOfService`.

Responses: `200`, `400`, `404`.

### Internal Rooms Endpoints

All internal endpoints require `X-Internal-Api-Key`.

#### `GET /internal/rooms/{roomId}`

Returns `InternalRoomDto` with capacity, base price, status, and `isOperational`.

#### `POST /internal/rooms/{roomId}/availability-blocks`

Request:

```json
{
  "startDate": "2027-08-10",
  "endDate": "2027-08-12",
  "referenceId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "notes": "Reservation hold"
}
```

Responses:

- `201` with `AvailabilityBlockDto`.
- `400` invalid dates or notes too long.
- `404` room missing.
- `409` room non-operational or overlapping block.

Block creation runs inside a serializable database transaction, so concurrent overlapping requests cannot both succeed.

#### `PUT /internal/rooms/{roomId}/occupancy`

Request:

```json
{
  "occupied": true
}
```

Sets an available room to `Occupied`, or releases an occupied room to `Available`. Maintenance and out-of-service rooms remain non-operational.

Responses: `200` with `RoomDto`, `404`, `409`.

#### `PUT /internal/availability-blocks/{blockId}`

Request:

```json
{
  "startDate": "2027-08-20",
  "endDate": "2027-08-23"
}
```

Moves an existing hold without releasing it first. The current block is excluded from the overlap check.

Responses:

- `200` with updated `AvailabilityBlockDto`.
- `400` invalid dates.
- `404` block missing.
- `409` room non-operational or overlapping another block.

#### `DELETE /internal/availability-blocks/{blockId}`

Releases a hold. Returns `204` on success and `404` if the block does not exist.

#### `GET /internal/rooms/{roomId}/availability`

Query parameters:

- `checkIn`
- `checkOut`

Response `200`:

```json
{
  "roomId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "isAvailable": true,
  "checkIn": "2027-08-10",
  "checkOut": "2027-08-12"
}
```

## Guests API

### `GET /api/guests`

Requires `Operations`.

Query parameters:

| Parameter | Type | Description |
|---|---|---|
| `page` | integer | Page number |
| `pageSize` | integer | Page size |
| `search` | string | First name, last name, email, or phone contains |
| `isActive` | boolean | Filter by active state |

Response `200`: paged `GuestDto`.

### `GET /api/guests/{id}`

Requires `Operations`. Returns `404` if missing.

### `POST /api/guests`

Requires `Management`.

Request:

```json
{
  "firstName": "Ada",
  "lastName": "Lovelace",
  "email": "ada@example.com",
  "phone": "+44 20 7946 0958",
  "dateOfBirth": "1990-05-15",
  "addressLine": "1 Analytical Engine Way",
  "city": "London",
  "country": "United Kingdom",
  "nationality": "British",
  "identityDocumentType": "Passport",
  "identityDocumentNumber": "P1234567",
  "notes": "Prefers a quiet room"
}
```

Rules:

- First name and last name required, maximum 100 characters.
- Email required, valid, maximum 254, unique case-insensitively.
- Phone required, maximum 30.
- Date of birth, when supplied, between 1900-01-01 and today.
- Optional text limits: address 250, city/country/nationality 100, notes 1000.
- Identity document type and number must be supplied together.
- Document number maximum 50.

Responses: `201`, `400`, `409`.

### `PUT /api/guests/{id}`

Requires `Management`. Same body and rules as create. Responses: `200`, `400`, `404`, `409`.

### `PUT /api/guests/{id}/status`

Requires `Management`.

Request:

```json
{
  "isActive": false
}
```

Inactive guests cannot be used for new reservations. Existing reservations are not modified.

Response `200`: `GuestDto`.

### `GET /internal/guests/{id}`

Requires `X-Internal-Api-Key`.

Response `200`:

```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "fullName": "Ada Lovelace",
  "email": "ada@example.com",
  "phone": "+44 20 7946 0958",
  "isActive": true
}
```

## Reservations API

### `GET /api/reservations`

Requires `Operations`.

Query parameters:

| Parameter | Type | Description |
|---|---|---|
| `page` | integer | Page number |
| `pageSize` | integer | Page size |
| `search` | string | Guest name, guest email, or room number contains |
| `status` | `ReservationStatus` | Filter by status |
| `guestId` | GUID | Filter by guest |

Response `200`: paged `ReservationDto`.

### `GET /api/reservations/{id}`

Requires `Operations`. Returns `404` if missing.

### `POST /api/reservations`

Requires `Operations`.

Request:

```json
{
  "guestId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "roomId": "4fa85f64-5717-4562-b3fc-2c963f66afa7",
  "checkIn": "2027-08-10",
  "checkOut": "2027-08-12",
  "adults": 2,
  "children": 0,
  "specialRequests": "High floor"
}
```

Behavior:

1. Validate request.
2. Fetch the guest from Guests.
3. Reject missing or inactive guests.
4. Fetch the room from Rooms.
5. Reject non-operational rooms and insufficient capacity.
6. Snapshot guest, room, room type, nightly rate, and total.
7. Create an availability hold in Rooms.
8. Save the reservation.
9. If saving fails, release the hold as compensation.

Responses:

- `201` with `ReservationDto`.
- `400` invalid dates or occupancy.
- `404` guest or room missing.
- `409` inactive guest, non-operational room, insufficient capacity, or overlapping hold.
- `503` Rooms or Guests unavailable.

Response example:

```json
{
  "id": "5fa85f64-5717-4562-b3fc-2c963f66afa8",
  "guestId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "guestName": "Ada Lovelace",
  "guestEmail": "ada@example.com",
  "guestPhone": "+44 20 7946 0958",
  "roomId": "4fa85f64-5717-4562-b3fc-2c963f66afa7",
  "roomNumber": "101",
  "roomTypeId": "6fa85f64-5717-4562-b3fc-2c963f66afa9",
  "roomTypeName": "Deluxe",
  "checkIn": "2027-08-10",
  "checkOut": "2027-08-12",
  "nights": 2,
  "adults": 2,
  "children": 0,
  "nightlyRate": 125.50,
  "totalAmount": 251.00,
  "status": "Confirmed",
  "specialRequests": "High floor",
  "cancellationReason": null,
  "createdAtUtc": "2026-10-27T12:00:00Z",
  "updatedAtUtc": null,
  "checkedInAtUtc": null,
  "checkedOutAtUtc": null,
  "cancelledAtUtc": null,
  "noShowAtUtc": null
}
```

### `POST /api/reservations/{id}/check-in`

Requires `Operations`.

Only `Confirmed` reservations can check in. The room must be available. The service:

1. Marks the reservation `CheckedIn`.
2. Sets the room to `Occupied` in Rooms.
3. Saves the reservation.
4. Compensates the room back to `Available` if the save fails.

Repeated check-in on an already checked-in reservation reasserts room occupancy and returns `200`, making retries safe.

Responses: `200`, `400`, `404`, `409` (room cannot be occupied), `503`.

### `POST /api/reservations/{id}/check-out`

Requires `Operations`.

Only `CheckedIn` reservations can check out. The service:

1. Saves the reservation as `CheckedOut`.
2. Releases the availability hold.
3. Sets the room to `Available`.

If downstream synchronization fails, the operation returns an error. A retry is safe: an already checked-out reservation is synchronized again without changing status.

Responses: `200`, `400`, `404`, `503`.

### `POST /api/reservations/{id}/no-show`

Requires `Operations`.

Only `Confirmed` reservations can be marked no-show. The hold is released and the status becomes `NoShow`.

Responses: `200`, `400`, `404`, `503`.

### `PUT /api/reservations/{id}/reschedule`

Requires `Operations`.

Request:

```json
{
  "checkIn": "2027-08-20",
  "checkOut": "2027-08-23"
}
```

Behavior:

1. Only `Confirmed` reservations can be rescheduled.
2. The reservation must have an availability block.
3. The block is moved atomically in Rooms without releasing it first.
4. The reservation dates, nights, and total are recalculated using the existing nightly rate.
5. If local persistence fails, the block is moved back to the original dates as compensation.

Responses:

- `200` with updated `ReservationDto`.
- `400` invalid dates or reservation not confirmed.
- `404` reservation not found.
- `409` no block or the new dates overlap another hold.
- `503` Rooms unavailable.

### `PUT /api/reservations/{id}/cancel`

Requires `Operations`.

Request:

```json
{
  "reason": "Guest changed plans"
}
```

Only `Confirmed` reservations can be cancelled. The hold is released and the status becomes `Cancelled`.

Responses: `200`, `400` (invalid state or reason too long), `404`, `503`.

## Gateway Routing Summary

| Public route | Downstream |
|---|---|
| `/api/auth/{everything}` | Auth `5111` |
| `/api/rooms` and `/api/rooms/{everything}` | Rooms `5112` |
| `/api/room-types` and `/api/room-types/{everything}` | Rooms `5112` |
| `/api/guests` and `/api/guests/{everything}` | Guests `5113` |
| `/api/reservations` and `/api/reservations/{everything}` | Reservations `5114` |

Gateway Swagger aggregates:

| Key | Source |
|---|---|
| `auth` | `http://localhost:5111/swagger/v1/swagger.json` |
| `rooms` | `http://localhost:5112/swagger/v1/swagger.json` |
| `guests` | `http://localhost:5113/swagger/v1/swagger.json` |
| `reservations` | `http://localhost:5114/swagger/v1/swagger.json` |

## Response DTO Summary

### `RoomDto`

```text
id, roomNumber, floor, roomTypeId, roomTypeName, basePrice,
status, notes, createdAtUtc, updatedAtUtc
```

### `AvailabilityBlockDto`

```text
id, roomId, startDate, endDate, referenceId, notes, createdAtUtc
```

### `GuestDto`

```text
id, firstName, lastName, fullName, email, phone, dateOfBirth,
addressLine, city, country, nationality, identityDocumentType,
identityDocumentNumber, notes, isActive, createdAtUtc, updatedAtUtc
```

### `ReservationDto`

```text
id, guestId, guestName, guestEmail, guestPhone, roomId, roomNumber,
roomTypeId, roomTypeName, checkIn, checkOut, nights, adults, children,
nightlyRate, totalAmount, status, specialRequests, cancellationReason,
createdAtUtc, updatedAtUtc, checkedInAtUtc, checkedOutAtUtc,
cancelledAtUtc, noShowAtUtc
```

### `UserSummaryDto`

```text
id, email, displayName, isActive, createdAtUtc, roles
```
