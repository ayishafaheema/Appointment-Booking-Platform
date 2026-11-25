## System Architecture (HLD)

The Hospital Appointment Booking System follows a service-oriented architecture consisting of two independent services: **Appointment Service** and **Notification Service**.

The system is designed to manage patient appointments across multiple departments within a single hospital while keeping appointment management and email notification responsibilities separate.

### Architecture Diagram

```text
                  PATIENT / ADMIN
                         |
                         v
                   Web Application
                         |
                         v
                APPOINTMENT SERVICE
                      (Django)
                         |
              +----------+----------+
              |                     |
              v                     v
       Appointment DB         Notification
        (PostgreSQL)          Outbox Events
                                    |
                                    v
                            Outbox Processor
                                    |
                               HTTP API
                                    |
                                    v
                          NOTIFICATION SERVICE
                                (Django)
                                    |
                                    v
                            Notification DB
                              (PostgreSQL)
                                    |
                                    v
                              Email Provider
```

### 1. Appointment Service

The Appointment Service is the core backend component responsible for managing hospital operations.

**Responsibilities:**
- Patient registration and authentication.
- Patient profile management.
- Department management.
- Doctor profile management.
- Doctor availability and schedule management.
- Appointment slot generation and booking.
- Appointment cancellation and rescheduling.
- Appointment history management.
- Creation of notification events.

### 2. Notification Service

The Notification Service independently handles appointment-related email communications.

**Responsibilities:**
- Receiving notification requests through HTTP APIs.
- Sending appointment confirmation emails.
- Sending cancellation and rescheduling notifications.
- Managing notification delivery status.
- Tracking failed delivery attempts and retrying failed notifications.

### 3. Database Architecture

The system uses two independent PostgreSQL databases.

| Database | Responsibility |
|---|---|
| Appointment Database | Stores users, patient profiles, departments, doctors, availability, appointments, rescheduling requests, and appointment history. |
| Notification Database | Stores notification records, delivery attempts, and notification templates. |

Each service manages its own database and does not directly access the database of the other service.

### 4. Inter-Service Communication

The Appointment Service and Notification Service communicate through HTTP APIs.

The **Transactional Outbox Pattern** is used to ensure reliable notification processing.

When an appointment-related operation occurs, the Appointment Service saves the corresponding notification event in its outbox table within the same database transaction.

A background processor later sends pending events to the Notification Service.

This ensures that email delivery failures do not affect successful appointment operations.

### 5. Key Architectural Considerations

- **Separation of Concerns:** Appointment management and notification delivery are handled by separate services.
- **Data Integrity:** Database transactions maintain consistency during appointment operations.
- **Concurrency Control:** Database-level mechanisms prevent multiple patients from booking the same appointment slot.
- **Reliability:** Failed notification requests can be retried without affecting appointment records.
- **Historical Data:** Appointment and rescheduling history are preserved for future reference.
- **Scalability:** The architecture allows additional features and services to be introduced in future versions.

### 6. Technology Stack

| Component | Technology |
|---|---|
| Backend | Python, Django |
| API Development | Django REST Framework |
| Frontend | Django Templates (initial implementation) |
| Database | PostgreSQL |
| Service Communication | HTTP APIs |
| Background Processing | Background Workers / Scheduled Tasks |
| Deployment | AWS EC2 |
| Web Server | Nginx |
| Application Server | Gunicorn |
| Process Management | systemd |
| Version Control | Git, GitHub |