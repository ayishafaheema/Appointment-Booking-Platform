# Low-Level Design (LLD)
## Hospital Appointment Booking System

## 1. Introduction

This document describes the low-level design of the Hospital Appointment Booking System, including database entities, attributes, relationships, constraints, business rules, and system workflows.

The system is designed for a single hospital with multiple departments, doctors, and patients. A centralized administrator manages the entire hospital.

### 1.1 Project Scope

- One hospital with multiple departments.
- One centralized administrator.
- Multiple doctors and patients.
- Each doctor belongs to exactly one department.
- Doctors do not have separate login accounts in the initial version.
- Patients can book multiple appointments, including multiple appointments on the same day.
- Appointment slots are generated dynamically from doctor availability.
- A separate Notification Service manages email notifications.
- Each service has its own PostgreSQL database.

### 1.2 Main Services

**Appointment Service**
- User and patient management.
- Department and doctor management.
- Doctor availability management.
- Appointment booking and cancellation.
- Rescheduling management.
- Appointment history.
- Notification event creation.

**Notification Service**
- Notification processing.
- Email delivery.
- Notification history.
- Delivery attempt tracking.
- Retry management.

---

# 2. Database Architecture

The system uses two independent PostgreSQL databases.

### 2.1 Appointment Service Database

Contains the following nine application tables:

1. User
2. PatientProfile
3. Department
4. Doctor
5. DoctorAvailability
6. Appointment
7. RescheduleRequest
8. AppointmentHistory
9. NotificationOutbox

### 2.2 Notification Service Database

Contains the following three application tables:

10. Notification
11. NotificationAttempt
12. NotificationTemplate

**Database ownership rules**

- The Appointment Service accesses only its own database.
- The Notification Service accesses only its own database.
- Services communicate through HTTP APIs.
- Neither service directly accesses the other's database.
- Each service manages its own transactions.

---

# 3. Appointment Service Database Design

## 3.1 User

The User entity manages authentication and authorization.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | User identifier |
| email | EmailField | Unique, Not Null | Login email |
| password | CharField | Not Null | Hashed password |
| first_name | CharField | Not Null | First name |
| last_name | CharField | Optional | Last name |
| phone_number | CharField | Optional | Contact number |
| role | CharField | Not Null | ADMIN or PATIENT |
| is_active | BooleanField | Default True | Account status |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Roles

- ADMIN: Centralized hospital administrator.
- PATIENT: Registered patient.

### Business Rules

1. Email addresses must be unique.
2. Passwords must be stored using Django's password hashing mechanism.
3. Administrative operations are restricted to ADMIN users.
4. Patients can access and manage only their own appointments.
5. Doctors do not have separate user accounts in the initial version.

---

## 3.2 PatientProfile

PatientProfile stores additional information about a patient separately from authentication details.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Profile identifier |
| user | OneToOneField | Unique, Not Null | Related User |
| date_of_birth | DateField | Optional | Date of birth |
| address | TextField | Optional | Patient address |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Relationships

- One User can have zero or one PatientProfile.
- Each PatientProfile belongs to exactly one User.
- Only patient accounts require a PatientProfile.

---

## 3.3 Department

Department represents the medical departments available in the hospital.

Examples:

- Dentistry
- Orthopedics
- Dermatology
- Ophthalmology
- Cardiology

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Department identifier |
| name | CharField | Unique, Not Null | Department name |
| description | TextField | Optional | Department description |
| is_active | BooleanField | Default True | Department status |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Relationships

- One Department can have multiple Doctors.
- Each Doctor belongs to exactly one Department.

### Business Rules

1. Department names must be unique.
2. Inactive departments cannot receive new appointments.
3. Historical appointments must remain accessible when a department becomes inactive.
4. Departments should be deactivated rather than physically deleted when historical records depend on them.

---

## 3.4 Doctor

Doctor stores doctor information and their department association.

Doctors do not have separate login accounts in the initial version.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Doctor identifier |
| department | ForeignKey | Not Null | Related Department |
| full_name | CharField | Not Null | Doctor name |
| qualification | CharField | Optional | Medical qualification |
| specialization | CharField | Optional | Specialization |
| phone_number | CharField | Optional | Contact number |
| email | EmailField | Optional | Email address |
| is_active | BooleanField | Default True | Doctor status |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Relationships

- One Department can have multiple Doctors.
- Each Doctor belongs to exactly one Department.
- One Doctor can have multiple DoctorAvailability records.
- One Doctor can have multiple Appointments.

### Business Rules

1. Every doctor must belong to exactly one department.
2. Inactive doctors cannot receive new appointments.
3. If a doctor changes department, the existing doctor record is deactivated and a new record is created.
4. Historical appointments remain associated with the original doctor record.
5. Doctor records should not be physically deleted when historical appointments exist.

---

## 3.5 DoctorAvailability

DoctorAvailability stores the doctor's working schedule.

It defines when a doctor is available for appointments. Individual appointment slots are not stored in a separate table.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Availability identifier |
| doctor | ForeignKey | Not Null | Related Doctor |
| day_of_week | IntegerField | Not Null | Day of the week |
| start_time | TimeField | Not Null | Working start time |
| end_time | TimeField | Not Null | Working end time |
| effective_from | DateField | Not Null | Schedule start date |
| effective_until | DateField | Optional | Schedule end date |
| is_active | BooleanField | Default True | Availability status |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Relationships

- One Doctor can have multiple DoctorAvailability records.
- Each availability record belongs to one Doctor.

### Business Rules

1. Availability must belong to a doctor.
2. The end time must be later than the start time.
3. Availability applies only within its effective date range.
4. Inactive availability records cannot be used for new bookings.
5. Only active doctors and active departments can receive new bookings.
6. Existing appointments must be considered before changing or deactivating availability.

### Dynamic Slot Generation

The system generates available appointment slots dynamically using:

- DoctorAvailability.
- Appointment duration.
- Existing confirmed appointments.
- The requested appointment date.

**Example**

Doctor's working schedule:

- Start time: 09:00 AM
- End time: 12:00 PM
- Fixed appointment duration: 30 minutes

Generated slots:

| Start | End |
|---|---|
| 09:00 | 09:30 |
| 09:30 | 10:00 |
| 10:00 | 10:30 |
| 10:30 | 11:00 |
| 11:00 | 11:30 |
| 11:30 | 12:00 |

The system checks existing appointments and excludes unavailable intervals.

No separate AppointmentSlot table is required.

The initial appointment duration is fixed and centrally configured.

---

## 3.6 Appointment

Appointment is the central entity responsible for managing patient bookings.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Appointment identifier |
| patient | ForeignKey | Not Null | Related PatientProfile |
| doctor | ForeignKey | Not Null | Related Doctor |
| start_at | DateTimeField | Not Null | Appointment start datetime |
| end_at | DateTimeField | Not Null | Appointment end datetime |
| status | CharField | Not Null | Appointment status |
| cancellation_reason | TextField | Optional | Cancellation reason |
| cancellation_remark | TextField | Optional | Additional cancellation details |
| cancelled_by | ForeignKey | Optional | User who cancelled |
| cancelled_at | DateTimeField | Optional | Cancellation time |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

Date and time are stored together as timezone-aware datetime values.

### Relationships

- One PatientProfile can have multiple Appointments.
- One Doctor can have multiple Appointments.
- Each Appointment belongs to one PatientProfile and one Doctor.
- One Appointment can have multiple RescheduleRequest records.
- One Appointment can have multiple AppointmentHistory records.

### Appointment Statuses

| Status | Description |
|---|---|
| CONFIRMED | Appointment successfully booked |
| RESCHEDULE_PENDING | Waiting for patient response |
| COMPLETED | Appointment completed |
| CANCELLED | Appointment cancelled |

### Business Rules

1. Patients can book appointments across different departments.
2. Patients can book multiple appointments on the same day.
3. Appointments can be booked up to seven days in advance.
4. Same-day future appointments are allowed.
5. Appointment duration is fixed initially.
6. Only active doctors can receive new bookings.
7. Only active departments can receive new bookings.
8. The selected appointment interval must fall within doctor availability.
9. A doctor cannot have overlapping confirmed appointments.
10. Patients can cancel their own appointments before the appointment starts.
11. Administrators can cancel appointments.
12. Administrator cancellation requires a reason and remark.
13. Cancelled appointments must remain in the database for historical purposes.

### Double-Booking Prevention

The system uses a PostgreSQL exclusion constraint to prevent overlapping confirmed appointments for the same doctor.

**Example**

| Appointment | Time | Result |
|---|---|---|
| Patient A | 10:00–10:30 | Accepted |
| Patient B | 10:00–10:30 | Rejected |
| Patient C | 10:15–10:45 | Rejected |
| Patient D | 10:30–11:00 | Accepted |

The database checks the complete appointment interval rather than only the start time.

**Database protection concept**

The exclusion constraint compares:

- Doctor ID: equality.
- Appointment time range: overlap.

Conceptually, the PostgreSQL constraint is:

```sql
EXCLUDE USING GIST (
    doctor_id WITH =,
    tstzrange(start_at, end_at, '[)') WITH &&
)
WHERE (status = 'CONFIRMED');
```

The `[)` range means the start is included and the end is excluded. Therefore, an appointment ending at 10:30 does not overlap another appointment beginning at 10:30.

PostgreSQL requires GiST support for this constraint. The `btree_gist` extension provides equality support for the doctor identifier.

The Django implementation will use:

- `transaction.atomic()`.
- PostgreSQL `ExclusionConstraint`.
- Appropriate range expressions.
- Database error handling.

If another request books an overlapping interval first, PostgreSQL rejects the conflicting booking.

The API returns an appropriate message indicating that the selected slot is no longer available.

Only confirmed appointments occupy the booking interval in this initial design. Cancelled and rescheduling-pending appointments do not reserve the original interval.

---

## 3.7 RescheduleRequest

RescheduleRequest manages appointment changes initiated by the administrator when a doctor becomes unavailable.

The patient chooses a replacement date, time, and doctor when necessary.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Request identifier |
| appointment | ForeignKey | Not Null | Related Appointment |
| reason | TextField | Not Null | Rescheduling reason |
| remark | TextField | Optional | Additional information |
| response_deadline | DateTimeField | Not Null | Patient response deadline |
| proposed_doctor | ForeignKey | Optional | Patient-selected replacement doctor |
| proposed_start_at | DateTimeField | Optional | Proposed replacement start |
| proposed_end_at | DateTimeField | Optional | Proposed replacement end |
| status | CharField | Not Null | Request status |
| created_at | DateTimeField | Auto Created | Request creation time |
| responded_at | DateTimeField | Optional | Patient response time |

### Relationships

- One Appointment can have multiple RescheduleRequest records over its lifetime.
- Each RescheduleRequest belongs to one Appointment.
- A RescheduleRequest can reference a proposed replacement Doctor.

### Reschedule Statuses

| Status | Description |
|---|---|
| PENDING | Waiting for patient response |
| ACCEPTED | Replacement appointment confirmed |
| EXPIRED | Response deadline passed |
| CANCELLED | Rescheduling request cancelled |

### Business Rules

1. The administrator initiates a rescheduling request.
2. The request must include a reason.
3. A response deadline must be specified.
4. The appointment status changes to RESCHEDULE_PENDING.
5. The patient receives an email notification.
6. The patient can browse available doctors and appointment times through the application.
7. The patient can select the original doctor if available or another active doctor in the same department.
8. The proposed doctor must belong to the original appointment's department.
9. The replacement slot must be validated before confirmation.
10. The replacement slot must not overlap another confirmed appointment for that doctor.
11. The original appointment must be updated only after the replacement is successfully validated and confirmed.
12. The appointment and rescheduling request must be updated within one database transaction.
13. The request is marked ACCEPTED after successful rescheduling.
14. If the patient does not respond before the deadline, the original appointment is automatically cancelled.
15. A reminder should be sent before the response deadline.
16. Rescheduling history must be preserved.

### Rescheduling Workflow

1. Administrator identifies affected appointments.
2. Administrator initiates a rescheduling request.
3. The request contains a reason and response deadline.
4. Appointment status changes to RESCHEDULE_PENDING.
5. Patient receives an email notification.
6. Patient opens the application.
7. Patient browses available doctors and slots in the same department.
8. Patient selects a replacement doctor, date, and time.
9. System validates the replacement appointment.
10. The system updates the original Appointment record.
11. The RescheduleRequest is marked ACCEPTED.
12. AppointmentHistory records the old and new appointment details.
13. A rescheduling confirmation notification is generated.

The system does not automatically move appointments to another time.

---

## 3.8 AppointmentHistory

AppointmentHistory maintains an audit trail of important appointment changes.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | History identifier |
| appointment | ForeignKey | Not Null | Related Appointment |
| changed_by | ForeignKey | Optional | User responsible for the change |
| action | CharField | Not Null | Action performed |
| previous_status | CharField | Optional | Previous status |
| new_status | CharField | Optional | New status |
| previous_doctor | ForeignKey | Optional | Previous doctor |
| new_doctor | ForeignKey | Optional | New doctor |
| previous_start_at | DateTimeField | Optional | Previous appointment time |
| new_start_at | DateTimeField | Optional | New appointment time |
| reason | TextField | Optional | Reason for the change |
| remark | TextField | Optional | Additional details |
| created_at | DateTimeField | Auto Created | Event timestamp |

### Relationships

- One Appointment can have multiple AppointmentHistory records.
- Each history record belongs to one Appointment.
- The changed_by field identifies the user who performed the action, when applicable.

### Recorded Events

- Appointment booked.
- Appointment cancelled by patient.
- Appointment cancelled by administrator.
- Rescheduling request initiated.
- Appointment rescheduled.
- Appointment automatically cancelled after the deadline.
- Appointment marked as completed.

### Business Rules

1. Important appointment changes must create history records.
2. Historical records must not be physically deleted.
3. History records must be accessible only to authorized users.
4. History records should be created in the same database transaction as the corresponding appointment change.
5. Previous and new doctor/time information must be recorded when an appointment is rescheduled.

---

## 3.9 NotificationOutbox

NotificationOutbox stores notification events generated by the Appointment Service.

It implements the Transactional Outbox Pattern.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Outbox identifier |
| event_id | UUIDField | Unique, Not Null | Unique event identifier |
| event_type | CharField | Not Null | Notification event type |
| aggregate_id | BigIntegerField | Not Null | Related appointment identifier |
| payload | JSONField | Not Null | Event data |
| status | CharField | Not Null | Processing status |
| retry_count | IntegerField | Default 0 | Number of retries |
| next_attempt_at | DateTimeField | Optional | Next retry time |
| created_at | DateTimeField | Auto Created | Event creation time |
| processed_at | DateTimeField | Optional | Successful processing time |
| last_error | TextField | Optional | Last processing error |

### Notification Event Types

- APPOINTMENT_CONFIRMED
- APPOINTMENT_CANCELLED
- RESCHEDULE_REQUESTED
- APPOINTMENT_RESCHEDULED
- APPOINTMENT_REMINDER
- RESCHEDULE_DEADLINE_REMINDER

### Outbox Statuses

| Status | Description |
|---|---|
| PENDING | Waiting to be processed |
| PROCESSING | Currently being processed |
| SENT | Accepted by the Notification Service |
| FAILED | Processing failed |

### Business Rules

1. An outbox event must be created in the same database transaction as the appointment operation that generates it.
2. A unique event ID identifies every event.
3. A background processor retrieves pending events.
4. The processor sends events to the Notification Service through an HTTP API.
5. Failed HTTP requests must be retried.
6. Notification failure must not roll back a successful appointment operation.
7. The outbox record is marked SENT after the Notification Service acknowledges acceptance.
8. Outbox records must be retained according to the data-retention policy.

---

# 4. Notification Service Database Design

The Notification Service has its own PostgreSQL database.

It receives notification events from the Appointment Service, processes them, and manages email delivery.

## 4.1 Notification

Notification stores notification records received from the Appointment Service.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Notification identifier |
| source_event_id | UUIDField | Unique, Not Null | Original event ID |
| recipient_email | EmailField | Not Null | Recipient email |
| event_type | CharField | Not Null | Notification event type |
| subject | CharField | Not Null | Email subject |
| body | TextField | Not Null | Rendered email content |
| status | CharField | Not Null | Delivery status |
| created_at | DateTimeField | Auto Created | Creation time |
| sent_at | DateTimeField | Optional | Successful delivery time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Notification Statuses

| Status | Description |
|---|---|
| PENDING | Waiting for email processing |
| PROCESSING | Email delivery in progress |
| SENT | Email successfully accepted by provider |
| FAILED | Delivery failed |

### Business Rules

1. Every notification must reference a unique source event ID.
2. Duplicate event IDs must not create duplicate notification records.
3. Notifications are created using events received from the Appointment Service.
4. Email delivery status must be tracked.
5. Notification processing is independent of appointment database transactions.

### Idempotency

The `source_event_id` has a unique constraint.

If the same event is received multiple times, the Notification Service checks whether it has already been recorded.

If it exists, the service returns an appropriate acknowledgement rather than creating another notification.

This prevents duplicate notification records during HTTP retries.

---

## 4.2 NotificationAttempt

NotificationAttempt tracks individual email delivery attempts.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Attempt identifier |
| notification | ForeignKey | Not Null | Related Notification |
| attempt_number | IntegerField | Not Null | Attempt sequence |
| status | CharField | Not Null | Attempt result |
| provider_response | TextField | Optional | Email provider response |
| error_message | TextField | Optional | Failure details |
| attempted_at | DateTimeField | Auto Created | Attempt timestamp |

### Relationships

- One Notification can have multiple NotificationAttempt records.
- Each NotificationAttempt belongs to one Notification.

### Business Rules

1. Each email delivery attempt must create a record.
2. Failed attempts must retain error details.
3. Retry attempts must increment the attempt number.
4. Successful delivery updates the related Notification status to SENT.
5. Retry processing must not create a new Notification for the same event.

---

## 4.3 NotificationTemplate

NotificationTemplate stores reusable email templates.

### Fields

| Field | Data Type | Constraints | Description |
|---|---|---|---|
| id | BigAutoField | Primary Key | Template identifier |
| event_type | CharField | Unique, Not Null | Notification event type |
| subject_template | CharField | Not Null | Email subject template |
| body_template | TextField | Not Null | Email body template |
| is_active | BooleanField | Default True | Template status |
| created_at | DateTimeField | Auto Created | Creation time |
| updated_at | DateTimeField | Auto Updated | Last update time |

### Business Rules

1. Each notification event type has an appropriate template.
2. Templates contain placeholders for dynamic information.
3. Inactive templates cannot be used for new notifications.
4. Templates are managed independently by the Notification Service.

---

# 5. Database Relationships Summary

## 5.1 Appointment Service

| Parent Entity | Child Entity | Relationship |
|---|---|---|
| User | PatientProfile | One-to-One |
| Department | Doctor | One-to-Many |
| Doctor | DoctorAvailability | One-to-Many |
| PatientProfile | Appointment | One-to-Many |
| Doctor | Appointment | One-to-Many |
| Appointment | RescheduleRequest | One-to-Many |
| Appointment | AppointmentHistory | One-to-Many |
| Doctor | RescheduleRequest | One-to-Many, optional proposed doctor |
| User | AppointmentHistory | One-to-Many, optional actor |

NotificationOutbox references appointments using `aggregate_id`. This is an event reference, not a cross-service relationship.

## 5.2 Notification Service

| Parent Entity | Child Entity | Relationship |
|---|---|---|
| Notification | NotificationAttempt | One-to-Many |

NotificationTemplate is independently identified by its event type.

There are no direct foreign keys between the two services' databases.

---

# 6. Important Business Workflows

## 6.1 Appointment Booking

1. Patient logs in.
2. Patient selects a department.
3. Patient selects a doctor.
4. System retrieves the doctor's availability.
5. System dynamically generates available appointment slots.
6. Patient selects a date and time.
7. System validates:
   - Doctor is active.
   - Department is active.
   - Date is within the seven-day booking window.
   - Selected time is in the future.
   - Selected interval falls within doctor availability.
   - Appointment duration is valid.
8. System starts a database transaction.
9. PostgreSQL exclusion constraint protects against overlapping confirmed appointments.
10. Appointment is created.
11. AppointmentHistory record is created.
12. NotificationOutbox event is created.
13. Transaction is committed.
14. Background processor sends the event to the Notification Service.
15. Notification Service processes the email.

If the database rejects the booking because of an overlapping appointment, the API returns an appropriate availability error.

## 6.2 Patient Cancellation

1. Patient selects an upcoming appointment.
2. System verifies appointment ownership.
3. System verifies that the appointment has not started.
4. Appointment status changes to CANCELLED.
5. Cancellation time is recorded.
6. AppointmentHistory record is created.
7. NotificationOutbox event is created.
8. Transaction is committed.
9. Notification Service processes the cancellation email.

## 6.3 Administrator Cancellation

1. Administrator selects an appointment.
2. Administrator provides a cancellation reason and remark.
3. Appointment status changes to CANCELLED.
4. Cancellation details are stored.
5. AppointmentHistory record is created.
6. NotificationOutbox event is created.
7. Transaction is committed.

## 6.4 Doctor Unavailability and Rescheduling

1. Administrator identifies affected appointments.
2. Administrator initiates a rescheduling request.
3. The request contains a reason and response deadline.
4. Appointment status changes to RESCHEDULE_PENDING.
5. AppointmentHistory record is created.
6. NotificationOutbox event is created.
7. Patient receives an email notification.
8. Patient opens the application.
9. Patient views the rescheduling request.
10. Patient browses available doctors and slots in the same department.
11. Patient selects a replacement doctor, date, and time.
12. System validates the replacement appointment.
13. System starts a database transaction.
14. Appointment is updated with the replacement doctor and time.
15. RescheduleRequest status changes to ACCEPTED.
16. AppointmentHistory records the previous and new details.
17. NotificationOutbox creates a confirmation event.
18. Transaction is committed.

If the patient does not respond before the deadline:

1. Background processing identifies expired requests.
2. Appointment status changes to CANCELLED.
3. RescheduleRequest status changes to EXPIRED.
4. AppointmentHistory records the cancellation.
5. NotificationOutbox creates a cancellation event.

## 6.5 Notification Processing

1. Appointment Service creates an outbox event.
2. Background processor retrieves pending events.
3. Event is sent to Notification Service through HTTP.
4. Notification Service checks the unique source event ID.
5. If the event is new, a Notification record is created.
6. Notification Service processes email delivery.
7. Each delivery attempt is recorded in NotificationAttempt.
8. Notification status is updated.
9. Appointment Service marks the outbox event SENT after receiving acknowledgement.

If HTTP communication fails, Appointment Service retries the event.

If the same event is received again, Notification Service uses its unique event ID to prevent duplicate notification records.

Email delivery itself is not guaranteed exactly once. The system uses retries and idempotency to reduce duplicate processing and handle temporary failures.

---

# 7. Transaction Management and Concurrency

The Appointment Service uses Django transactions to maintain database consistency.

The following operations should be performed within appropriate database transactions:

- Appointment creation.
- Appointment cancellation.
- Rescheduling request creation.
- Appointment rescheduling.
- Automatic cancellation after a deadline.
- Appointment history creation.
- Notification outbox event creation.

### Transactional Outbox Pattern

When an appointment operation generates a notification, the appointment change and corresponding outbox event must be committed in the same transaction.

For example, during booking:

- Appointment is created.
- Appointment history is created.
- NotificationOutbox event is created.

If any of these operations fails, the transaction is rolled back.

The email is processed after the transaction commits.

### Concurrency Protection

PostgreSQL exclusion constraints prevent overlapping confirmed appointments for the same doctor.

Django transaction handling ensures related database changes are committed together.

The system must also handle constraint violations and return appropriate API responses.

---

# 8. Background Processing

Background processing is required for operations that should not block patient requests.

Responsibilities include:

- Processing pending NotificationOutbox events.
- Retrying failed HTTP requests.
- Sending appointment reminders.
- Sending rescheduling deadline reminders.
- Automatically cancelling expired rescheduling requests.
- Retrying failed email delivery in the Notification Service.

The exact background task implementation will use a suitable scheduling and worker mechanism. The initial architecture does not require RabbitMQ.

---

# 9. Security and Data Integrity

- Use Django authentication and password hashing.
- Restrict administrative actions to the centralized ADMIN role.
- Patients can access only their own appointment information.
- Validate appointment dates and times on the backend.
- Use foreign keys to maintain relationships.
- Use unique constraints where required.
- Use PostgreSQL exclusion constraints to prevent overlapping appointments.
- Use database transactions for related operations.
- Preserve historical appointment records.
- Do not allow inactive doctors or departments to receive new bookings.
- Validate HTTP requests between services.
- Do not expose notification service database access to the Appointment Service.

---

# 10. Final Design Decisions

| No. | Design Area | Final Decision |
|---|---|---|
| 1 | Hospital scope | One hospital |
| 2 | Administrator | One centralized administrator |
| 3 | Departments | Multiple departments |
| 4 | Doctor-department relationship | One doctor belongs to exactly one department |
| 5 | Doctor login | Not included in initial version |
| 6 | Doctor department change | Deactivate old record and create a new record |
| 7 | Appointment duration | Fixed initially |
| 8 | Slot management | Dynamic slot generation |
| 9 | Booking window | Seven days |
| 10 | Same-day booking | Allowed for future slots |
| 11 | Multiple same-day appointments | Allowed |
| 12 | Double-booking prevention | PostgreSQL exclusion constraint |
| 13 | Doctor unavailability | Initiate rescheduling requests |
| 14 | Replacement doctor | Original doctor or another active doctor in the same department |
| 15 | Rescheduling deadline | Required |
| 16 | Notification architecture | Separate Notification Service |
| 17 | Database architecture | Separate PostgreSQL database per service |
| 18 | Service communication | HTTP APIs |
| 19 | Notification reliability | Transactional Outbox Pattern |
| 20 | Duplicate event prevention | Idempotency using unique event IDs |
| 21 | Email retry tracking | NotificationAttempt |
| 22 | Message broker | Not included initially |

---

# 11. Conclusion

This LLD defines the initial database structure, entity relationships, business rules, concurrency protection, and workflows of the Hospital Appointment Booking System.

The design separates appointment management from notification processing while maintaining data consistency through database transactions and the Transactional Outbox Pattern.

The system uses dynamically generated appointment slots and PostgreSQL exclusion constraints to protect against overlapping bookings.

This document serves as the technical blueprint for implementing the Appointment Service and Notification Service.