🐾 Pet Management System
Core Components:

Entities: Pet and Appointment with JPA annotations
Repositories: JPA repositories with custom queries
Services: Business logic with validation and JMS integration
Controllers: RESTful API endpoints with proper HTTP status codes
Exception Handling: Global exception handler with custom exceptions

Advanced Features:

Scheduled Services:

Daily vaccination reminders (9 AM)
Hourly appointment reminders
Midnight cleanup of old records


JMS Messaging:

Event-driven architecture
Message queues for notifications
Automatic event publishing


Configuration:

Spring Security with basic auth
Swagger/OpenAPI documentation
H2 database with console access
ActiveMQ embedded broker



Testing Suite:

Unit Tests: Service and controller layer tests with Mockito
Integration Tests: Full application workflow testing
Test Configuration: Separate test profiles

Additional Features:

Validation: Input validation with custom validators
Caching: Performance optimization with Spring Cache
Utilities: Date handling and validation utilities
Monitoring: Actuator endpoints for health checks

How to Run:

Build: mvn clean install
Run: mvn spring-boot:run
Access: http://localhost:8080/api
Swagger UI: http://localhost:8080/api/swagger-ui/
H2 Console: http://localhost:8080/api/h2-console

API Endpoints:

Pets: GET, POST, PUT, DELETE /pets
Appointments: GET, POST, PUT, DELETE /appointments
Search: GET /pets/search?name=...
Pet Appointments: GET /appointments/pet/{petId}

Key Features:
* Complete CRUD operations
* Data validation and error handling
* Scheduled tasks for reminders
* JMS messaging system
* Comprehensive testing
* Security configuration
* API documentation
* Production-ready configuration
