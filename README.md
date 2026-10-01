# Appointment Booking API

A REST backend for appointment booking (clinics, salons, meeting rooms) built with Django REST Framework and PostgreSQL. Providers publish availability slots, customers book and cancel them, and the system guarantees that **a slot can never be double-booked, even under concurrent requests**.

## Features

- Two roles: **provider** (publishes slots) and **customer** (books them)
- JWT authentication (access + refresh tokens)
- Availability slots with filtering by provider and date range
- Booking and cancellation with status tracking
- Double-booking and overlapping-slot prevention at two layers (see below)
- Timezone-aware scheduling: stored in UTC, returned in the provider's timezone
- Confirmation and cancellation emails (Django console email backend)
- Object-level permissions: users only see and modify their own data
- Interactive OpenAPI docs (Swagger UI) via drf-spectacular
- pytest suite, including a concurrent-booking test
- Dockerised setup and GitHub Actions CI

## Tech stack

Python 3.12 · Django · Django REST Framework · PostgreSQL · simplejwt · drf-spectacular · pytest / pytest-django · Docker · GitHub Actions

## Design decisions

### Preventing double-booking (two layers)

Validating in the serializer is not enough: two simultaneous requests can both pass the check before either one writes. So the guarantee is enforced twice.

1. **Application layer.** The booking service runs inside `transaction.atomic()` and locks the slot row with `select_for_update()`. It rejects past slots, already-booked slots and invalid cancellations with clear error messages.
2. **Database layer.** Constraints make the invalid state impossible to store, regardless of what the application code does:
   - An `ExclusionConstraint` on `Slot` (using `btree_gist`) rejects any two slots for the same provider whose time ranges overlap.
   - A `CheckConstraint` ensures `end > start`.
   - A partial `UniqueConstraint` on `Booking(slot)` where `status = 'confirmed'` allows only one active booking per slot, while still letting a cancelled slot be rebooked.

If the database rejects a write, the service catches the `IntegrityError` and returns **409 Conflict** instead of a 500.

### Timezones

`USE_TZ = True`. All datetimes are stored in UTC. Each provider has an IANA timezone (e.g. `Asia/Karachi`), and API responses include times converted to it.

## Data model

| Model | Key fields |
|---|---|
| `User` | email, role (`provider` / `customer`) |
| `Provider` | user, name, timezone |
| `Slot` | provider, start, end |
| `Booking` | customer, slot, status (`confirmed` / `cancelled`), created_at, cancelled_at |

## API overview

Full interactive documentation is available at `/api/docs/` when the server is running.

| Method | Endpoint | Description | Access |
|---|---|---|---|
| POST | `/api/auth/register/` | Create an account | Public |
| POST | `/api/auth/token/` | Obtain JWT pair | Public |
| POST | `/api/auth/token/refresh/` | Refresh access token | Public |
| GET | `/api/slots/` | List free slots (`?provider=`, `?from=`, `?to=`) | Authenticated |
| POST | `/api/slots/` | Create a slot | Provider |
| DELETE | `/api/slots/{id}/` | Delete own unbooked slot | Provider (owner) |
| POST | `/api/bookings/` | Book a slot | Customer |
| GET | `/api/bookings/` | List own bookings | Authenticated |
| POST | `/api/bookings/{id}/cancel/` | Cancel a booking | Booking owner |

### Example

```bash
# Get a token
curl -X POST http://localhost:8000/api/auth/token/ \
  -H "Content-Type: application/json" \
  -d '{"email": "customer@example.com", "password": "secret123"}'

# Book a slot
curl -X POST http://localhost:8000/api/bookings/ \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"slot": 12}'
```

Booking a slot that is already taken returns:

```json
{"detail": "This slot is no longer available."}
```
with status `409 Conflict`.

## Getting started

### With Docker (recommended)

```bash
git clone https://github.com/anum-349/booking-api.git
cd booking-api
cp .env.example .env
docker compose up --build
```

The API is served at `http://localhost:8000` and the docs at `http://localhost:8000/api/docs/`.

### Without Docker

Requires Python 3.12+ and a running PostgreSQL instance.

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env            # set your database credentials
python manage.py migrate
python manage.py runserver
```

### Environment variables

| Variable | Description |
|---|---|
| `SECRET_KEY` | Django secret key |
| `DEBUG` | `True` for development |
| `DATABASE_URL` | PostgreSQL connection string |
| `ALLOWED_HOSTS` | Comma-separated host list |

## Running tests

```bash
pytest
```

The suite covers:

- Overlapping slot rejection, both through the API and directly against the database constraint
- Booking, cancelling and rebooking the same slot
- Object-level permissions (customers and providers cannot touch each other's data)
- Timezone conversion
- **Concurrency:** two threads booking the same slot simultaneously; exactly one succeeds

Tests run automatically on every push through GitHub Actions against a PostgreSQL service container.

## Project structure

```
booking-api/
├── config/            # settings, urls, wsgi
├── accounts/          # user model, auth, permissions
├── scheduling/        # providers, slots, bookings
│   ├── models.py      # models + database constraints
│   ├── services.py    # booking / cancellation logic
│   ├── views.py
│   ├── serializers.py
│   └── tests/
├── Dockerfile
├── docker-compose.yml
└── .github/workflows/ci.yml
```

## Possible improvements

- Recurring availability rules that generate slots automatically
- Rescheduling endpoint
- Real email delivery and reminder notifications through a task queue
- Rate limiting on booking endpoints

## Author

**Anum Kousar** — [GitHub](https://github.com/anum-349) · [LinkedIn](https://linkedin.com/in/anum-kousar)
