# HealthSphere Platform Architecture Documentation

## Containers Overview

### Database Layer

#### postgres-db
**Description:** The PostgreSQL database container provides persistent data storage for the entire HealthSphere platform. It manages all relational data including user accounts, doctor profiles, appointments, communications, and medical documents.

**User Stories:** All user stories (US1-US24) depend on this database for data persistence.

**Ports:** 5432:5432

**Persistence Evaluation:** This container provides persistent storage using PostgreSQL with a mounted volume (pgdata). All application data is stored persistently including user credentials, medical records, appointments, and communications. Database health checks ensure service availability.

**External Services Connections:**
- Connected to all backend microservices for data persistence
- Accessed by pgAdmin for database administration

**Microservice:** PostgreSQL Database
- **Type:** database
- **Description:** Manages persistent storage of all platform data
- **Ports:** 5432
- **Technological Specification:**
  - Database: PostgreSQL (latest)
  - Volume: pgdata for persistent storage
  - Health Check: pg_isready monitoring
  - Environment: POSTGRES_DB=healthsphere, POSTGRES_USER=postgres
- **Service Architecture:** The database follows standard PostgreSQL architecture with persistent volume mounting. Health checks ensure service availability before dependent services start.

##### Database Structure

**users:**
- id (PK)
- email
- password
- role
- profile
- isverified
- isactive
- lastlogin
- createdat
- updatedat
- firstname
- lastname
- phonenumber
- birthplace
- birthdate
- fiscalcode

**doctors:**
- id (PK)
- userid (FK)
- speciality
- bio
- location
- languages
- fee
- rating
- experienceyear
- credentials
- verificationstatus
- profilephoto
- profile
- isactive
- createdat
- updatedat

**doctorprofiles:**
- id (PK)
- userid (FK)
- specialization
- certifications
- experience

**patientprofile:**
- id (PK)
- userid (FK)
- nome
- cognome
- telefono
- datanascita
- luogonascita
- codfiscale

**appointments:**
- id (PK)
- doctorid (FK)
- patientid (FK)
- slotid (FK)
- status
- notes
- createdat
- updatedat

**appointmentslot:**
- id (PK)
- doctorid (FK)
- date
- starttime
- endtime
- isbooked
- createdat
- updatedat

**availabilities:**
- id (PK)
- doctorid (FK)
- type
- data
- isactive

**chat:**
- id (PK)
- participant1id (FK)
- participant2id (FK)
- type
- createdat
- updatedat

**messages:**
- id (PK)
- chatid (FK)
- senderid (FK)
- content
- timestamp
- isread
- createdat
- updatedat

**doctordocuments:**
- id (PK)
- doctorid (FK)
- filename
- originalname
- mimetype
- purpose
- uploadedat

**doctor_patients_relationship:**
- id (PK)
- doctorid (FK)
- patientid (FK)
- status
- requestedat
- respondedat
- terminatedat

**reviews:**
- id (PK)
- patientid (FK)
- doctorid (FK)
- rating
- comment
- createdat
- updatedat

**prescriptions:**
- id (PK)
- patientid (FK)
- doctorid (FK)
- filename
- filepath
- filetype
- note
- uploaddate
- createdat
- updatedat

#### pgadmin
**Description:** Web-based PostgreSQL administration tool that provides a user-friendly interface for database management, monitoring, and maintenance tasks.

**User Stories:** Supports administrative tasks for US21, US22, US23, US24 (admin functionalities).

**Ports:** 5050:80

**Persistence Evaluation:** Uses persistent volume (pgadmin-data) to store pgAdmin configuration and user preferences.

**External Services Connections:**
- Connects to postgres-db for database administration
- Depends on postgres-db health check

**Microservice:** pgAdmin Web Interface
- **Type:** database administration
- **Description:** Provides web interface for PostgreSQL database management
- **Ports:** 5050 (accessible at http://localhost:5050)
- **Technological Specification:**
  - Image: dpage/pgadmin4:latest
  - Environment: PGADMIN_DEFAULT_EMAIL, PGADMIN_DEFAULT_PASSWORD
  - Volume: pgadmin-data for persistent configuration

---

## Backend Microservices

### user-service
**Description:** This container manages user authentication, registration, and profile management for both patients and doctors. It handles user accounts, authentication tokens, and basic profile operations.

**User Stories:** US1, US2, US12, US13, US21

**Ports:** 8081:8081

**Persistence Evaluation:** Uses PostgreSQL database for persistent storage of user accounts, authentication data, and profile information. JWT tokens are stateless and managed in-memory.

**External Services Connections:**
- Connects to postgres-db for user data persistence
- Provides authentication services to other microservices
- Accessed via API Gateway at /api/auth and /api/users routes

**Microservice:** User Authentication & Profile Management
- **Type:** backend
- **Description:** Handles user registration, authentication, and profile management for patients and doctors
- **Ports:** 8081
- **Technological Specification:**
  - Runtime: Node.js
  - Language: JavaScript
  - Authentication: JWT tokens
  - Database: PostgreSQL via connection pooling
  - Environment: DB_NAME=healthsphere, DB_USER=postgres, JWT_SECRET
- **Service Architecture:** RESTful API architecture with JWT-based authentication. Provides centralized user management and authentication services for the entire platform. Uses environment variables for database configuration and JWT secret management.

#### Endpoints

| HTTP Method | URL | Description | User Stories |
|-------------|-----|-------------|--------------|
| POST | /api/auth/register | Register new user (patient/doctor) | US1, US12 |
| POST | /api/auth/login | User login and token generation | US1, US12 |
| GET | /api/users/profile | Get current user profile | US2, US13 |
| PUT | /api/users/profile | Update user profile information | US2, US13 |
| GET | /api/users/me | Get current user details | US2, US13 |

**Database Structure:** Uses users, patientprofile, doctorprofiles, and doctors tables as defined in postgres-db section.

### doctor-service
**Description:** This container manages doctor-specific operations including profile management, credential uploads, availability settings, and doctor verification processes.

**User Stories:** US3, US4, US12, US13, US14, US20, US22

**Ports:** 8082:8082

**Persistence Evaluation:** Uses PostgreSQL for doctor profile data and file system for document storage. Uploaded documents (certifications, licenses) are stored persistently in the uploads volume.

**External Services Connections:**
- Connects to postgres-db for doctor profile data
- File upload handling with persistent volume mounting
- Accessed via API Gateway at /api/doctors route

**Microservice:** Doctor Profile & Document Management
- **Type:** backend
- **Description:** Manages doctor profiles, credentials, availability, and document uploads
- **Ports:** 8082
- **Technological Specification:**
  - Runtime: Node.js
  - Language: JavaScript
  - File Storage: Local file system with volume mounting
  - Database: PostgreSQL connection
  - Volume: ./services/doctor-service/uploads:/app/uploads
  - Environment: JWT_SECRET, INTERNAL_SERVICE_SECRET
- **Service Architecture:** RESTful API with file upload capabilities. Handles multipart form data for document uploads and provides doctor profile management with search and filtering capabilities.

#### Endpoints

| HTTP Method | URL | Description | User Stories |
|-------------|-----|-------------|--------------|
| GET | /api/doctors/search | Search doctors by specialty, location, language | US3, US4 |
| GET | /api/doctors/:id | Get specific doctor profile | US3, US4 |
| PUT | /api/doctors/profile | Update doctor profile | US12, US13 |
| POST | /api/doctors/documents | Upload doctor credentials/certificates | US13, US22 |
| GET | /api/doctors/documents | Get doctor uploaded documents | US13, US22 |
| PUT | /api/doctors/availability | Set working hours and availability | US14 |
| GET | /api/doctors/reviews | Get doctor reviews and ratings | US20 |

**Database Structure:** Uses doctors, doctorprofiles, doctordocuments, and availabilities tables as defined in postgres-db section.

### appointment-service
**Description:** This container handles all appointment-related functionality including booking, scheduling, slot management, and appointment status updates for both patients and doctors.

**User Stories:** US8, US9, US14, US15, US16

**Ports:** 8083:8083

**Persistence Evaluation:** Uses PostgreSQL for persistent storage of appointments, time slots, and availability data. All appointment data is maintained across service restarts.

**External Services Connections:**
- Connects to postgres-db for appointment data persistence
- Uses JWT authentication for user identification
- Accessed via API Gateway at /api/appointments route

**Microservice:** Appointment Booking & Management
- **Type:** backend
- **Description:** Manages appointment bookings, scheduling, and slot availability for patients and doctors
- **Ports:** 8083
- **Technological Specification:**
  - Runtime: Node.js
  - Language: JavaScript
  - Authentication: JWT token validation
  - Database: PostgreSQL connection
  - Environment: DB_HOST=postgres-db, JWT_SECRET
- **Service Architecture:** RESTful API architecture with JWT-based authentication. Manages appointment lifecycle from booking to completion, with real-time slot availability management.

#### Endpoints

| HTTP Method | URL | Description | User Stories |
|-------------|-----|-------------|--------------|
| POST | /api/appointments/book | Book new appointment with doctor | US8 |
| GET | /api/appointments/patient | Get patient's appointments | US9 |
| GET | /api/appointments/doctor | Get doctor's appointments | US16 |
| PUT | /api/appointments/:id/reschedule | Reschedule existing appointment | US9 |
| DELETE | /api/appointments/:id/cancel | Cancel appointment | US9 |
| GET | /api/appointments/slots/:doctorId | Get available time slots for doctor | US8, US14 |
| POST | /api/appointments/slots | Create new appointment slots | US14 |
| PUT | /api/appointments/slots/:id | Update appointment slot | US14 |
| PUT | /api/appointments/:id/status | Update appointment status | US15, US16 |

**Database Structure:** Uses appointments, appointmentslot, and availabilities tables as defined in postgres-db section.

### communication-service
**Description:** This container manages all communication features including chat functionality, support tickets, document sharing, and messaging between patients and doctors.

**User Stories:** US5, US6, US7, US17, US18, US19

**Ports:** 8084:8084

**Persistence Evaluation:** Uses PostgreSQL for persistent storage of chat histories, messages, and communication metadata. All conversations and shared documents are maintained permanently.

**External Services Connections:**
- Connects to postgres-db for message and chat data persistence
- Uses JWT authentication for user identification
- Integrates with user-service for user validation
- Accessed via API Gateway at /api/communication route

**Microservice:** Chat & Communication Management
- **Type:** backend
- **Description:** Manages chat functionality, document sharing, and communication between patients and doctors
- **Ports:** 8084
- **Technological Specification:**
  - Runtime: Node.js
  - Language: JavaScript
  - Authentication: JWT token validation
  - Database: PostgreSQL connection
  - External Services: USER_SERVICE_URL integration
  - Environment: FRONTEND_URL for CORS configuration, INTERNAL_SERVICE_SECRET
- **Service Architecture:** RESTful API with real-time communication capabilities. Manages chat creation, message handling, and document sharing with proper authentication and user relationship validation.

#### Endpoints

| HTTP Method | URL | Description | User Stories |
|-------------|-----|-------------|--------------|
| POST | /api/communication/chat/initiate | Start new chat with doctor | US5 |
| GET | /api/communication/chats | Get user's chat history | US6, US17 |
| POST | /api/communication/messages | Send message in chat | US5, US17 |
| GET | /api/communication/messages/:chatId | Get chat messages | US6, US17 |
| PUT | /api/communication/messages/:id/read | Mark message as read | US6, US17 |
| POST | /api/communication/documents/upload | Upload medical documents | US7, US18 |
| GET | /api/communication/documents | Get shared documents | US7, US18 |
| POST | /api/communication/prescription | Send digital prescription | US19 |
| GET | /api/communication/prescriptions | Get patient prescriptions | US10, US19 |

**Database Structure:** Uses chat, messages, prescriptions, and doctordocuments tables as defined in postgres-db section.

### review-service
**Description:** This container manages the review and rating system, allowing patients to rate doctors and leave feedback about their healthcare experience.

**User Stories:** US11, US20

**Ports:** 3005:3005

**Persistence Evaluation:** Uses PostgreSQL for persistent storage of reviews, ratings, and feedback data. All review information is maintained permanently to build doctor reputation profiles.

**External Services Connections:**
- Connects to postgres-db for review data persistence
- Uses JWT authentication for user identification
- Accessed via API Gateway at /api/reviews route

**Microservice:** Review & Rating Management
- **Type:** backend
- **Description:** Manages patient reviews and ratings for doctors
- **Ports:** 3005
- **Technological Specification:**
  - Runtime: Node.js
  - Language: JavaScript
  - Authentication: JWT token validation
  - Database: PostgreSQL connection
  - Environment: DB_HOST=postgres-db, JWT_SECRET, INTERNAL_SERVICE_SECRET
- **Service Architecture:** RESTful API architecture with JWT-based authentication. Manages review submission, rating aggregation, and feedback display with proper user validation.

#### Endpoints

| HTTP Method | URL | Description | User Stories |
|-------------|-----|-------------|--------------|
| POST | /api/reviews | Submit new review for doctor | US11 |
| GET | /api/reviews/doctor/:id | Get reviews for specific doctor | US20 |
| GET | /api/reviews/patient | Get patient's submitted reviews | US11 |
| PUT | /api/reviews/:id | Update existing review | US11 |
| DELETE | /api/reviews/:id | Delete review | US11 |

**Database Structure:** Uses reviews table as defined in postgres-db section.

### document-service
**Description:** This container manages document upload, storage, and retrieval for medical files, prescriptions, and healthcare-related documents shared between patients and doctors.

**User Stories:** US7, US10, US18, US19

**Ports:** 8085:8085

**Persistence Evaluation:** Uses PostgreSQL for document metadata and file system storage for actual files. Documents are stored persistently in the uploads volume with proper file organization.

**External Services Connections:**
- Connects to postgres-db for document metadata persistence
- File storage with persistent volume mounting
- Accessed via API Gateway at /api/documents route

**Microservice:** Document Management
- **Type:** backend
- **Description:** Manages upload, storage, and retrieval of medical documents and prescriptions
- **Ports:** 8085
- **Technological Specification:**
  - Runtime: Node.js
  - Language: JavaScript
  - File Storage: Local file system with volume mounting
  - Database: PostgreSQL connection
  - Volume: ./services/document-service/uploads:/app/uploads
  - Environment: DB_HOST=postgres-db, PORT=8085
- **Service Architecture:** RESTful API with file upload capabilities. Handles multipart form data for document uploads and provides secure document access with proper authentication.

#### Endpoints

| HTTP Method | URL | Description | User Stories |
|-------------|-----|-------------|--------------|
| POST | /api/documents/upload | Upload medical document | US7, US18 |
| GET | /api/documents | Get user's documents | US7, US10 |
| GET | /api/documents/:id | Download specific document | US7, US10 |
| DELETE | /api/documents/:id | Delete document | US7, US18 |
| POST | /api/documents/prescription | Upload prescription | US19 |
| GET | /api/documents/prescriptions | Get patient prescriptions | US10, US19 |

**Database Structure:** Uses prescriptions and doctordocuments tables as defined in postgres-db section.

---

## Gateway Layer

### api-gateway
**Description:** The API Gateway serves as the central entry point for all client requests, providing routing, load balancing, security, and service orchestration for the HealthSphere microservices architecture.

**User Stories:** All user stories (US1-US24) are routed through the API Gateway.

**Ports:** 3000:3000

**Persistence Evaluation:** Stateless service with no persistent data storage. All requests are proxied to appropriate backend services.

**External Services Connections:**
- Routes requests to user-service, doctor-service, appointment-service, communication-service, review-service, document-service
- Implements CORS for frontend communication
- Provides centralized security and rate limiting

**Microservice:** API Gateway & Proxy
- **Type:** backend gateway
- **Description:** Central routing and security layer for all API requests
- **Ports:** 3000
- **Technological Specification:**
  - Runtime: Node.js
  - Framework: Express.js
  - Proxy: http-proxy-middleware
  - Security: Helmet, CORS, Rate Limiting
  - Language: JavaScript
  - Environment: USER_SERVICE_URL, APPOINTMENT_SERVICE_URL, FRONTEND_URL
- **Service Architecture:** Microservices gateway pattern with request proxying, security middleware, and service discovery. Implements rate limiting (200 requests per 15 minutes) and comprehensive error handling.

#### Routing Configuration

| Route Pattern | Target Service | Description | User Stories |
|---------------|----------------|-------------|--------------|
| /api/auth/* | user-service:8081 | Authentication routes | US1, US12 |
| /api/users/* | user-service:8081 | User management routes | US2, US13, US21 |
| /api/doctors/* | doctor-service:8082 | Doctor profile routes | US3, US4, US12, US13, US14, US20, US22 |
| /api/appointments/* | appointment-service:8083 | Appointment routes | US8, US9, US14, US15, US16 |
| /api/communication/* | communication-service:8084 | Communication routes | US5, US6, US7, US17, US18, US19 |
| /api/reviews/* | review-service:3005 | Review routes | US11, US20 |
| /api/documents/* | document-service:8085 | Document routes | US7, US10, US18, US19 |
| /health | internal | Gateway health check | US23 |

#### Middleware Configuration
- **Security:** Helmet for HTTP headers protection
- **CORS:** Cross-origin requests from frontend
- **Rate Limiting:** 200 requests per 15-minute window
- **Timeout:** 5-second proxy timeout
- **Error Handling:** 503 responses for service unavailability

---

## Frontend Layer

### frontend-service
**Description:** The Frontend-Service container delivers the complete Single Page Application (SPA) for HealthSphere, providing interfaces for patients, doctors, and administrators. Patients can register, search doctors, book appointments, communicate via chat, and manage their health data. Doctors can manage profiles, appointments, patient communications, and upload credentials. Administrators can monitor the platform, verify doctors, and manage users.

**User Stories:** All user stories (US1-US24) are implemented through the frontend interface.

**Ports:** 3001:3001

**Persistence Evaluation:** Client-side application with no persistent data storage. All state management and data persistence is handled by backend microservices through API communication.

**External Services Connections:**
- Communicates via REST API with API Gateway (port 3000)
- All backend interactions routed through api-gateway:
  - Auth & Users for registration, login, and profile management
  - Doctors for search, profiles, and credential management
  - Appointments for booking and scheduling
  - Communication for chat and document sharing
  - Reviews for rating and feedback
  - Documents for file management

**Microservice:** React SPA Frontend
- **Type:** frontend
- **Description:** Single Page Application providing user interfaces for all platform functionality
- **Ports:** 3001
- **Technological Specification:**
  - Framework: React.js
  - Routing: React Router DOM
  - HTTP Client: Axios
  - Styling: Bootstrap
  - Language: JavaScript
  - State Management: React Context (AuthProvider)
  - Build Tool: Node.js
- **Service Architecture:** Single Page Application (SPA) architecture with component-based design, client-side routing, and centralized authentication state management through React Context.

#### Pages

| Name | Description | Related Microservice | User Stories |
|------|-------------|---------------------|--------------|
| Login | Login page for patients, doctors, and admins | user-service | US1, US12 |
| Register | Registration form for new users | user-service | US1, US12 |
| HomePage | Main dashboard after login | api-gateway | US1, US12 |
| Profile | View and edit user profile information | user-service | US2, US13 |
| InfoManagement | Manage additional profile information | user-service | US2, US13 |
| DoctorSearch | Search and filter doctors by criteria | doctor-service | US3, US4 |
| BookAppointment | Book appointments with selected doctors | appointment-service | US8 |
| AppointmentList | View and manage user's appointments | appointment-service | US9, US16 |
| ManageSlots | Doctor interface for managing availability | appointment-service | US14, US15 |
| ChatContainer | Chat interface for patient-doctor communication | communication-service | US5, US6, US17 |
| PendingRequests | Manage doctor-patient relationship requests | communication-service | US15, US17 |
| MyCollaborations | View established patient-doctor relationships | communication-service | US17, US20 |
| ReviewsPage | Submit and view doctor reviews | review-service | US11, US20 |
| DocumentsPage | Manage medical documents and prescriptions | document-service | US7, US10, US18, US19 |

#### Routing Structure
- / - HomePage (dashboard)
- /login - User authentication
- /register - New user registration
- /profile - User profile management
- /infomanagement - Extended profile information
- /doctors/search - Doctor search and filtering
- /appointments/book/:doctorId - Appointment booking
- /appointmentlist - Appointment management
- /manageslots - Doctor availability management
- /chat - Communication interface
- /collaborations/pending - Relationship requests
- /collaborations - Active collaborations
- /reviews - Review management
- /documents - Document management

---

## Monitoring Layer

### prometheus
**Description:** Prometheus container provides metrics collection and monitoring capabilities for the HealthSphere platform, gathering performance data from all microservices and infrastructure components.

**User Stories:** US23, US24 (monitoring and analytics)

**Ports:** 9090:9090

**Persistence Evaluation:** Time-series data storage for metrics collection. Configured with retention policies for historical monitoring data.

**External Services Connections:**
- Collects metrics from all microservices
- Provides data source for Grafana dashboards
- Monitors container health and performance

**Microservice:** Metrics Collection
- **Type:** monitoring
- **Description:** Collects and stores time-series metrics from all platform components
- **Ports:** 9090
- **Technological Specification:**
  - Image: prom/prometheus:v2.50.0
  - Configuration: ./monitoring/prometheus.yml
  - Data Collection: HTTP endpoints scraping
  - Storage: Time-series database

### grafana
**Description:** Grafana container provides visualization and dashboarding capabilities for monitoring HealthSphere platform performance, usage metrics, and system health indicators.

**User Stories:** US23, US24 (monitoring and analytics)

**Ports:** 3006:3000

**Persistence Evaluation:** Dashboard configurations and user settings stored persistently. Connects to Prometheus for data visualization.

**External Services Connections:**
- Connects to Prometheus for metrics data
- Provides web interface for monitoring dashboards
- Depends on Prometheus service availability

**Microservice:** Monitoring Dashboard
- **Type:** monitoring
- **Description:** Provides visual dashboards and alerts for system monitoring
- **Ports:** 3006 (web UI accessible at http://localhost:3006)
- **Technological Specification:**
  - Image: grafana/grafana:10.0.0
  - Authentication: Admin user with configurable password
  - Data Source: Prometheus integration
  - Environment: GF_SECURITY_ADMIN_PASSWORD=admin