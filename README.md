# 🏥 Hospital Appointment Booking System

A backend-focused hospital appointment booking system designed to manage doctors, departments, appointment scheduling, cancellations, and rescheduling through a centralized platform.

## 📌 Project Overview

The Hospital Appointment Booking System aims to simplify the appointment management process for patients and hospital administrators.

Patients can browse departments, view doctors, check available appointment slots, and manage their appointments. A centralized administrator manages doctors, departments, schedules, and appointment-related operations.

The system is designed with a focus on **data consistency, reliable appointment booking, concurrency handling, and maintainable backend architecture.**

## ✨ Key Features

### Patient
- Register and log in to the platform.
- Browse hospital departments and doctors.
- View available appointment slots.
- Book multiple appointments, including on the same day.
- View upcoming appointments and appointment history.
- Cancel appointments before their scheduled start time.
- Respond to rescheduling requests.

### Administrator
- Manage hospital departments and doctor records.
- Manage doctor availability.
- View and manage appointments.
- Cancel appointments with a recorded reason and remark.
- Initiate rescheduling when a doctor becomes unavailable.

### System
- Prevent double booking of appointment slots.
- Maintain appointment and rescheduling history.
- Send email notifications for important appointment events.
- Process notifications independently of appointment operations.

## 🏗️ System Architecture

The system follows a service-oriented architecture with two main services:

- **Appointment Service:** Handles patients, departments, doctors, availability, appointments, and rescheduling.
- **Notification Service:** Handles email notifications and notification processing.

Each service has its own PostgreSQL database. The services communicate through HTTP APIs.

## 🛠️ Technology Stack

| Category | Technology |
|---|---|
| Backend | Python, Django, Django REST Framework |
| Database | PostgreSQL |
| API Communication | REST API |
| Frontend | Django Templates |
| Deployment | AWS EC2 |
| Web Server | Nginx, Gunicorn |
| Version Control | Git, GitHub |

## 📚 Project Documentation

Detailed system design and technical documentation are maintained separately.

- **[High-Level Design (HLD)](docs/HLD.md)** – System architecture, service responsibilities, communication, and major design decisions.
- **[Low-Level Design (LLD)](docs/LLD.md)** – Database design, entities, relationships, business rules, and detailed implementation logic.

## 🚀 Project Status

<escape>**Status:** In Progress</escape>

The project is currently in the system design and development phase. Features and architectural components will be implemented and refined as development progresses.

## 🎯 Project Goal

To build a reliable and maintainable backend system that demonstrates practical understanding of backend development, database design, REST APIs, system architecture, transaction management, and concurrency handling.
