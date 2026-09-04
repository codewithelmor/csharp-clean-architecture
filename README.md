# C# Clean Architecture

In a clean architecture with C#, project separation typically involves organizing your codebase into distinct layers, each with its specific responsibilities. Here is a breakdown of what every layer does, what it contains, and how the dependencies flow:

## 1. Core Layer (The Domain)
### What it does:
The **Core Layer** is the heart of your application. It contains the fundamental business concepts, enterprise logic, and data entities. It must have **zero external dependencies**—no references to ORMs (like EF Core), databases, UI frameworks, or other projects in your solution. It represents the pure business reality.

### What it contains:
* **Entities & Value Objects:** Blueprints of your business objects (e.g., `User.cs`, `Order.cs`).
* **Abstractions/Interfaces:** Contracts for data access and infrastructure that outer layers must fulfill (Dependency Inversion).
* **Domain Exceptions:** Core business errors (e.g., `InsufficientFundsException.cs`).
* **Domain Services:** Logic that belongs to the business domain but doesn't naturally fit inside a single Entity.

```plaintext
MyProject.Core
|-- Entities

|   |-- User.cs
|   |-- Order.cs
|-- Interfaces

|   |-- Repositories
|   |   |-- IBaseRepository.cs
|   |   |-- IUserRepository.cs
|   |-- Specifications
|   |   |-- ISomeSpecification.cs
|   |-- IUnitOfWork.cs
|   |-- IDateTimeProvider.cs
|-- DomainServices
|-- Exceptions

|   |-- MyCustomDomainException.cs
|-- Constants

|   |-- DomainConstants.cs
```

---

## 2. Application Layer
### What it does:
The **Application Layer** implements the actual business use cases. It orchestrates the flow of data to and from the entities by using the core interfaces. It defines **how** the application responds to user actions but does not care where the data comes from or how it is displayed. It depends **only** on the Core layer.

### What it contains:
* **Use Cases / Command & Query Handlers:** Application flow logic (often organized via MediatR handlers).
* **Application Interfaces:** Service interfaces specific to application needs (e.g., `IEmailService`).
* **Validation Rules:** Rules checking incoming data before executing a use case (e.g., FluentValidation).
* **Mappers & DTOs:** Mapping logic to convert internal Entities into flat Data Transfer Objects.

```plaintext
MyProject.Application
|-- UseCases

|   |-- Users
|   |   |-- CreateUserCommand.cs
|   |   |-- GetUserQuery.cs
|-- Authorization

|   |-- AuthorizationHandlers
|   |   |-- AuthorizationHandler.cs
|   |-- RoleRequirements
|   |   |-- AdminRequirement.cs
|-- Mappers

|   |-- UserMappingProfile.cs
|-- Validators

|   |-- CreateUserValidator.cs
|   |-- UpdateUserValidator.cs
|-- DTOs

|   |-- UserResponseDto.cs
```

---

## 3. Infrastructure Layer
### What it does:
The **Infrastructure Layer** handles all technical details, external agencies, and plumbing. It implements the interfaces defined in the Core and Application layers. This is where you write code that talks to databases, file systems, third-party APIs, and cloud services. It depends on both **Application** and **Core**.

### What it contains:
* **Database & ORM:** `DbContext`, Entity Framework Configurations, and concrete Repository implementations.
* **External API Clients:** Implementations for sending emails, processing payments, or calling third-party services.
* **Background Tasks:** Hosted services, Quartz/Hangfire jobs, and worker queues.
* **Security Implementations:** Technical tokens, hashing algorithms, and identity management.

```plaintext
MyProject.Infrastructure
|-- Data

|   |-- DbContexts
|   |   |-- ApplicationDbContext.cs
|   |-- EntityConfigurations
|   |   |-- UserEntityConfiguration.cs
|   |-- Repositories (Implements interfaces from Core)
|   |   |-- BaseRepository.cs
|   |   |-- UserRepository.cs
|   |-- UnitOfWork
|   |   |-- EfUnitOfWork.cs
|   |-- Migrations
|-- ExternalServices

|   |-- Email
|   |   |-- SendGridEmailService.cs
|-- Identity / Authentication

|   |-- ClaimsTransformationService.cs
|   |-- JwtTokenGenerator.cs
|-- BackgroundServices

|   |-- BackgroundTaskService.cs
|-- Specifications

|   |-- SomeSpecification.cs
```

---

## 4. Presentation Layer (Web API / UI)
### What it does:
The **Presentation Layer** is the entry point of your system. It processes incoming HTTP requests, handles routing, manages user sessions, and sends back responses (JSON or Views). It is responsible for translating user inputs into Application Use Cases. It depends on **Application** and references Infrastructure strictly to register dependencies in the DI container.

### What it contains:
* **Endpoints:** Controllers, Minimal API endpoints, or UI Razor views.
* **Middleware & Filters:** Global Exception Handlers, custom HTTP logging middleware, and API behaviors.
* **Configuration / Options:** Binding `appsettings.json` properties to strongly typed C# options objects.
* **ViewModels / Request Models:** Simple shapes mapping strictly to incoming HTTP payloads.

```plaintext
MyProject.Presentation
|-- Controllers (or Minimal API Endpoints)

|   |-- UsersController.cs
|-- Middlewares

|   |-- GlobalExceptionMiddleware.cs
|-- Configuration

|   |-- AppSettingsOptions.cs
|-- ViewModels (or Request Payloads)

|   |-- CreateUserRequest.cs
```

---

## 5. Cross-Cutting Concerns
### What it does:
The **Cross-Cutting Concerns** layer contains infrastructure, policies, or code blocks that span multiple layers of your application. It provides the **execution architecture** (the plumbing) for system-wide behaviors. It allows the Core and Application layers to stay clean while remaining supported by logging, technical validation engines, and global authentication frameworks.

### What it contains:
* **Logging System:** Shared logging extensions, Serilog/NLog configurations, and correlation ID managers.
* **Authentication Plumbing:** Global JWT handlers, claims principal extensions, or policy definition registries.
* **Validation Pipeline Engines:** Cross-cutting MediatR pipeline behaviors that intercept requests and run validators automatically before use cases execute.
* **Shared Helpers/Extensions:** Primitive system extensions (e.g., deep cloning, string parsing, reflection utilities).

```plaintext
MyProject.CrossCutting
|-- Logging

|   |-- SerilogConfiguration.cs
|   |-- CorrelationIdMiddleware.cs
|-- Authentication

|   |-- SecurityExtensions.cs
|-- Validation

|   |-- ValidationBehavior.cs (MediatR pipeline behavior)
|-- Helpers

|   |-- EncryptionHelper.cs
|-- Extensions

|   |-- StringExtensions.cs
```

---

Remember to maintain dependencies inwards, meaning layers closer to the core should not depend on outer layers. This promotes a clear separation of concerns and makes the codebase more modular and testable.
