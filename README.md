# C# Clean Architecture

In a clean architecture with C#, project separation typically involves organizing your codebase into distinct layers, each with its specific responsibilities. Here's a simplified breakdown:

1. ```Core Layer```:
* **What it does:** The absolute heart of your system. It contains fundamental enterprise business rules, entities, and logic. It has **zero dependencies** on external frameworks, databases, or UI systems (no references to EF Core, SQL Server, etc.).
* **What it contains:** Core database models (Entities), contract definitions for repositories/services, specifications, domain exceptions, and system-wide utilities/constants.

```plaintext
MyProject.Core
|-- Entities
|-- Interfaces
   |-- Repositories
      |-- IBaseRepository.cs
   |-- Specifications
      |-- ISomeSpecification.cs
   |-- IUnitOfWork.cs
|-- Services
|-- Repositories
|-- Exceptions
   |-- MyCustomException.cs
|-- Helpers
   |-- UtilityHelper.cs
   |-- OtherHelper.cs
|-- Extensions
   |-- EntityExtensions.cs
   |-- StringExtensions.cs
   |-- OtherExtensions.cs
|-- Constants
   |-- AppConstants.cs
   |-- OtherConstants.cs
```

2. ```Application Layer```:
* **What it does:** Implements and orchestrates user workflows and use cases. It acts as the coordinator of data, taking commands from the UI, pulling entities using the Core interfaces, and executing business logic.
* **What it contains:** Concrete use case execution flows, system authorization policies, mapping tools to format output, request validators (e.g., checking that passwords match or email fields are valid before saving data) and data transfer objects (DTOs includes Request and Reponse)
* **DTO Specifics (Request / Response):** 
  * **Role:** They serve as the application's input/output network contracts. They cross into the Presentation Layer to accept data or return results, but they belong to the Application layer to keep the Domain pure. They contain zero user-interface or visual state logic.
  * **Request Models:** Flat data carriers mapping incoming HTTP payloads or user intents (e.g., `CreateProductRequest`).
  * **Response Models:** Structured wrappers mapping outgoing results securely back over the wire while hiding raw database tracks (e.g., `ProductResponse`).

```plaintext
MyProject.Application
|-- UseCases
   |-- SomeUseCase.cs
|-- Authorization
   |-- AuthorizationHandlers
      |-- AuthorizationHandler.cs
   |-- RoleRequirements
      |-- AdminRequirement.cs
      |-- ManagerRequirement.cs
|-- Services (implements interfaces from Core)
|-- Mappers
|-- Validators
   |-- CreateUserValidator.cs
   |-- UpdateUserValidator.cs
   |-- ...
|-- DTOs (Request / Response)
   |-- Requests
      |-- CreateUserRequest.cs
      |-- UpdateProductRequest.cs
   |-- Responses
      |-- UserResponse.cs
      |-- ProductResponseDto.cs
```

3 ```Infrastructure Layer```:
* **What it does:** Deals with external, data-driven concerns and external agencies. It implements the abstractions defined in the Core Layer (like Data Access, Email Providers, or Crypto engines). It depends directly on Core and Application.
* **What it contains:** Technical plumbing like EF Core DbContexts, repository database implementations, SQL migrations, background/hosted workers, data generator seeds, and web-specific infrastructure like custom HTTP middlewares or AppSettings parsing.

```plaintext
MyProject.Infrastructure
|-- Authentication
   |-- ClaimsTransformationService.cs
|-- Configuration
   |-- AppSettingsOptions.cs
   |-- OtherOptionsModels.cs
|-- Data (database-related implementations)
   |-- EntityConfigurations
      |-- UserEntityConfiguration.cs
      |-- OtherEntityConfiguration.cs
   |-- DbContexts
      |-- ApplicationDbContext.cs
      |-- OtherDbContext.cs
   |-- UnitOfWork
      |-- EfUnitOfWork.cs
      |-- OtherUnitOfWork.cs
|-- ExternalServices
|-- Repositories (implements interfaces from Core)
|-- Migrations
|-- Specifications
   |-- SomeSpecification.cs
|-- BackgroundServices
   |-- BackgroundTaskService.cs
   |-- OtherBackgroundService.cs
|-- Middlewares
   |-- CustomMiddleware.cs
   |-- OtherMiddleware.cs
|-- DummyDataGenerators
   |-- TestDataGenerator.cs
   |-- OtherDataGenerator.cs
```

4. ```Presentation Layer```:
* **What it does:** The user-facing interface or network API boundary. Its only job is to receive a network call or input request, translate it into an application action (like sending a command to a use case), and return the response layout back to the caller.
* **What it contains:** API controllers, frontend Razor views, view models.
* **ViewModel Specifics:** 
  * **Role:** They belong exclusively to this layer. They shape and hold data tailored to a specific user-facing layout screen (like Blazor components or MVC Razor views). They hold state properties that the backend doesn't care about—such as UI error display strings, loading spinners (`IsSaving`), interactive boolean flags, or style themes.
  * **Workflow:** UI elements bind directly to the ViewModel. On initialization, incoming **Response DTOs** from the Application layer are mapped into a local ViewModel. Upon user interaction or execution (such as a form submit), the page maps the values out of the **ViewModel** into an Application-friendly **Request DTO** to safely pass through the boundaries.

```plaintext
MyProject.Presentation
|-- Controllers (for API)
|-- Views (for UI / Pages)
|-- ViewModels
   |-- UserProfileViewModel.cs
   |-- ProductFormViewModel.cs
```

5. ```Cross-cutting Concerns```:
* **What it does:** Houses system-wide technical execution code blocks that bridge across multiple, independent project layers. It supplies the engineering architecture framework needed to monitor, validate, and secure execution flows uniformly.
* **What it contains:** Multi-layer logging aggregators, cross-layer authentication tokens/claims evaluation, and system-wide automatic validation filters or pipeline behaviors.

```plaintext
MyProject.CrossCutting
|-- Logging
|-- Authentication
|-- Validation
```

Remember to maintain dependencies inwards, meaning layers closer to the core should not depend on outer layers. This promotes a clear separation of concerns and makes the codebase more modular and testable.
