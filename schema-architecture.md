# Section 1: Architecture summary

This Smart Clinic app is a Spring Boot, three-tier web system that mixes server-rendered MVC with API-first design. Admin and Doctor dashboards are generated with Thymeleaf, while patient-facing modules (appointments, dashboards, records) talk to REST endpoints over JSON. All inbound requests hit controllers that delegate into a focused service layer, where business rules live (e.g., validating inputs, coordinating workflows, enforcing availability). Persistence is split across two datastores to match data shape: MySQL via Spring Data JPA for relational entities (Patient, Doctor, Appointment, Admin) and MongoDB via Spring Data MongoDB for flexible, document-style prescriptions. This separation of concerns improves scalability, testability, and deployability—each tier can evolve independently and ships cleanly in containers.

Under the hood, repositories hide database details behind simple interfaces, and model binding keeps things consistent end-to-end: JPA entities map to relational tables; MongoDB documents map to BSON/JSON collections. Responses surface as either HTML (MVC views) or JSON (REST DTOs), enabling everything from browser dashboards to future mobile apps and third-party integrations.

# Section 2: Numbered flow of data and control

1. A user opens an Admin/Doctor dashboard (Thymeleaf) or a patient module (Appointments/PatientDashboard/PatientRecord) via REST.
2. The request is routed to the matching controller—MVC for views, REST for JSON.
3. The controller validates input and calls the service layer to execute the use case.
4. Services apply business rules and orchestrate work across entities (e.g., check doctor availability, enforce constraints).
5. Services delegate persistence to repositories—JPA repositories for MySQL, Mongo repositories for prescriptions.
6. Repositories read/write the databases, binding results into domain models: `@Entity` for MySQL, `@Document` for MongoDB.
7. The controller returns the outcome: models to Thymeleaf for server-rendered HTML, or DTOs serialized to JSON for API clients.
