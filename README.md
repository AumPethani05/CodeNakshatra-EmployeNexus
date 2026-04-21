# EmployeNexus

A full-stack Employee Management System built on a React 16 frontend and a Spring Boot 2.3 backend, connected through a RESTful HTTP API over Axios. The system provides complete CRUD lifecycle management for employee records persisted in a MySQL relational database via JPA/Hibernate ORM.

```
# Terminal 1 — start the Spring Boot API server
cd springboot-backend
./mvnw spring-boot:run

# Terminal 2 — start the React development server
cd react-frontend
npm install
npm start
```

---

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [Architecture](#architecture)
- [Data Model](#data-model)
- [API Architecture](#api-architecture)
- [Component Reference](#component-reference)
- [Backend Engine](#backend-engine)
- [Configuration](#configuration)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

EmployeNexus is a layered, full-stack Employee Management architecture. The frontend is a React class-component SPA that manages local state, dispatches HTTP calls through a dedicated service layer, and uses React Router v5 for client-side navigation. The backend is a Spring Boot application that exposes a versioned REST API, delegates all persistence to Spring Data JPA, and maps a single `Employee` entity to a MySQL table with automatic DDL management.

The goal is a clean, two-tier separation: the React layer never touches the database directly, and the Spring layer never concerns itself with presentation. All inter-tier communication is JSON over HTTP.

**What EmployeNexus implements:**

- Full CRUD REST API: `GET`, `POST`, `PUT`, `DELETE` on `/api/v1/employees`
- React class component lifecycle (`componentDidMount`) for data fetching on route mount
- Axios HTTP client encapsulated in a typed service module (`EmployeeService.js`)
- JPA/Hibernate ORM mapping — zero hand-written SQL; schema managed via `ddl-auto=update`
- Spring `@RestController` with `@CrossOrigin` CORS policy scoped to `localhost:3000`
- `ResourceNotFoundException` mapped to HTTP `404 Not Found` via `@ResponseStatus`
- React Router `<Switch>` with parametric routes (`/add-employee/:id`, `/view-employee/:id`)
- Unified create/update form component routing on the sentinel value `_add` as the path `id`
- Bootstrap 4 table and card layout for the employee list and detail views
- Maven Wrapper (`mvnw`) and Create React App (`react-scripts`) for zero-global-tooling builds

---

## Installation

**Requirements:**

- Java 8+ (JDK)
- Maven 3.6+ (or use the included `mvnw` wrapper — no global install needed)
- Node.js 12+ and npm
- MySQL 5.7+ running locally on port `3306`

**1. Database setup**

Create the target schema in MySQL. The Spring Boot application will create the `employees` table automatically on first run via `ddl-auto=update`.

```sql
CREATE DATABASE employee_management_system;
```

**2. Configure credentials**

Edit `springboot-backend/src/main/resources/application.properties` and set your MySQL username and password:

```properties
spring.datasource.username=<your_username>
spring.datasource.password=<your_password>
```

**3. Start the backend**

```
cd springboot-backend
./mvnw spring-boot:run
```

The API server starts on `http://localhost:8080`. Maven resolves all dependencies (`spring-boot-starter-data-jpa`, `spring-boot-starter-web`, `mysql-connector-java`) from the central repository on first run.

**4. Start the frontend**

```
cd react-frontend
npm install
npm start
```

The React development server starts on `http://localhost:3000` and proxies nothing — all API calls are made directly to `localhost:8080` via the base URL defined in `EmployeeService.js`.

---

## Architecture

EmployeNexus is organized into two top-level modules, each with a distinct internal layer structure:

```
CodeNakshatra-EmployeNexus/
├── react-frontend/                        React SPA — presentation & HTTP dispatch
│   ├── package.json                       npm manifest; declares axios, react-router-dom, bootstrap
│   └── src/
│       ├── index.js                       DOM entry point — mounts <App /> into #root
│       ├── App.js                         Router root — declares all <Route> mappings
│       ├── components/
│       │   ├── HeaderComponent.js         Static navigation bar
│       │   ├── FooterComponent.jsx        Static page footer
│       │   ├── ListEmployeeComponent.jsx  Route "/" and "/employees" — list + delete
│       │   ├── CreateEmployeeComponent.jsx Route "/add-employee/:id" — create or update
│       │   ├── UpdateEmployeeComponent.jsx Standalone update form (currently unused route)
│       │   └── ViewEmployeeComponent.jsx  Route "/view-employee/:id" — read-only detail
│       └── services/
│           └── EmployeeService.js         Axios API client — single source of truth for all HTTP calls
│
└── springboot-backend/                    Spring Boot REST API — business logic & persistence
    ├── pom.xml                            Maven build descriptor; Spring Boot 2.3, Java 8
    └── src/main/
        ├── resources/
        │   └── application.properties     Server port, datasource URL, Hibernate dialect
        └── java/net/javaguides/springboot/
            ├── SpringbootBackendApplication.java   @SpringBootApplication entry point
            ├── controller/
            │   └── EmployeeController.java         @RestController — HTTP endpoint definitions
            ├── model/
            │   └── Employee.java                   @Entity — JPA-mapped domain object
            ├── repository/
            │   └── EmployeeRepository.java         JpaRepository<Employee, Long> interface
            └── exception/
                └── ResourceNotFoundException.java  @ResponseStatus(404) runtime exception
```

**Data flow for creating an employee:**

```
User fills form in CreateEmployeeComponent
  → onChange handlers update component state (firstName, lastName, emailId)
  → saveOrUpdateEmployee() detects id === '_add'
  → EmployeeService.createEmployee(employee)
      → axios.post("http://localhost:8080/api/v1/employees", { firstName, lastName, emailId })
          → EmployeeController.createEmployee(@RequestBody Employee)
              → employeeRepository.save(employee)
                  → Hibernate generates INSERT INTO employees ...
                      → MySQL persists the row; auto-incremented id returned
  → Response JSON propagates back up the Axios promise chain
  → history.push('/employees') triggers route change → ListEmployeeComponent mounts
  → componentDidMount() fires getEmployees() → table re-renders with new record
```

**Data flow for deleting an employee:**

```
User clicks Delete in ListEmployeeComponent
  → deleteEmployee(id) called
  → EmployeeService.deleteEmployee(id)
      → axios.delete("http://localhost:8080/api/v1/employees/{id}")
          → EmployeeController.deleteEmployee(@PathVariable Long id)
              → employeeRepository.findById(id) — throws ResourceNotFoundException if absent
              → employeeRepository.delete(employee)
                  → Hibernate generates DELETE FROM employees WHERE id = ?
  → Response { "deleted": true } received
  → setState filters deleted record from employees array — React re-renders table in place
```

---

## Data Model

All employee data is stored in a single `employees` table, managed exclusively through the `Employee` JPA entity. Hibernate derives the full DDL from the annotations — no migration scripts are required in this version.

**`Employee.java` — annotated entity:**

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    @Column(name = "first_name")
    private String firstName;

    @Column(name = "last_name")
    private String lastName;

    @Column(name = "email_id")
    private String emailId;
}
```

**Database column mapping:**

| Java Field    | Column Name    | SQL Type         | Constraints              | Notes                                             |
|---------------|----------------|------------------|--------------------------|---------------------------------------------------|
| `id`          | `id`           | `BIGINT`         | `PRIMARY KEY NOT NULL`   | Auto-incremented by the database (`IDENTITY` strategy) |
| `firstName`   | `first_name`   | `VARCHAR(255)`   | none                     | Bound to form input; no length validation in V1  |
| `lastName`    | `last_name`    | `VARCHAR(255)`   | none                     | Bound to form input; no length validation in V1  |
| `emailId`     | `email_id`     | `VARCHAR(255)`   | none                     | No uniqueness constraint or format validation in V1 |

**`@GeneratedValue(strategy = GenerationType.IDENTITY)`** delegates ID generation to the MySQL `AUTO_INCREMENT` column attribute. Hibernate never reads the sequence; it relies on `LAST_INSERT_ID()` returned from the driver after each insert.

**`ddl-auto = update`** instructs Hibernate to inspect the live schema at startup and issue `ALTER TABLE` statements for any missing columns or tables, without dropping existing data. The `employees` table is created on first boot if it does not already exist.

---

## API Architecture

`EmployeeController.java` is the sole HTTP interface for the backend. It is annotated `@RestController`, which combines `@Controller` and `@ResponseBody`, meaning every handler method serializes its return value directly to JSON via Jackson — no view templates involved.

**Base path:** `/api/v1/`

**CORS:** `@CrossOrigin(origins = "http://localhost:3000")` is applied at the class level. Every endpoint in the controller accepts pre-flight and actual requests from the React dev server origin. This must be updated to a production domain before deployment.

**Endpoint inventory:**

| Method   | Path                      | Handler                  | Request Body          | Success Response                              |
|----------|---------------------------|--------------------------|-----------------------|-----------------------------------------------|
| `GET`    | `/api/v1/employees`       | `getAllEmployees()`       | none                  | `200 OK` — JSON array of all `Employee` objects |
| `POST`   | `/api/v1/employees`       | `createEmployee()`       | `Employee` JSON body  | `200 OK` — persisted `Employee` with generated `id` |
| `GET`    | `/api/v1/employees/{id}`  | `getEmployeeById()`      | none                  | `200 OK` — single `Employee` JSON; `404` if absent |
| `PUT`    | `/api/v1/employees/{id}`  | `updateEmployee()`       | `Employee` JSON body  | `200 OK` — updated `Employee` JSON; `404` if absent |
| `DELETE` | `/api/v1/employees/{id}`  | `deleteEmployee()`       | none                  | `200 OK` — `{ "deleted": true }`; `404` if absent |

**Payload structure (create / update request body):**

```json
{
  "firstName": "Jane",
  "lastName":  "Smith",
  "emailId":   "jane.smith@example.com"
}
```

The `id` field is never included in write payloads. For `POST`, the database assigns the ID and returns it in the response body. For `PUT`, the target `id` is taken exclusively from the URL path variable — the body `id` field, if present, is ignored.

**Error response (404):**

When `findById()` returns `Optional.empty()`, the `orElseThrow` lambda throws `ResourceNotFoundException`. Because the class is annotated `@ResponseStatus(HttpStatus.NOT_FOUND)`, Spring MVC automatically writes a `404` status with a default error body — no global exception handler is required.

---

## Component Reference

### `EmployeeService.js` — Axios API encapsulation

The central HTTP adapter. Constructed as a singleton class instance exported as the module's default export, meaning all components share one object and one configured base URL.

```
EMPLOYEE_API_BASE_URL = "http://localhost:8080/api/v1/employees"
```

| Method                              | HTTP Verb | URL Pattern                          | Purpose                         |
|-------------------------------------|-----------|--------------------------------------|---------------------------------|
| `getEmployees()`                    | `GET`     | `/employees`                         | Fetch all employee records      |
| `createEmployee(employee)`          | `POST`    | `/employees`                         | Persist a new employee          |
| `getEmployeeById(employeeId)`       | `GET`     | `/employees/{employeeId}`            | Fetch one employee by numeric ID |
| `updateEmployee(employee, id)`      | `PUT`     | `/employees/{id}`                    | Overwrite an employee's fields  |
| `deleteEmployee(employeeId)`        | `DELETE`  | `/employees/{employeeId}`            | Remove an employee by ID        |

All methods return raw Axios promise objects. Callers chain `.then()` to access `res.data`. No centralized error handling or interceptor is configured in V1.

---

### `ListEmployeeComponent.jsx` — List view with inline actions

**Route:** `/` and `/employees`

Fetches the full employee list in `componentDidMount()` and stores it in `this.state.employees` (initially `[]`). The render method maps the state array to Bootstrap table rows. Three action buttons per row dispatch `viewEmployee()`, `editEmployee()`, and `deleteEmployee()` via `this.props.history.push()` or `EmployeeService` calls.

**Delete flow:** `deleteEmployee(id)` calls `EmployeeService.deleteEmployee(id)`, then on resolution applies a `filter()` against the current state array to remove the deleted record — the component never re-fetches the list; it mutates the local state copy.

**Navigation:** `addEmployee()` pushes `/add-employee/_add`. `editEmployee(id)` pushes `/add-employee/{id}`. Both routes resolve to `CreateEmployeeComponent`; the sentinel `_add` vs. a real numeric ID controls create vs. update mode.

---

### `CreateEmployeeComponent.jsx` — Unified create / update form

**Route:** `/add-employee/:id`

Doubles as both the create form and the update form. The `id` path parameter is extracted in the constructor via `this.props.match.params.id` and stored in state.

**Mode detection in `componentDidMount()`:**
- `id === '_add'` → short-circuits immediately; the form starts empty.
- `id` is a numeric string → calls `EmployeeService.getEmployeeById(id)` and pre-populates state with the existing employee's fields.

**Submission via `saveOrUpdateEmployee()`:**
- `id === '_add'` → `EmployeeService.createEmployee(employee)` → `POST`
- otherwise → `EmployeeService.updateEmployee(employee, id)` → `PUT`

On resolution of either promise, `history.push('/employees')` returns the user to the list view.

**Controlled inputs:** `changeFirstNameHandler`, `changeLastNameHandler`, `changeEmailHandler` each call `setState()` on their respective field — React re-renders the input on every keystroke, keeping the DOM in sync with component state.

---

### `UpdateEmployeeComponent.jsx` — Dedicated update form

**Route:** Currently commented out in `App.js` (`{/* <Route path="/update-employee/:id" .../> */}`).

Functionally equivalent to the update branch of `CreateEmployeeComponent`. Extracts `id` from `props.match.params`, pre-populates fields via `getEmployeeById()` in `componentDidMount()`, and submits via `EmployeeService.updateEmployee()`. Exists as a standalone component but is not wired to an active route in the current build.

---

### `ViewEmployeeComponent.jsx` — Read-only detail view

**Route:** `/view-employee/:id`

Extracts the numeric `id` from `this.props.match.params.id` in the constructor and stores it alongside an empty `employee` object in state. `componentDidMount()` calls `EmployeeService.getEmployeeById(this.state.id)` and merges the response into `this.state.employee`. The render method accesses `this.state.employee.firstName`, `.lastName`, and `.emailId` directly inside Bootstrap card rows. No edit controls are rendered — this is a pure display component.

---

### `HeaderComponent.js` / `FooterComponent.jsx` — Layout scaffolding

Static presentational components. `HeaderComponent` renders the application navigation bar. `FooterComponent` renders the page footer. Neither component holds state or makes API calls. Both are rendered unconditionally by `App.js` outside the `<Switch>`, so they persist across all route transitions.

---

## Backend Engine

### Spring Application Bootstrap

`SpringbootBackendApplication.java` carries the `@SpringBootApplication` annotation, which is a meta-annotation composing `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`. On startup, Spring Boot auto-configures:

- A `DataSource` bean from `spring.datasource.*` properties
- A `LocalContainerEntityManagerFactoryBean` wrapping Hibernate as the JPA provider
- A `PlatformTransactionManager` backed by JPA
- Jackson `ObjectMapper` for JSON serialization of `@RestController` return values
- Embedded Tomcat server on port `8080`

No XML configuration exists. The entire application context is constructed from classpath scanning and auto-configuration conventions.

---

### `EmployeeRepository` — JPA boilerplate elimination

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
}
```

`JpaRepository<Employee, Long>` is a Spring Data interface parameterized on the entity type and its primary key type. Extending it with an empty body is sufficient to generate a full-featured repository proxy at runtime. The following operations are available without a single line of implementation code:

| Method                        | SQL generated                                   |
|-------------------------------|-------------------------------------------------|
| `findAll()`                   | `SELECT * FROM employees`                       |
| `findById(Long id)`           | `SELECT * FROM employees WHERE id = ?`         |
| `save(Employee e)`            | `INSERT INTO employees ...` or `UPDATE ... WHERE id = ?` (based on whether `id` is set) |
| `delete(Employee e)`          | `DELETE FROM employees WHERE id = ?`           |
| `existsById(Long id)`         | `SELECT COUNT(*) FROM employees WHERE id = ?`  |

Spring Data generates a concrete proxy class at application startup via `SimpleJpaRepository`. All operations run within a `@Transactional` context managed by the JPA `PlatformTransactionManager`.

---

### `ResourceNotFoundException` — API fault tolerance

```java
@ResponseStatus(value = HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

`ResourceNotFoundException` extends `RuntimeException`, making it an unchecked exception — callers are not required to declare or catch it. The `@ResponseStatus(HttpStatus.NOT_FOUND)` annotation instructs Spring MVC's `DefaultHandlerExceptionResolver` to translate any unhandled throw of this exception into an HTTP `404 Not Found` response automatically.

In `EmployeeController`, it is thrown inline via `Optional.orElseThrow()`:

```java
employeeRepository.findById(id)
    .orElseThrow(() -> new ResourceNotFoundException("Employee not exist with id :" + id));
```

This pattern avoids null checks and keeps controller methods single-expression for the lookup-or-fail path. The `serialVersionUID` field satisfies Java's `Serializable` contract inherited through the `RuntimeException` hierarchy.

---

## Configuration

### `springboot-backend/src/main/resources/application.properties`

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/employee_management_system?useSSL=false
spring.datasource.username=root
spring.datasource.password=root

spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL5InnoDBDialect

spring.jpa.hibernate.ddl-auto=update
```

| Property                                      | Value                              | Effect                                                                                     |
|-----------------------------------------------|------------------------------------|--------------------------------------------------------------------------------------------|
| `spring.datasource.url`                       | `jdbc:mysql://localhost:3306/...`  | JDBC connection string; `useSSL=false` disables SSL for local development                  |
| `spring.datasource.username` / `password`     | `root` / `root`                    | MySQL credentials — **must be changed** before any non-local deployment                    |
| `spring.jpa.properties.hibernate.dialect`     | `MySQL5InnoDBDialect`              | Tells Hibernate to emit MySQL 5 InnoDB-compatible SQL; affects DDL and DML generation      |
| `spring.jpa.hibernate.ddl-auto`               | `update`                           | Hibernate inspects the live schema on startup and applies additive changes; never drops data |

The server port is not explicitly set, so it defaults to `8080` per Spring Boot convention.

---

### `react-frontend/package.json` — key dependencies

| Package                  | Version   | Role                                                                                      |
|--------------------------|-----------|-------------------------------------------------------------------------------------------|
| `react`                  | `^16.13.1`| Core React library — virtual DOM diffing, class component lifecycle                       |
| `react-dom`              | `^16.13.1`| DOM renderer — mounts the React component tree into `index.html#root`                     |
| `react-router-dom`       | `^5.2.0`  | `BrowserRouter`, `Route`, `Switch` — client-side navigation without full-page reloads     |
| `axios`                  | `^0.19.2` | Promise-based HTTP client — used exclusively via `EmployeeService.js`                     |
| `bootstrap`              | `^4.5.0`  | CSS utility classes — table striping, card layout, button variants                         |
| `react-scripts`          | `3.4.1`   | Create React App build toolchain — Webpack, Babel, ESLint, dev server                     |

No CSS-in-JS, state management library (Redux, MobX), or test runner beyond the CRA defaults is included.

---

## Known Limitations

The following are known architectural gaps in this V1 release:

- **No authentication or authorization.** All API endpoints are publicly accessible. There is no JWT issuance, session management, or Spring Security configuration. Any client that can reach port `8080` can read, create, update, or delete any employee record.

- **No input validation.** The `Employee` entity has no `@NotNull`, `@Size`, or `@Email` constraints. The React form fields have no client-side validation either. Empty strings and malformed email addresses are accepted and persisted without error.

- **No pagination on the list view.** `getAllEmployees()` issues an unbounded `SELECT *`, and `ListEmployeeComponent` renders every record in a single table. As the dataset grows, both the API response time and the DOM render cost scale linearly.

- **No server-side error handling beyond 404.** Database connection failures, constraint violations, and unexpected exceptions surface as unformatted `500 Internal Server Error` responses. There is no `@ControllerAdvice` global exception handler producing structured error JSON.

- **Hard-coded API base URL.** `EmployeeService.js` contains `const EMPLOYEE_API_BASE_URL = "http://localhost:8080/api/v1/employees"` as a literal string. Deploying the frontend to any environment requires a manual source edit; there is no environment variable substitution via `.env` files.

- **CORS policy pinned to localhost.** `@CrossOrigin(origins = "http://localhost:3000")` will reject requests from any production domain as a pre-flight CORS failure.

- **No caching layer.** Each navigation to `/employees` triggers a fresh full-table database query. There is no HTTP cache header, Spring Cache abstraction, or client-side memoization.

- **`UpdateEmployeeComponent` is unreachable.** The `/update-employee/:id` route is commented out in `App.js`. The component exists in the source tree but is never rendered in the running application.

---

## Contributing

Bug reports, improvement proposals, and pull requests are welcome.

**Primary author and maintainer:**
AUM PETHANI — [aum.pethani@gmail.com](mailto:aum.pethani@gmail.com)

To contribute:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature-name`
3. Commit your changes: `git commit -m "describe your change"`
4. Push to your fork: `git push origin feature/your-feature-name`
5. Open a pull request against `main`

Please keep backend changes (Java/Spring) and frontend changes (React/JS) in logically separate commits. For any change that alters the REST API contract (endpoint paths, request/response shape), update both the controller and the relevant `EmployeeService.js` call in the same PR.

---

## License

This project is released under the [MIT License](https://opensource.org/licenses/MIT).

Copyright © 2024 AUM PETHANI
