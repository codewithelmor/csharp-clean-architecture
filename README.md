# C# Clean Architecture

In a clean architecture with C#, project separation typically involves organizing your codebase into distinct layers, each with its specific responsibilities. Here's a simplified breakdown:

1. ```Core Layer```:
* Contains entities and business logic.
* Defines interfaces for services and repositories.

```plaintext
MyProject.Core
|-- Entities
|-- Interfaces
   |-- Repositories
      |-- IBaseRepository.cs
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
* Implements use cases and orchestrates business logic.
* Depends on the Core layer.

```plaintext
MyProject.Application
|-- UseCases
   |-- SomeUseCase.cs
|-- Authorization
   |-- AuthorizationHandler.cs
   |-- RoleRequirements
      |-- AdminRequirement.cs
      |-- ManagerRequirement.cs
|-- Services (implements interfaces from Core)
|-- Mappers
├── Validators
│   ├── CreateUserValidator.cs
│   ├── UpdateUserValidator.cs
│   └── ...
```

3 ```Infrastructure Layer```:
* Deals with external concerns (e.g., databases, external services).
* Implements repository interfaces from the Core layer.

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
* Handles input/output (UI, API, etc.).
* Depends on the Application layer.

```plaintext
MyProject.Presentation
|-- Controllers (for API)
|-- Views (for UI)
|-- ViewModels
|-- DTOs
```

5. ```Cross-cutting Concerns```:
* May include aspects like logging, authentication, and validation.
* Shared across layers.

```plaintext
MyProject.CrossCutting
|-- Logging
|-- Authentication
|-- Validation
```

Remember to maintain dependencies inwards, meaning layers closer to the core should not depend on outer layers. This promotes a clear separation of concerns and makes the codebase more modular and testable.
