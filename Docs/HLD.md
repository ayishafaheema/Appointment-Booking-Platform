# Hospital Appointment Booking System

## 1. Project Overview

The Hospital Appointment Booking System is a web-based application designed to simplify appointment scheduling between patients and doctors within a single hospital.

The hospital consists of multiple medical departments, such as Dentistry, Cardiology, Orthopedics, Ophthalmology, and Dermatology.

Patients can register, browse departments, explore doctors, check available appointment slots, and book appointments online. A centralized administrator manages hospital departments, doctor profiles, availability schedules, and appointment-related operations.

The system follows a service-oriented architecture with separate Appointment and Notification Services to maintain clear separation of responsibilities and improve reliability.

## 2. System Scope and Actors

The initial version supports one hospital with multiple departments and a centralized administrator.

### Patient

Patients can:

- Register and authenticate.
- Manage their personal profiles.
- Browse hospital departments.
- View doctors within each department.
- Check available appointment slots.
- Book appointments.
- View upcoming appointments and appointment history.
- Cancel their own appointments before the appointment starts.
- Respond to rescheduling requests.
- Receive appointment-related email notifications.

### Administrator

A single administrator manages the entire hospital.

The administrator can:

- Create and manage departments.
- Add, update, activate, and deactivate doctor profiles.
- Assign doctors to departments.
- Manage doctor availability schedules.
- View and manage appointments.
- Handle doctor unavailability.
- Initiate appointment rescheduling requests.
- Set response deadlines for rescheduling requests.
- Cancel appointments with a reason and remark.

Doctors do not have separate login accounts in the initial version.

## 3. Key Features

### Authentication and User Management
- Patient registration and login.
- Role-based access control for patients and administrators.
- Patient profile management.

### Department and Doctor Management
- Multiple hospital departments.
- Doctor profiles associated with individual departments.
- Doctor activation and deactivation.
- Preservation of historical doctor and appointment records.

### Doctor Availability Management
- Management of doctor working schedules.
- Availability based on working days and time ranges.
- Generation of appointment slots based on configured schedules.

### Appointment Management
- Appointment booking based on available slots.
- Booking allowed up to seven days in advance.
- Same-day future appointments supported.
- Multiple appointments per patient per day allowed.
- Fixed appointment duration in the initial version.
- Prevention of multiple confirmed bookings for the same slot.
- Patient and administrator cancellation workflows.

### Rescheduling Management
- Administrator-initiated rescheduling requests.
- Patient notification when an appointment is affected by doctor unavailability.
- Patient selection of a new suitable date and time.
- Configurable response deadlines.
- Automatic cancellation when a patient does not respond before the deadline.

### Email Notifications
- Appointment confirmation emails.
- Cancellation notifications.
- Rescheduling request notifications.
- Rescheduling reminders and outcome notifications.
- Retry handling for failed notification delivery.

## 4. High-Level System Architecture

The application consists of two independent backend services:

1. Appointment Service
2. Notification Service

The Appointment Service manages the core hospital operations, while the Notification Service handles email communication.

### Architecture Diagram

```text
                    PATIENT / ADMIN
                           |
                           v
                    WEB APPLICATION
                           |
                           v
                  APPOINTMENT SERVICE
                        (Django)
                           |
              +------------+------------+
              |                         |
              v                         v
       APPOINTMENT DATABASE       NOTIFICATION OUTBOX
          (PostgreSQL)                    |
                                          v
                                  OUTBOX PROCESSOR
                                          |
                                     HTTP API
                                          |
                                          v
                                 NOTIFICATION SERVICE
                                       (Django)
                                          |
                                          v
                                  NOTIFICATION DATABASE
                                      (PostgreSQL)
                                          |
                                          v
                                    EMAIL PROVIDER
```

### 4.1 Appointment Service

The Appointment Service is the core backend service responsible for the application's business logic.

**Responsibilities:**

- User authentication and authorization.
- Patient profile management.
- Department management.
- Doctor profile management.
- Doctor availability management.
- Appointment slot generation.
- Appointment booking and cancellation.
- Rescheduling request management.
- Appointment history tracking.
- Creation of notification events.

**Database:** PostgreSQL

### 4.2 Notification Service

The Notification Service is responsible for handling email communication independently of the appointment operations.

**Responsibilities:**

- Receiving notification requests through HTTP APIs.
- Processing appointment-related notification events.
- Generating email content using notification templates.
- Sending emails through an external email provider.
- Tracking notification delivery status.
- Recording failed delivery attempts.
- Retrying failed notifications.

**Database:** PostgreSQL

## 5. Database Architecture

The system uses separate PostgreSQL databases for the two services.

### Appointment Service Database

Stores the core hospital and appointment-related information.

Main entities:

- User
- PatientProfile
- Department
- Doctor
- DoctorAvailability
- Appointment
- RescheduleRequest
- AppointmentHistory
- NotificationOutbox

### Notification Service Database

Stores notification-related information.

Main entities:

- Notification
- NotificationAttempt
- NotificationTemplate

**Database Ownership:** Each service manages its own database. The Notification Service does not directly access the Appointment Service database. Communication takes place through HTTP APIs.

## 6. Appointment Booking Workflow

The appointment booking process follows these steps:

1. The patient logs into the application.
2. The patient selects a hospital department.
3. The patient selects a doctor from that department.
4. The patient selects an appointment date.
5. The system retrieves the doctor's available appointment slots.
6. The patient selects a suitable slot and submits the booking request.
7. The Appointment Service validates the booking request.
8. The system verifies doctor availability and checks whether the selected slot is still available.
9. The appointment is created within a database transaction.
10. A corresponding notification event is saved in the outbox.
11. The patient receives an appointment confirmation email through the Notification Service.

### Double-Booking Prevention

The system must prevent multiple patients from successfully booking the same appointment slot.

For example, if two patients attempt to book the same doctor's 10:00 AM slot simultaneously, only one booking should succeed.

This is handled through database-level concurrency control rather than relying only on application-level availability checks.

The exact implementation will be defined during Low-Level Design.

## 7. Notification Processing Workflow

The Appointment Service and Notification Service communicate through HTTP APIs.

The system uses the **Transactional Outbox Pattern** to ensure reliable notification processing.

Workflow:

1. An appointment-related operation takes place.
2. The Appointment Service saves the appointment changes and corresponding outbox event within the same database transaction.
3. A background processor retrieves pending outbox events.
4. The processor sends the notification request to the Notification Service.
5. The Notification Service creates a notification record.
6. The service attempts to send the email through the external email provider.
7. The notification delivery status is recorded.
8. Failed notification attempts are retried according to the retry policy.

This approach ensures that temporary email or notification service failures do not undo successful appointment operations.

## 8. Background Processing

The system uses background processing for operations that should not depend on an active user request.

Main background tasks include:

- Processing pending notification outbox events.
- Sending rescheduling reminders.
- Identifying expired rescheduling requests.
- Automatically cancelling appointments when the response deadline expires.
- Retrying failed notification requests.

A background worker and scheduled tasks will handle these operations.

## 9. Important Design Decisions

| Design Area | Decision |
|---|---|
| Hospital Structure | One hospital with multiple departments |
| Administration | One centralized administrator |
| Doctor Accounts | No separate doctor login in the initial version |
| Doctor Department Relationship | Each doctor belongs to exactly one department |
| Doctor Department Change | Deactivate the old record and create a new doctor record |
| Patient Booking | Multiple appointments per patient per day |
| Booking Window | Up to seven days in advance |
| Appointment Duration | Fixed duration initially |
| Double Booking | Prevented through database-level concurrency control |
| Backend Architecture | Separate Appointment and Notification Services |
| Database | Independent PostgreSQL database for each service |
| Service Communication | HTTP APIs |
| Message Broker | Not included in the initial version |
| Notification Reliability | Transactional Outbox Pattern |
| Background Processing | Background workers and scheduled tasks |

## 10. Security and Reliability Considerations

### Security
- Secure password hashing.
- Role-based access control.
- Patient-specific appointment access.
- Input validation and authorization checks.
- CSRF protection.
- Secure session handling.
- Environment-based secret management.
- HTTPS in production.

### Reliability and Data Integrity
- Database transactions for critical appointment operations.
- Database-level protection against double booking.
- Independent notification processing.
- Retry handling for failed email delivery.
- Preservation of appointment and rescheduling history.
- Database backup and recovery strategy.

## 11. Technology Stack

| Component | Technology |
|---|---|
| Programming Language | Python |
| Backend Framework | Django |
| API Development | Django REST Framework |
| Frontend | Django Templates (initial implementation) |
| Database | PostgreSQL |
| Service Communication | HTTP APIs |
| Background Processing | Background Workers / Scheduled Tasks |
| Notification Service | Django |
| Email Delivery | External Email Provider |
| Deployment | AWS EC2 |
| Web Server | Nginx |
| Application Server | Gunicorn |
| Process Management | systemd |
| Version Control | Git and GitHub |

## 12. Future Enhancements

The architecture can be extended in future versions to support:

- Multiple hospitals and branches.
- Multiple administrator accounts.
- Doctor login and dedicated dashboards.
- React-based frontend.
- Online payment integration.
- SMS notifications.
- Mobile application.
- Message broker integration.
- Advanced reporting and analytics.

These enhancements are outside the scope of the initial version.