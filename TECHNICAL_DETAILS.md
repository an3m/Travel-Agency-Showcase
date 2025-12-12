# Technical Architecture & Implementation Details

This document provides an in-depth technical analysis of the Travel Agency ERP Solution, focusing on architecture decisions, database design, and complex problem-solving approaches.

## Architecture Overview

### Clean Architecture with Feature-First Organization

The project strictly follows **Clean Architecture** principles with a **feature-first** folder structure, ensuring scalability and maintainability. The architecture adheres to SOLID principles, with clear separation of concerns across three primary layers.

```
lib/
├── core/                    # Shared infrastructure and utilities
│   ├── Infrastructure/      # Database, transaction management
│   ├── shared/             # Shared widgets, entities, layouts
│   ├── enums/             # Application-wide enumerations
│   ├── error/             # Error handling (Failures, Exceptions)
│   └── utils/             # Utilities, converters, validators
│
└── features/               # Feature modules (self-contained)
    ├── invoices/
    │   ├── data/          # Data layer
    │   │   ├── data_sources/  # DAOs, local data sources
    │   │   ├── models/       # Data models (DTOs)
    │   │   └── repositories/ # Repository implementations
    │   ├── domain/        # Business logic layer
    │   │   ├── entities/     # Domain entities
    │   │   ├── repositories/ # Repository interfaces
    │   │   └── usecases/     # Business use cases
    │   └── presentation/   # UI layer
    │       ├── bloc/         # State management (BLoC/Cubit)
    │       ├── pages/        # Screen widgets
    │       └── widgets/      # Feature-specific widgets
    ├── services/
    ├── customers/
    ├── parties/
    └── ...
```

### Layer Responsibilities

#### 1. Presentation Layer

- **State Management**: Uses Flutter Bloc and Cubit for predictable state management
- **UI Components**: Feature-specific widgets and pages
- **Navigation**: GoRouter for declarative routing with deep linking support
- **Responsive Design**: Flutter ScreenUtil for adaptive layouts across devices

#### 2. Domain Layer

- **Entities**: Pure Dart classes representing business objects, framework-agnostic
- **Repository Interfaces**: Abstract contracts for data operations, following Dependency Inversion Principle
- **Use Cases**: Encapsulated business logic operations, each with single responsibility
- **Error Handling**: Dartz Either type for functional error handling, ensuring all error paths are explicitly defined
- **No Dependencies**: Domain layer is completely framework-agnostic, enabling testability and maintainability

#### 3. Data Layer

- **Data Sources**: DAOs (Data Access Objects) using Drift for database operations
- **Models**: Data Transfer Objects (DTOs) for serialization and data transformation
- **Repository Implementations**: Concrete implementations of domain repository interfaces
- **Error Handling**: Converts exceptions to domain-level failures using Dartz Either type

## Functional Error Handling with Dartz

### Type-Safe Error Management

The application uses **Dartz**'s `Either<Failure, Success>` pattern throughout the domain and data layers. This functional programming approach ensures:

- **Compile-Time Safety**: All error paths must be explicitly handled, preventing unhandled exceptions
- **Type Safety**: Errors are part of the type system, making error handling mandatory
- **Explicit Error Propagation**: Error types are clearly defined and propagated through the call stack
- **Testability**: Error scenarios can be easily tested by checking the Left side of Either

### Implementation Pattern

```dart
Future<Either<Failure, List<Service>>> getAllServices() {
  return FunctionHelper.repoFunctionContainer(
    () async {
      return await _serviceDao.getAllServices();
    },
    'ServicesRepositoryImpl:getAllServices',
  );
}
```

### Failure Types

- `ServerFailure`: Network/API errors
- `CacheFailure`: Local storage errors
- `ValidationFailure`: Input validation errors
- `UnknownFailure`: Unexpected errors

### Benefits in Clean Architecture Context

When combined with Clean Architecture:

1. **Domain Layer Independence**: Domain layer defines Failure types without knowing implementation details
2. **Data Layer Flexibility**: Data layer can map various exceptions to domain failures
3. **Presentation Layer Clarity**: UI layer receives explicit error types, enabling precise error handling
4. **Code Stability**: Compiler enforces error handling, reducing runtime exceptions

## Database Architecture

### Drift ORM with Type-Safe SQL

The application uses **Drift** (formerly Moor) for type-safe SQLite database operations. All database schemas are defined as Dart classes, enabling compile-time query verification and preventing SQL injection vulnerabilities.

### Core Database Tables

#### 1. Double-Entry Accounting System

The financial system implements a complete double-entry accounting model:

**JournalEntries Table**

- Represents accounting transactions
- Tracks entry type, date, description, currency, and settlement method
- Links to multiple entry lines (debit/credit pairs)

**EntryLines Table**

- Implements double-entry bookkeeping
- Each journal entry has balanced debit and credit lines
- Tracks account relationships, amounts, and conversion ratios
- Ensures accounting equation: Assets = Liabilities + Equity

**Accounts Table**

- Chart of accounts with hierarchical structure
- System accounts (predefined) and user-created accounts
- Multi-currency balance tracking (SAR, YER)
- Unique account codes for identification

#### 2. Invoice Management

**Invoices Table**

- Supports both Sale and Purchase invoices
- Multi-currency support with conversion ratios
- Payment terms and status tracking
- Links to journal entries for accounting integration
- References parties (agents/suppliers) for relationship tracking

**InvoicePayments Table**

- Many-to-many relationship between invoices and payments
- Tracks partial payments and payment allocation
- Maintains invoice remaining amount calculations

#### 3. Service Management

**Services Table**

- Polymorphic service types: Umrah, Visa, Ticket, Passport, Other
- Links to customers, sale invoices, and purchase invoices
- Service status tracking with audit trail
- Cost and sale amount tracking in multiple currencies

**Service Detail Tables** (Polymorphic Design)

- `UmrahDetails`: Stay dates, hotel information, package details
- `VisaDetails`: Visa type, issue date, expiration, country
- `TicketDetails`: Flight information, travel dates, routes
- `PassportIssuanceDetails`: Passport number, issue/expiry dates
- `OtherServiceDetails`: Flexible structure for custom services

**ServiceStatusAudits Table**

- Complete audit trail of service status changes
- Tracks previous/current status, timestamps, and notes
- Enables service lifecycle analysis and reporting

#### 4. Party Management

**PartyTable** (Unified Table)

- Single table design for both Agents and Suppliers
- Discriminated by `partyType` enum
- Supports soft deletion with `isDisabled` flag
- Supplier categorization for grouping

**Views for Optimization**

- `AgentView`: Filtered view of agents from PartyTable
- `SupplierView`: Filtered view of suppliers from PartyTable
- Reduces query complexity and improves performance

#### 5. Payment System

**Payments Table**

- Unified payment/receipt tracking
- Multiple settlement methods: Cash, Bank Transfer
- Bank account linking for transfer tracking
- Conversion rate tracking for multi-currency payments
- Immediate vs. deferred payment flag

**Payment-Invoice Relationship**

- `InvoicePayments` junction table
- Supports partial payments and payment allocation
- Automatic remaining amount calculation

### Database Views for Complex Queries

The system uses **Drift Views** to optimize complex queries:

- **SalesView**: Aggregated sales data with party information
- **PurchaseView**: Aggregated purchase data with supplier details
- **InvoiceView**: Complete invoice information with party details
- **Service Views**: Type-specific views (UmrahView, VisaView, etc.)
- **ExpenseTypeView**: Expense categorization with statistics

### Database Migrations

The system implements a robust migration strategy:

```dart
@override
int get schemaVersion => 16;

@override
MigrationStrategy get migration => MigrationStrategy(
  onCreate: (migrator) async {
    await migrator.createAll();
    await customInserts(); // Initialize system accounts
  },
  onUpgrade: (migrator, from, to) async {
    // Version-specific migration logic
    // Handles column additions, view recreations, data transformations
  },
);
```

**Migration Highlights**:

- Handles schema evolution from version 1 to 16
- Automatic view recreation when base tables change
- Data preservation during structural changes
- System account initialization on fresh installs

## Transaction Management

### ACID-Compliant Operations

The application implements a custom `TransactionManager` abstraction:

```dart
abstract class TransactionManager {
  Future<T> runInTransaction<T>(Future<T> Function() operation);
}
```

**Implementation**: `DriftTransactionManager` wraps Drift's transaction API, ensuring:

- **Atomicity**: All operations succeed or fail together
- **Consistency**: Database constraints are maintained
- **Isolation**: Concurrent operations don't interfere
- **Durability**: Committed transactions persist

**Use Cases**:

- Invoice creation with journal entries and entry lines
- Payment processing with account balance updates
- Service creation with status audit trail
- Multi-step operations requiring rollback capability

## Complex Challenges Solved

### 1. Multi-Currency Financial Calculations

**Challenge**: Maintaining accurate balances across multiple currencies (SAR, YER) with conversion rate tracking.

**Solution**:

- `Money` entity with currency-aware calculations
- Conversion ratio tracking at transaction level
- Account-level balance tracking per currency
- Real-time conversion for reporting

**Implementation**:

```dart
class Money {
  final Currency currency;
  final double amount;

  Money convert(Currency to, double ratio) { ... }
  Money operator +(Money other) { ... }
}
```

### 2. Double-Entry Accounting Automation

**Challenge**: Ensuring every financial transaction maintains accounting equation balance.

**Solution**:

- Automated journal entry creation for invoices and payments
- Entry line generation with balanced debits/credits
- Account balance updates through entry lines
- Validation to prevent unbalanced entries

**Flow**:

1. Business event (e.g., invoice creation)
2. Generate journal entry
3. Create balanced entry lines (debit = credit)
4. Update account balances
5. Link to invoice/payment record

### 3. Service Status Lifecycle Management

**Challenge**: Managing complex service status transitions with audit trails and business rules.

**Solution**:

- State machine pattern for status transitions
- Automated status updates based on business events:
  - Service creation → `pending`
  - Purchase invoice linking → `onProgress`
  - Service completion → `completed`
  - Delivery → `delivered`
- Complete audit trail in `ServiceStatusAudits` table
- Notification triggers for status changes using WorkManager and local notifications

### 4. Offline-First Data Synchronization

**Challenge**: Ensuring data consistency and availability without internet connectivity.

**Solution**:

- SQLite local database as single source of truth
- All operations work offline by default
- Optimistic UI updates with local persistence
- Transaction-based operations for data integrity
- Supabase integration for optional remote synchronization when connectivity is available

### 5. Generic Service Type Handling

**Challenge**: Supporting multiple service types (Umrah, Visa, Ticket, etc.) with type-specific details.

**Solution**:

- Polymorphic database design with base `Services` table
- Type-specific detail tables with foreign key relationships
- Factory pattern for service model creation
- Type-safe service type enums
- Generic repository methods with type filtering

### 6. Complex Reporting Queries

**Challenge**: Generating reports across multiple related entities with filtering and aggregation.

**Solution**:

- Database views for pre-computed aggregations
- Repository pattern with query builders
- Filter options objects for flexible querying
- Date range filtering with timezone handling
- Entity type filtering (customer, agent, supplier, service)
- PDF generation for professional report output
- Excel export for external analysis

## Dependency Injection Architecture

### GetIt Service Locator

The application uses **GetIt** for dependency injection with a hierarchical registration pattern:

```dart
final sl = GetIt.instance;

Future<void> initializeDependencies() async {
  // Core infrastructure
  sl.registerSingleton<AppDatabase>(AppDatabase());
  sl.registerLazySingleton<TransactionManager>(...);

  // Feature-specific dependencies
  AuthDependency.init();
  CustomersDependency.init();
  InvoicesDependency.init();
  // ... other features
}
```

**Benefits**:

- Lazy initialization for performance
- Singleton management for shared resources
- Factory pattern for multiple instances
- Easy testing with mock replacements
- Clear dependency graph
- Adheres to Dependency Inversion Principle

### Feature-Level Dependency Modules

Each feature has its own dependency injection module:

```dart
class InvoicesDependency {
  static void init() {
    // Data sources
    sl.registerLazySingleton<InvoiceDao>(...);

    // Repositories
    sl.registerLazySingleton<InvoiceRepository>(...);

    // Use cases
    sl.registerLazySingleton<AddSaleInvoiceUsecase>(...);

    // BLoCs/Cubits
    sl.registerFactory<SaleInvoiceCreationCubit>(...);
  }
}
```

## State Management Patterns

### BLoC Pattern for Complex State

**InvoicesBloc**: Manages invoice list state with pagination, filtering, and real-time updates.

**ServicesBloc**: Handles service catalog with type filtering and search.

**CustomerBloc**: Manages customer CRUD operations with document handling.

### Cubit for Simpler State

**SaleInvoiceCreationCubit**: Manages complex invoice creation form state with validation.

**PaymentCubit**: Handles payment processing with multi-step workflows.

**CashFlowCubit**: Manages cash flow dashboard with real-time balance calculations.

### Flutter Hooks Integration

Used for reusable stateful logic in UI components:

- Form field management
- Debounced search
- Animation controllers
- Lifecycle management

## Document Generation & Reporting

### PDF Generation

The system uses the `pdf` and `printing` packages for professional document generation:

- **Invoice Printing**: Generate professional PDF invoices with company branding
- **Report Generation**: Create comprehensive financial and operational reports
- **Cross-Platform Printing**: Support for printing on Android and iOS devices

### Excel Export

The `excel` package enables data export capabilities:

- **Data Export**: Export reports and data sets to Excel format
- **External Analysis**: Enable users to perform advanced analysis in spreadsheet applications
- **Bulk Operations**: Support for exporting large datasets

### Data Visualization

**FL Chart** provides interactive data visualization:

- **Financial Dashboards**: Real-time charts for revenue, expenses, and cash flow
- **Trend Analysis**: Time-series charts for business metrics
- **Interactive Elements**: User-friendly chart interactions for data exploration

## Background Processing & Notifications

### WorkManager Integration

The `workmanager` package enables background task scheduling:

- **Periodic Tasks**: Scheduled checks for expiring services and stays
- **Data Synchronization**: Background sync operations when connectivity is available
- **System Integration**: Native platform integration for reliable background execution

### Local Notifications

The `flutter_local_notifications` package provides notification capabilities:

- **Service Alerts**: Notifications for service status changes
- **Expiration Warnings**: Alerts for expiring Umrah stays and visas
- **Payment Reminders**: Notifications for pending payments
- **Notification History**: Complete audit trail of all notifications

## AI & Document Processing

### Face Detection

**Google ML Kit Face Detection** provides machine learning capabilities:

- **Customer Verification**: Face detection for customer photo verification
- **Document Validation**: Verify identity documents during customer onboarding
- **Security**: Enhanced security through biometric verification

### Mobile Scanner

**Mobile Scanner** enables barcode and QR code scanning:

- **Quick Data Entry**: Scan documents and codes for rapid data input
- **Document Processing**: Process identification documents and travel documents
- **Inventory Management**: Scan service codes and tracking numbers

## Responsive Design

### Flutter ScreenUtil

The `flutter_screenutil` package ensures responsive layouts:

- **Adaptive Sizing**: Automatic scaling based on device screen size
- **Consistent UI**: Maintains design consistency across different devices
- **Tablet Support**: Optimized layouts for tablet devices
- **Orientation Handling**: Support for portrait and landscape orientations

## Performance Optimizations

### 1. Database Indexing

Strategic indexes on frequently queried columns:

- `invoices_transaction_id_index`: Fast journal entry lookups
- `invoices_invoice_reference_index`: Quick reference searches
- `account_code_index`: Unique account code lookups

### 2. Lazy Loading & Pagination

- Paginated queries for large datasets
- Lazy initialization of expensive resources
- On-demand data loading in lists

### 3. View Optimization

Database views pre-join related tables, reducing query complexity:

- Single query instead of multiple joins
- Reduced data transfer
- Improved query performance

### 4. State Management Efficiency

- Selective state updates (BLoC's `buildWhen`)
- Memoization of expensive calculations
- Debounced search inputs
<!-- 
## Testing Strategy

### Unit Testing

- Use case testing with mock repositories
- Repository testing with in-memory databases
- Business logic validation
- Error handling verification with Dartz Either types

### Integration Testing

- Database transaction testing
- Multi-step workflow testing
- Error scenario validation

### Widget Testing

- UI component testing
- State management verification
- User interaction testing -->

## Code Generation

### Drift Code Generation

Drift generates type-safe database code:

- Table classes
- DAO implementations
- Query builders
- Type converters

**Build Command**:

```bash
flutter pub run build_runner build --delete-conflicting-outputs
```

### Benefits

- Compile-time query verification
- Type-safe database operations
- Reduced runtime errors
- Better IDE support

## Platform-Specific Features

### Android

- WorkManager for background tasks
- Notification channels
- File system access for documents

### Cross-Platform

- Responsive design with ScreenUtil
- RTL support for Arabic
- Theme-aware UI components

## SOLID Principles Implementation

### Single Responsibility Principle

- Each use case handles a single business operation
- Repository interfaces define clear data access contracts
- BLoCs manage specific feature state

### Open/Closed Principle

- Repository pattern allows extension without modification
- Use case pattern enables adding new operations without changing existing code
- Factory patterns for service model creation

### Liskov Substitution Principle

- Repository implementations are interchangeable
- Use case implementations follow consistent contracts
- Entity inheritance maintains behavioral contracts

### Interface Segregation Principle

- Repository interfaces are focused and specific
- Use case interfaces define minimal required methods
- Clear separation between data access and business logic

### Dependency Inversion Principle

- Domain layer defines repository interfaces
- Data layer implements domain interfaces
- Presentation layer depends on domain abstractions
- GetIt enables dependency injection without tight coupling

## Scalability Considerations

### 1. Modular Architecture

- Feature-first structure enables team scalability
- Independent feature development
- Easy feature addition/removal

### 2. Database Scalability

- Efficient indexing strategy
- Query optimization through views
- Pagination for large datasets

### 3. State Management Scalability

- BLoC pattern supports complex state
- Cubit for simpler use cases
- Clear state boundaries

### 4. Code Maintainability

- Clean Architecture separation
- Repository pattern for testability
- Use case pattern for reusability
- Functional error handling ensures predictable error paths

## Key Design Patterns Used

1. **Repository Pattern**: Data access abstraction
2. **Use Case Pattern**: Business logic encapsulation
3. **Factory Pattern**: Service model creation
4. **Observer Pattern**: BLoC state management
5. **Strategy Pattern**: Payment method handling
6. **State Machine Pattern**: Service status management
7. **Builder Pattern**: Query construction
8. **Singleton Pattern**: Service locator, database instance

## Best Practices Implemented

- **SOLID Principles**: All five principles strictly followed
- **DRY**: Reusable components and utilities
- **Type Safety**: Strong typing throughout, including error types
- **Error Handling**: Comprehensive error management with Dartz Either
- **Code Organization**: Clear structure and naming conventions
- **Documentation**: Inline comments and type documentation
- **Performance**: Optimized queries and state management
- **Security**: Local-first data storage with optional secure backend integration

---

This technical architecture demonstrates enterprise-level software engineering practices, focusing on maintainability, scalability, and reliability. The offline-first design ensures business continuity, while Clean Architecture combined with functional error handling (Dartz) ensures code stability and predictable behavior. The strict adherence to SOLID principles enables long-term maintainability and team collaboration.
