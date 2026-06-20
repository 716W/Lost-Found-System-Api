# LostAndFoundApp — Backend API

> A university Lost & Found management system built with **ASP.NET Core 8**, Clean Architecture, DDD, CQRS, and Firebase Cloud Messaging.

---

## 📄 Documentation

| Document | Description |
|---|---|
| **[API_DOCUMENTATION.md](./API_DOCUMENTATION.md)** | Full API reference — architecture, auth, all endpoints, FCM integration, and deployment |

---

## 🏗️ Tech Stack

| Layer | Technology |
|---|---|
| Framework | ASP.NET Core 8 |
| ORM | Entity Framework Core 8 |
| Database | SQL Server 2022 |
| Auth | ASP.NET Identity + JWT Bearer |
| Messaging | Firebase Cloud Messaging (FCM) |
| CQRS | MediatR |
| Containerisation | Docker + Docker Compose |

---

## 🚀 Quick Start (Docker)

```bash
# Clone the repo
git clone https://github.com/your-org/Lost-Found-System-Api.git
cd Lost-Found-System-Api

# Start all services (DB → Migrations → API)
docker compose up --build
```

The API will be available at **`http://localhost:8080`**.  
Swagger UI: **`http://localhost:8080/swagger`** *(Development only)*

---

## 📁 Project Structure

```text
src/
├── LostAndFound.Api/            # Presentation — Controllers, DTOs, Middleware
├── LostAndFound.Core/           # Domain — Entities, Enums, Interfaces, CQRS
└── LostAndFound.Infrastructure/ # Infrastructure — EF Core, Repositories, Services
```

---

## 🔑 Roles

| Role | Capabilities |
|---|---|
| `User` | Submit reports, claims, feedback; view notifications |
| `Admin` | All user actions + admin dashboards, match verification, handovers |
| `SuperAdmin` | All admin actions + user management |

---

## 🧩 Key Features

- **Automated Matching Algorithm** — fires in the background after every new report; scores Lost vs Found items and sends an FCM push to the owner.
- **Claim & Handover Workflow** — Admin approves claim → creates handover with ID verification and signature capture → report marked `Returned`.
- **Audit Logging** — All mutating admin actions are recorded via `[AuditLog]` action filter.
- **Paginated & Filtered Endpoints** — All list endpoints support pagination; admin endpoints support rich date/status/type filtering.

---

## 📬 FCM Data Payload (Mobile Contract)

When a match is found, the mobile app receives:

```json
{
  "data": {
    "matchId": "42",
    "type":    "match"
  }
}
```

The app should call `GET /api/v1/matches/{matchId}` (with JWT) and navigate to the Match Detail screen.

---

> For the full endpoint reference, request/response shapes, and domain logic — see **[API_DOCUMENTATION.md](./API_DOCUMENTATION.md)**.