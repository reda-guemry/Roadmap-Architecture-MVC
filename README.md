<aside>

A comprehensive guide to building a custom PHP MVC Framework from scratch. This documentation covers architectural decisions, design patterns, and best practices for creating a production-ready framework.

</aside>

---

# Phase 1: The Foundation (Core Structure)

## 1. MVC Architecture

### What is MVC?

MVC (Model-View-Controller) is an architectural pattern that separates an application into three interconnected components:

- **Model:** Handles data and business logic
- **View:** Manages the presentation layer (HTML, templates)
- **Controller:** Acts as the intermediary between Model and View, handling user input

### Why MVC?

- **Separation of Concerns:** Each component has a single, well-defined responsibility
- **Maintainability:** Changes to the UI don't affect business logic, and vice versa
- **Testability:** Each layer can be tested independently
- **Reusability:** Models and business logic can be reused across different views
- **Team Collaboration:** Developers can work on different layers simultaneously

### How it Works

1. User sends a request (e.g., clicks a button)
2. **Controller** receives the request
3. Controller interacts with the **Model** to retrieve or manipulate data
4. Controller passes data to the **View**
5. View renders the response and sends it back to the user

### Folder Structure (Best Practices)

```
/project-root
├── /app
│   ├── /Controllers      # Handle HTTP requests
│   ├── /Models
│   │   ├── /Entities     # Database table representations
│   │   ├── /DTOs         # Data Transfer Objects
│   │   └── /DAOs         # Data Access Objects
│   ├── /Services         # Business logic layer
│   ├── /Mappers          # Entity <-> DTO conversion
│   ├── /Middleware       # Request interceptors
│   └── /Helpers          # Utility classes
├── /core
│   ├── Router.php        # URL routing system
│   ├── Container.php     # Dependency Injection
│   ├── Database.php      # Database connection (Singleton)
│   ├── Request.php       # HTTP request wrapper
│   └── Response.php      # HTTP response wrapper
├── /config
│   └── .env              # Environment configuration
├── /public
│   └── index.php         # Application entry point
├── /views                # Template files
├── /vendor               # Composer dependencies
└── composer.json
```

<aside>
💡

**Key Principle:** Keep the public folder minimal. Only index.php and static assets (CSS, JS, images) should be accessible to the web server.

</aside>

---

## 2. The Entry Point

### What is index.php?

The **single entry point** of your application. Every HTTP request is routed through this file, which bootstraps the framework and dispatches requests to appropriate controllers.

### Why a Single Entry Point?

- **Security:** Only one file is exposed to the web, reducing attack surface
- **Centralized Configuration:** All initialization happens in one place
- **URL Rewriting:** Enables clean URLs (e.g., `/user/profile` instead of `/user.php?page=profile`)
- **Request Control:** All requests pass through middleware and security checks

### How it Works

**public/index.php:**

```php
<?php

// Load Composer's autoloader
require_once __DIR__ . '/../vendor/autoload.php';

// Load environment variables
$dotenv = Dotenv\Dotenv::createImmutable(__DIR__ . '/../config');
$dotenv->load();

// Initialize the DI Container
$container = new Core\Container();

// Create Request and Response objects
$request = new Core\Request();
$response = new Core\Response();

// Initialize the Router
$router = new Core\Router($container);

// Load routes
require_once __DIR__ . '/../routes/web.php';

// Dispatch the request
$router->dispatch($request, $response);
```

### Autoloading (PSR-4) & Composer

**What is PSR-4?**

PSR-4 is a PHP Standard Recommendation for autoloading classes from file paths. It maps namespaces to directory structures.

**Why Composer Autoloading?**

❌ **Without Autoloading:**

```php
require_once 'app/Controllers/UserController.php';
require_once 'app/Services/UserService.php';
require_once 'app/Models/DAOs/UserDAO.php';
require_once 'app/Models/Entities/User.php';
// ... hundreds of require statements
```

✅ **With Composer Autoloading:**

```php
// No require statements needed!
$user = new App\Models\Entities\User();
// Composer automatically finds and loads the class
```

**Benefits:**

- **Performance:** Classes are loaded only when needed (lazy loading)
- **Maintainability:** No manual dependency management
- **Standards Compliance:** Follows PHP-FIG standards

**composer.json Configuration:**

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "app/",
            "Core\\": "core/"
        }
    }
}
```

After defining this, run:

```bash
composer dump-autoload
```

<aside>
⚠️

**Never use manual require/include in modern PHP.** Autoloading is faster, cleaner, and prevents "class already defined" errors.

</aside>

---

# Phase 2: The Core Engine (System Logic)

## 3. The Router

### What is a Router?

A router maps incoming HTTP requests (URLs) to specific controller actions. It's the "traffic controller" of your application.

### Why Do We Need It?

- **Clean URLs:** Transform `/index.php?controller=user&action=show&id=5` into `/user/5`
- **RESTful Design:** Support different HTTP methods (GET, POST, PUT, DELETE)
- **Centralized Routing:** All routes defined in one place
- **Middleware Support:** Apply security checks before reaching controllers

### How it Works

**Basic Routing:**

```php
// routes/web.php
$router->get('/users', 'UserController@index');
$router->get('/user/{id}', 'UserController@show');
$router->post('/user', 'UserController@store');
$router->put('/user/{id}', 'UserController@update');
$router->delete('/user/{id}', 'UserController@destroy');
```

**Router Implementation (Simplified):**

```php
class Router {
    private array $routes = [];
    
    public function get(string $uri, string $action) {
        $this->addRoute('GET', $uri, $action);
    }
    
    public function post(string $uri, string $action) {
        $this->addRoute('POST', $uri, $action);
    }
    
    private function addRoute(string $method, string $uri, string $action) {
        // Convert {id} to regex pattern
        $pattern = preg_replace('/\{(\w+)\}/', '(?P<$1>[^/]+)', $uri);
        $pattern = '#^' . $pattern . '$#';
        
        $this->routes[$method][] = [
            'pattern' => $pattern,
            'action' => $action
        ];
    }
    
    public function dispatch(Request $request, Response $response) {
        $method = $request->getMethod();
        $uri = $request->getUri();
        
        foreach ($this->routes[$method] ?? [] as $route) {
            if (preg_match($route['pattern'], $uri, $matches)) {
                // Extract controller and method
                [$controller, $method] = explode('@', $route['action']);
                
                // Resolve controller from Container
                $controllerInstance = $this->container->resolve($controller);
                
                // Extract parameters (e.g., id => 5)
                $params = array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
                
                // Call the controller method
                return $controllerInstance->$method($request, $response, ...$params);
            }
        }
        
        // 404 Not Found
        $response->status(404)->send('Page not found');
    }
}
```

**Handling Parameters:**

When a URL like `/user/42` matches the pattern `/user/{id}`, the router extracts `id => 42` and passes it to the controller:

```php
class UserController {
    public function show(Request $request, Response $response, $id) {
        $user = $this->userService->findById($id);
        return $response->json($user);
    }
}
```

---

## 4. The Dependency Injection Container (The "Brain")

### What is a DI Container?

A Dependency Injection Container is a **smart factory** that automatically creates objects and resolves their dependencies. It's the "brain" of your framework.

### Why Do We Need It?

❌ **Without DI Container (Tight Coupling):**

```php
class UserController {
    private $userService;
    
    public function __construct() {
        $db = new Database(); // Manually creating dependencies
        $userDAO = new UserDAO($db);
        $this->userService = new UserService($userDAO);
    }
}
```

**Problems:**

- Hard to test (can't mock dependencies)
- Violates Single Responsibility Principle
- Changes to dependencies require changing every controller

✅ **With DI Container (Loose Coupling):**

```php
class UserController {
    private UserService $userService;
    
    // Container automatically injects UserService
    public function __construct(UserService $userService) {
        $this->userService = $userService;
    }
}
```

### How it Works

**Container Implementation:**

```php
class Container {
    private array $bindings = [];
    private array $instances = [];
    
    // Register a binding
    public function bind(string $abstract, callable $concrete) {
        $this->bindings[$abstract] = $concrete;
    }
    
    // Register a singleton
    public function singleton(string $abstract, callable $concrete) {
        $this->bindings[$abstract] = $concrete;
        $this->instances[$abstract] = null;
    }
    
    // Resolve a class
    public function resolve(string $abstract) {
        // Check if singleton instance exists
        if (isset($this->instances[$abstract]) && $this->instances[$abstract] !== null) {
            return $this->instances[$abstract];
        }
        
        // Check if binding exists
        if (isset($this->bindings[$abstract])) {
            $instance = call_user_func($this->bindings[$abstract], $this);
            
            // Store singleton instance
            if (array_key_exists($abstract, $this->instances)) {
                $this->instances[$abstract] = $instance;
            }
            
            return $instance;
        }
        
        // Auto-resolve using reflection
        return $this->autoResolve($abstract);
    }
    
    // Automatically resolve dependencies using reflection
    private function autoResolve(string $class) {
        $reflection = new ReflectionClass($class);
        $constructor = $reflection->getConstructor();
        
        if (!$constructor) {
            return new $class;
        }
        
        $parameters = $constructor->getParameters();
        $dependencies = [];
        
        foreach ($parameters as $parameter) {
            $type = $parameter->getType();
            
            if ($type && !$type->isBuiltin()) {
                // Recursively resolve dependencies
                $dependencies[] = $this->resolve($type->getName());
            }
        }
        
        return $reflection->newInstanceArgs($dependencies);
    }
}
```

**Usage Example:**

```php
// Bind Database as singleton
$container->singleton(Database::class, function($c) {
    return new Database($_ENV['DB_HOST'], $_ENV['DB_NAME']);
});

// Auto-resolve UserController (dependencies injected automatically)
$controller = $container->resolve(UserController::class);
```

### Connection to Factory Pattern

The Container **is** a Factory Pattern, but smarter:

- **Simple Factory:** Creates objects manually
- **DI Container:** Creates objects AND resolves their dependencies recursively

<aside>
🧠

**The Container is the brain because:** It knows how to build every class in your application and manages their lifecycles (singleton vs. transient).

</aside>

---

## 5. Request & Response Objects

### What Are They?

Wrapper objects that encapsulate HTTP request and response data, providing a clean, object-oriented interface.

### Why Not Use $_POST, $_GET, $_SERVER Directly?

❌ **Problems with Superglobals:**

```php
$username = $_POST['username'] ?? null; // No validation
$id = $_GET['id']; // Unsafe, no type checking
$method = $_SERVER['REQUEST_METHOD']; // Inconsistent access
```

**Issues:**

- **No Validation:** Raw input is dangerous (SQL injection, XSS)
- **Hard to Test:** Can't mock superglobals in unit tests
- **Poor Abstraction:** Scattered access throughout code
- **No Type Safety:** Everything is a string

✅ **With Request/Response Objects:**

```php
$username = $request->input('username'); // Sanitized
$id = $request->route('id'); // From URL parameters
$method = $request->getMethod(); // Clean interface
```

### How They Work

**Request Object:**

```php
class Request {
    private array $queryParams;
    private array $bodyParams;
    private array $routeParams;
    private array $files;
    private array $headers;
    private string $method;
    private string $uri;
    
    public function __construct() {
        $this->queryParams = $_GET;
        $this->bodyParams = $_POST;
        $this->files = $_FILES;
        $this->method = $_SERVER['REQUEST_METHOD'];
        $this->uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
        $this->headers = getallheaders();
    }
    
    // Get input from query or body
    public function input(string $key, $default = null) {
        return $this->bodyParams[$key] ?? $this->queryParams[$key] ?? $default;
    }
    
    // Get sanitized input
    public function sanitize(string $key, $default = null) {
        $value = $this->input($key, $default);
        return htmlspecialchars($value, ENT_QUOTES, 'UTF-8');
    }
    
    // Get route parameter (e.g., /user/{id})
    public function route(string $key, $default = null) {
        return $this->routeParams[$key] ?? $default;
    }
    
    public function getMethod(): string {
        return $this->method;
    }
    
    public function getUri(): string {
        return $this->uri;
    }
    
    public function header(string $key, $default = null) {
        return $this->headers[$key] ?? $default;
    }
}
```

**Response Object:**

```php
class Response {
    private int $statusCode = 200;
    private array $headers = [];
    
    public function status(int $code): self {
        $this->statusCode = $code;
        return $this;
    }
    
    public function header(string $key, string $value): self {
        $this->headers[$key] = $value;
        return $this;
    }
    
    public function json(array $data): void {
        $this->header('Content-Type', 'application/json');
        $this->send(json_encode($data));
    }
    
    public function send(string $content): void {
        http_response_code($this->statusCode);
        
        foreach ($this->headers as $key => $value) {
            header("$key: $value");
        }
        
        echo $content;
    }
    
    public function redirect(string $url): void {
        $this->status(302)->header('Location', $url)->send('');
    }
}
```

**Usage in Controller:**

```php
public function store(Request $request, Response $response) {
    $username = $request->sanitize('username');
    $email = $request->sanitize('email');
    
    $user = $this->userService->create($username, $email);
    
    return $response->status(201)->json([
        'message' => 'User created successfully',
        'user' => $user
    ]);
}
```

<aside>
🔒

**Security First:** Always sanitize user input. The Request object provides a safe layer between raw HTTP data and your application logic.

</aside>

---

# Phase 3: Database & Data Access (Crucial Decisions)

## 6. Database Connection

### What is the Singleton Pattern?

A design pattern that ensures **only one instance** of a class exists throughout the application lifecycle.

### Why Only One Database Connection?

**Problems with Multiple Connections:**

- **Resource Waste:** Each connection consumes memory and database resources
- **Connection Pool Exhaustion:** Databases limit concurrent connections
- **Inconsistent State:** Different connections may have different transaction states
- **Performance:** Opening/closing connections is expensive

✅ **Singleton Solution:**

```php
class Database {
    private static ?Database $instance = null;
    private PDO $connection;
    
    // Private constructor prevents direct instantiation
    private function __construct() {
        $host = $_ENV['DB_HOST'];
        $dbname = $_ENV['DB_NAME'];
        $username = $_ENV['DB_USER'];
        $password = $_ENV['DB_PASSWORD'];
        
        $dsn = "pgsql:host=$host;dbname=$dbname;charset=utf8mb4";
        
        $this->connection = new PDO($dsn, $username, $password, [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES => false
        ]);
    }
    
    // Get the single instance
    public static function getInstance(): Database {
        if (self::$instance === null) {
            self::$instance = new Database();
        }
        return self::$instance;
    }
    
    public function getConnection(): PDO {
        return $this->connection;
    }
    
    // Prevent cloning
    private function __clone() {}
    
    // Prevent unserialization
    public function __wakeup() {
        throw new Exception("Cannot unserialize singleton");
    }
}
```

### Configuration (.env file)

**Why .env?**

- **Security:** Credentials never committed to version control
- **Flexibility:** Different configs for dev, staging, production
- **12-Factor App Compliance:** Environment-based configuration

**.env:**

```
DB_HOST=[localhost](http://localhost)
DB_NAME=myapp_db
DB_USER=postgres
DB_PASSWORD=secret123
DB_PORT=5432
```

**Load with Composer:**

```bash
composer require vlucas/phpdotenv
```

---

## 7. DAO Pattern (Data Access Object)

### What is DAO?

A pattern that provides an **abstract interface to the database**, separating data access logic from business logic.

### DAO vs Repository: Why We Choose DAO

<aside>
⚠️

**Important Architectural Decision:** We are using **DAO** and **explicitly NOT using the Repository Pattern** to avoid over-engineering.

</aside>

**Repository Pattern (What We're NOT Using):**

```php
interface UserRepositoryInterface {
    public function find(int $id): User;
    public function findByEmail(string $email): User;
    public function findAllActive(): array;
    public function save(User $user): void;
}

class UserRepository implements UserRepositoryInterface {
    // Collection-like interface
    // Pretends database is an in-memory collection
    // Requires complex query builders
}
```

**DAO Pattern (What We're Using):**

```php
class UserDAO {
    private PDO $db;
    
    public function __construct(Database $database) {
        $this->db = $database->getConnection();
    }
    
    // Direct SQL execution
    public function insert(User $user): int {
        $sql = "INSERT INTO users (username, email) VALUES (:username, :email)";
        $stmt = $this->db->prepare($sql);
        $stmt->execute([
            'username' => $user->getUsername(),
            'email' => $user->getEmail()
        ]);
        return (int) $this->db->lastInsertId();
    }
    
    public function findById(int $id): ?User {
        $sql = "SELECT * FROM users WHERE id = :id";
        $stmt = $this->db->prepare($sql);
        $stmt->execute(['id' => $id]);
        
        $row = $stmt->fetch();
        return $row ? $this->mapToEntity($row) : null;
    }
    
    public function update(User $user): bool {
        $sql = "UPDATE users SET username = :username, email = :email WHERE id = :id";
        $stmt = $this->db->prepare($sql);
        return $stmt->execute([
            'id' => $user->getId(),
            'username' => $user->getUsername(),
            'email' => $user->getEmail()
        ]);
    }
    
    public function delete(int $id): bool {
        $sql = "DELETE FROM users WHERE id = :id";
        $stmt = $this->db->prepare($sql);
        return $stmt->execute(['id' => $id]);
    }
    
    private function mapToEntity(array $row): User {
        $user = new User();
        $user->setId($row['id']);
        $user->setUsername($row['username']);
        $user->setEmail($row['email']);
        return $user;
    }
}
```

### Why DAO Over Repository?

| Aspect | Repository Pattern | DAO Pattern (Our Choice) |
| --- | --- | --- |
| **Complexity** | High (requires query builders, specifications) | Low (direct SQL) |
| **Learning Curve** | Steep | Gentle |
| **Flexibility** | Abstract (hard to optimize queries) | Direct SQL control |
| **Performance** | Query builder overhead | Optimized raw SQL |
| **Use Case** | Large enterprise apps with domain-driven design | Custom frameworks, smaller to medium apps |

**Our Reasoning:**

- We're building a **custom framework**, not using Doctrine or Eloquent
- We want **full control** over SQL queries
- We don't need the abstraction layer that Repository provides
- **DAO is simpler** and easier to understand for learning purposes
- We can always migrate to Repository later if needed

---

## 8. Entities

### What is an Entity?

A PHP class that **mirrors a database table**. One table = one Entity class.

### Why Entities?

- **Type Safety:** Use objects instead of associative arrays
- **Encapsulation:** Hide data access behind getters/setters
- **Validation:** Enforce business rules
- **IDE Support:** Autocomplete and type hints

### How They Work

**Database Table:**

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Entity Class:**

```php
class User {
    private ?int $id = null;
    private string $username;
    private string $email;
    private ?DateTime $createdAt = null;
    
    // Getters
    public function getId(): ?int {
        return $this->id;
    }
    
    public function getUsername(): string {
        return $this->username;
    }
    
    public function getEmail(): string {
        return $this->email;
    }
    
    public function getCreatedAt(): ?DateTime {
        return $this->createdAt;
    }
    
    // Setters
    public function setId(int $id): void {
        $this->id = $id;
    }
    
    public function setUsername(string $username): void {
        if (strlen($username) < 3) {
            throw new InvalidArgumentException("Username must be at least 3 characters");
        }
        $this->username = $username;
    }
    
    public function setEmail(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException("Invalid email format");
        }
        $this->email = $email;
    }
    
    public function setCreatedAt(DateTime $createdAt): void {
        $this->createdAt = $createdAt;
    }
}
```

**Usage:**

```php
$user = new User();
$user->setUsername('john_doe');
$user->setEmail('[john@example.com](mailto:john@example.com)');

$userDAO->insert($user);
```

---

## 9. Handling Joins & Complex Data

### The Problem: Entities Can't Handle JOINs

**Scenario:** Display users with their posts count and latest post title.

**SQL Query:**

```sql
SELECT 
    [u.id](http://u.id), 
    u.username, 
    [u.email](http://u.email), 
    COUNT([p.id](http://p.id)) as post_count,
    MAX(p.title) as latest_post_title
FROM users u
LEFT JOIN posts p ON [u.id](http://u.id) = p.user_id
GROUP BY [u.id](http://u.id), u.username, [u.email](http://u.email);
```

**Why Entity Doesn't Work:**

```php
// User Entity doesn't have post_count or latest_post_title properties!
class User {
    private int $id;
    private string $username;
    private string $email;
    // ❌ No place for post_count or latest_post_title
}
```

### The Solution: DTOs (Data Transfer Objects)

**What is a DTO?**

A simple object designed to carry data between processes, specifically shaped for **view requirements**.

**Entity vs DTO:**

| Aspect | Entity | DTO |
| --- | --- | --- |
| **Purpose** | Represents a database table | Represents view requirements |
| **Structure** | Matches table columns exactly | Contains any data the view needs |
| **Lifecycle** | Persistent (saved to database) | Temporary (used for data transfer) |
| **Logic** | May contain validation | Pure data container (no logic) |

**DTO Example:**

```php
class UserWithPostsDTO {
    public int $id;
    public string $username;
    public string $email;
    public int $postCount;
    public ?string $latestPostTitle;
    
    public function __construct(
        int $id,
        string $username,
        string $email,
        int $postCount,
        ?string $latestPostTitle
    ) {
        $this->id = $id;
        $this->username = $username;
        $this->email = $email;
        $this->postCount = $postCount;
        $this->latestPostTitle = $latestPostTitle;
    }
}
```

**Using DTO in DAO:**

```php
class UserDAO {
    public function findAllWithPosts(): array {
        $sql = "
            SELECT 
                [u.id](http://u.id), 
                u.username, 
                [u.email](http://u.email), 
                COUNT([p.id](http://p.id)) as post_count,
                MAX(p.title) as latest_post_title
            FROM users u
            LEFT JOIN posts p ON [u.id](http://u.id) = p.user_id
            GROUP BY [u.id](http://u.id), u.username, [u.email](http://u.email)
        ";
        
        $stmt = $this->db->query($sql);
        $results = [];
        
        while ($row = $stmt->fetch()) {
            $results[] = new UserWithPostsDTO(
                $row['id'],
                $row['username'],
                $row['email'],
                $row['post_count'],
                $row['latest_post_title']
            );
        }
        
        return $results;
    }
}
```

<aside>
🎯

**Key Principle:** Use **Entities** for database persistence (CRUD). Use **DTOs** for complex queries and view-specific data.

</aside>

---

## 10. ORM Concept

### What is ORM?

**Object-Relational Mapping** is a technique that lets you interact with a database using objects instead of SQL.

### The Problem ORM Solves

**Without ORM (Raw SQL):**

```php
$sql = "INSERT INTO users (username, email) VALUES (:username, :email)";
$stmt = $pdo->prepare($sql);
$stmt->execute(['username' => 'john', 'email' => '[john@example.com](mailto:john@example.com)']);

$sql = "SELECT * FROM users WHERE id = :id";
$stmt = $pdo->prepare($sql);
$stmt->execute(['id' => 1]);
$row = $stmt->fetch();
```

**With ORM (Object-Oriented):**

```php
$user = new User();
$user->setUsername('john');
$user->setEmail('[john@example.com](mailto:john@example.com)');
$user->save(); // ORM generates INSERT query

$user = User::find(1); // ORM generates SELECT query
```

### Full ORM vs Our BaseDAO (Mini-ORM)

**Full ORM (Doctrine, Eloquent):**

- Complete abstraction over SQL
- Automatic migrations
- Relationship management
- Query builders
- **Heavy and complex**

**Our BaseDAO (Mini-ORM):**

- Handles basic CRUD automatically
- Reduces boilerplate for simple queries
- Still allows raw SQL for complex queries
- **Lightweight and educational**

### BaseDAO Implementation

```php
abstract class BaseDAO {
    protected PDO $db;
    protected string $table;
    protected string $entityClass;
    
    public function __construct(Database $database) {
        $this->db = $database->getConnection();
    }
    
    // Generic find by ID
    public function findById(int $id): ?object {
        $sql = "SELECT * FROM {$this->table} WHERE id = :id";
        $stmt = $this->db->prepare($sql);
        $stmt->execute(['id' => $id]);
        
        $row = $stmt->fetch();
        return $row ? $this->hydrate($row) : null;
    }
    
    // Generic find all
    public function findAll(): array {
        $sql = "SELECT * FROM {$this->table}";
        $stmt = $this->db->query($sql);
        
        $results = [];
        while ($row = $stmt->fetch()) {
            $results[] = $this->hydrate($row);
        }
        return $results;
    }
    
    // Generic insert
    public function insert(object $entity): int {
        $data = $this->extract($entity);
        $columns = implode(', ', array_keys($data));
        $placeholders = ':' . implode(', :', array_keys($data));
        
        $sql = "INSERT INTO {$this->table} ($columns) VALUES ($placeholders)";
        $stmt = $this->db->prepare($sql);
        $stmt->execute($data);
        
        return (int) $this->db->lastInsertId();
    }
    
    // Generic update
    public function update(object $entity): bool {
        $data = $this->extract($entity);
        $setParts = [];
        
        foreach (array_keys($data) as $column) {
            if ($column !== 'id') {
                $setParts[] = "$column = :$column";
            }
        }
        
        $setClause = implode(', ', $setParts);
        $sql = "UPDATE {$this->table} SET $setClause WHERE id = :id";
        
        $stmt = $this->db->prepare($sql);
        return $stmt->execute($data);
    }
    
    // Generic delete
    public function delete(int $id): bool {
        $sql = "DELETE FROM {$this->table} WHERE id = :id";
        $stmt = $this->db->prepare($sql);
        return $stmt->execute(['id' => $id]);
    }
    
    // Convert database row to Entity (must be implemented by child)
    abstract protected function hydrate(array $row): object;
    
    // Convert Entity to database array (must be implemented by child)
    abstract protected function extract(object $entity): array;
}
```

**Using BaseDAO:**

```php
class UserDAO extends BaseDAO {
    protected string $table = 'users';
    protected string $entityClass = User::class;
    
    protected function hydrate(array $row): User {
        $user = new User();
        $user->setId($row['id']);
        $user->setUsername($row['username']);
        $user->setEmail($row['email']);
        return $user;
    }
    
    protected function extract(object $entity): array {
        return [
            'id' => $entity->getId(),
            'username' => $entity->getUsername(),
            'email' => $entity->getEmail()
        ];
    }
    
    // Custom query not covered by BaseDAO
    public function findByEmail(string $email): ?User {
        $sql = "SELECT * FROM users WHERE email = :email";
        $stmt = $this->db->prepare($sql);
        $stmt->execute(['email' => $email]);
        
        $row = $stmt->fetch();
        return $row ? $this->hydrate($row) : null;
    }
}
```

**Benefits of Our Mini-ORM:**

- ✅ No repetitive CRUD code
- ✅ Type safety with Entities
- ✅ Flexibility to write custom SQL when needed
- ✅ Learning-friendly (you see exactly what's happening)

<aside>
💡

**Best of Both Worlds:** BaseDAO eliminates boilerplate for simple operations, but you can still write raw SQL for complex queries.

</aside>

---

# Phase 4: Business Logic & Data Transformation

## 11. Services Layer

### What is a Service?

The **business logic layer** that sits between Controllers and DAOs. Services orchestrate operations and enforce business rules.

### Why Services?

❌ **Without Services (Fat Controllers):**

```php
class UserController {
    public function register(Request $request, Response $response) {
        // Validation
        if (!filter_var($request->input('email'), FILTER_VALIDATE_EMAIL)) {
            return $response->status(400)->json(['error' => 'Invalid email']);
        }
        
        // Business logic
        $user = new User();
        $user->setUsername($request->input('username'));
        $user->setEmail($request->input('email'));
        $user->setPassword(password_hash($request->input('password'), PASSWORD_BCRYPT));
        
        // Database access
        $userDAO = new UserDAO(Database::getInstance());
        $userId = $userDAO->insert($user);
        
        // Send email
        mail($user->getEmail(), 'Welcome!', 'Thanks for registering');
        
        return $response->json(['id' => $userId]);
    }
}
```

**Problems:**

- Controller is doing too much (violates Single Responsibility)
- Business logic is not reusable
- Hard to test
- Can't use the same logic in CLI commands or API endpoints

✅ **With Services (Thin Controllers):**

```php
class UserService {
    private UserDAO $userDAO;
    private EmailService $emailService;
    
    public function __construct(UserDAO $userDAO, EmailService $emailService) {
        $this->userDAO = $userDAO;
        $this->emailService = $emailService;
    }
    
    public function register(string $username, string $email, string $password): User {
        // Validation
        $this->validateEmail($email);
        $this->validatePassword($password);
        
        // Check if user exists
        if ($this->userDAO->findByEmail($email)) {
            throw new Exception('Email already registered');
        }
        
        // Create user
        $user = new User();
        $user->setUsername($username);
        $user->setEmail($email);
        $user->setPassword(password_hash($password, PASSWORD_BCRYPT));
        
        // Save to database
        $userId = $this->userDAO->insert($user);
        $user->setId($userId);
        
        // Send welcome email
        $this->emailService->sendWelcomeEmail($user);
        
        return $user;
    }
    
    private function validateEmail(string $email): void {
        if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidArgumentException('Invalid email format');
        }
    }
    
    private function validatePassword(string $password): void {
        if (strlen($password) < 8) {
            throw new InvalidArgumentException('Password must be at least 8 characters');
        }
    }
}
```

**Controller becomes thin:**

```php
class UserController {
    private UserService $userService;
    
    public function __construct(UserService $userService) {
        $this->userService = $userService;
    }
    
    public function register(Request $request, Response $response) {
        try {
            $user = $this->userService->register(
                $request->input('username'),
                $request->input('email'),
                $request->input('password')
            );
            
            return $response->status(201)->json([
                'message' => 'User registered successfully',
                'user_id' => $user->getId()
            ]);
        } catch (Exception $e) {
            return $response->status(400)->json(['error' => $e->getMessage()]);
        }
    }
}
```

### Service Layer Responsibilities

- **Validation:** Check business rules
- **Orchestration:** Coordinate multiple DAOs
- **Calculations:** Perform business calculations
- **Transactions:** Manage database transactions
- **External Services:** Call APIs, send emails, etc.

<aside>
🎯

**Golden Rule:** Controllers handle HTTP. Services handle business logic. DAOs handle data access.

</aside>

---

## 12. The Mapper Pattern

### What is a Mapper?

A class responsible for **converting between Entities and DTOs**. It keeps transformation logic separate from Controllers and Entities.

### Why Mappers?

❌ **Without Mapper (Cluttered Controller):**

```php
class UserController {
    public function index(Request $request, Response $response) {
        $users = $this->userDAO->findAllWithPosts();
        
        // Transformation logic in controller
        $dtos = [];
        foreach ($users as $user) {
            $dtos[] = [
                'id' => $user->getId(),
                'username' => $user->getUsername(),
                'email' => $user->getEmail(),
                'display_name' => ucfirst($user->getUsername()),
                'post_count' => $user->getPostCount()
            ];
        }
        
        return $response->json($dtos);
    }
}
```

✅ **With Mapper (Clean Separation):**

```php
class UserMapper {
    public function toDTO(User $user): UserDTO {
        return new UserDTO(
            $user->getId(),
            $user->getUsername(),
            $user->getEmail(),
            ucfirst($user->getUsername()) // Display name transformation
        );
    }
    
    public function toDTOList(array $users): array {
        return array_map([$this, 'toDTO'], $users);
    }
    
    public function toEntity(UserDTO $dto): User {
        $user = new User();
        $user->setId($dto->id);
        $user->setUsername($dto->username);
        $user->setEmail($dto->email);
        return $user;
    }
    
    public function toDetailDTO(User $user, int $postCount): UserDetailDTO {
        return new UserDetailDTO(
            $user->getId(),
            $user->getUsername(),
            $user->getEmail(),
            $postCount
        );
    }
}
```

**Controller becomes clean:**

```php
class UserController {
    private UserService $userService;
    private UserMapper $userMapper;
    
    public function __construct(UserService $userService, UserMapper $userMapper) {
        $this->userService = $userService;
        $this->userMapper = $userMapper;
    }
    
    public function index(Request $request, Response $response) {
        $users = $this->userService->findAll();
        $dtos = $this->userMapper->toDTOList($users);
        
        return $response->json($dtos);
    }
}
```

### When to Use Mappers

- Converting Entity → DTO for API responses
- Converting DTO → Entity from API requests
- Transforming data shapes (e.g., renaming fields, formatting dates)
- Aggregating data from multiple entities

<aside>
🧹

**Clean Code:** Mappers keep Controllers focused on HTTP and Entities focused on data. All transformation logic lives in one place.

</aside>

---

## 13. Helpers

### What are Helpers?

**Static utility classes** that provide reusable functions for common tasks that don't fit into any specific layer.

### Why Helpers?

- **Reusability:** Use the same function across Controllers, Services, Views
- **DRY Principle:** Don't Repeat Yourself
- **Namespace Pollution:** Avoid global functions

### How They Work

**TextHelper:**

```php
class TextHelper {
    public static function truncate(string $text, int $length = 100): string {
        if (strlen($text) <= $length) {
            return $text;
        }
        return substr($text, 0, $length) . '...';
    }
    
    public static function slugify(string $text): string {
        $text = strtolower($text);
        $text = preg_replace('/[^a-z0-9]+/', '-', $text);
        return trim($text, '-');
    }
    
    public static function sanitizeHtml(string $html): string {
        return htmlspecialchars($html, ENT_QUOTES, 'UTF-8');
    }
}
```

**DateHelper:**

```php
class DateHelper {
    public static function formatDate(DateTime $date, string $format = 'Y-m-d'): string {
        return $date->format($format);
    }
    
    public static function humanReadable(DateTime $date): string {
        $now = new DateTime();
        $diff = $now->diff($date);
        
        if ($diff->days === 0) {
            return 'Today';
        } elseif ($diff->days === 1) {
            return 'Yesterday';
        } elseif ($diff->days < 7) {
            return $diff->days . ' days ago';
        } else {
            return $date->format('M d, Y');
        }
    }
    
    public static function isWeekend(DateTime $date): bool {
        return in_array($date->format('N'), [6, 7]); // Saturday, Sunday
    }
}
```

**ArrayHelper:**

```php
class ArrayHelper {
    public static function pluck(array $array, string $key): array {
        return array_map(fn($item) => $item[$key] ?? null, $array);
    }
    
    public static function groupBy(array $array, string $key): array {
        $result = [];
        foreach ($array as $item) {
            $groupKey = $item[$key] ?? 'undefined';
            $result[$groupKey][] = $item;
        }
        return $result;
    }
}
```

**Usage:**

```php
// In a Controller
$slug = TextHelper::slugify($request->input('title'));

// In a View
$summary = TextHelper::truncate($post->getContent(), 200);

// In a Service
$formattedDate = DateHelper::humanReadable($user->getCreatedAt());
```

### Helpers vs Services

| Aspect | Helpers | Services |
| --- | --- | --- |
| **State** | Stateless (static methods) | Stateful (injected dependencies) |
| **Dependencies** | None | DAOs, other Services |
| **Purpose** | Simple utility functions | Business logic orchestration |
| **Example** | Format date, slugify string | Register user, process payment |

<aside>
⚙️

**When to use Helpers:** For simple, stateless operations that don't require dependencies. If it needs a database or other services, use a Service class instead.

</aside>

---

# Phase 5: Security & Flow Control

## 14. Middleware

### What is Middleware?

Middleware are **"security guards"** that intercept HTTP requests before they reach Controllers. They check conditions and can allow, block, or modify requests.

### Why Middleware?

- **Centralized Security:** Authentication and authorization in one place
- **DRY Principle:** Don't repeat auth checks in every controller
- **Request Modification:** Add data to requests (e.g., authenticated user)
- **Response Modification:** Add headers, logging, etc.

### How it Works

**Flow Without Middleware:**

```
Request → Router → Controller → Response
```

**Flow With Middleware:**

```
Request → Middleware 1 → Middleware 2 → Router → Controller → Response
                ↓                ↓
              Block            Block
```

### Middleware Implementation

**Middleware Interface:**

```php
interface MiddlewareInterface {
    public function handle(Request $request, Response $response, callable $next);
}
```

**AuthMiddleware (Check if user is logged in):**

```php
class AuthMiddleware implements MiddlewareInterface {
    public function handle(Request $request, Response $response, callable $next) {
        session_start();
        
        // Check if user is authenticated
        if (!isset($_SESSION['user_id'])) {
            return $response->status(401)->json([
                'error' => 'Unauthorized. Please log in.'
            ]);
        }
        
        // Add authenticated user to request
        $request->setAttribute('user_id', $_SESSION['user_id']);
        
        // Continue to next middleware or controller
        return $next($request, $response);
    }
}
```

**RoleMiddleware (Check if user has required role):**

```php
class RoleMiddleware implements MiddlewareInterface {
    private string $requiredRole;
    private UserDAO $userDAO;
    
    public function __construct(string $requiredRole, UserDAO $userDAO) {
        $this->requiredRole = $requiredRole;
        $this->userDAO = $userDAO;
    }
    
    public function handle(Request $request, Response $response, callable $next) {
        $userId = $request->getAttribute('user_id');
        $user = $this->userDAO->findById($userId);
        
        if (!$user || $user->getRole() !== $this->requiredRole) {
            return $response->status(403)->json([
                'error' => 'Forbidden. You do not have permission to access this resource.'
            ]);
        }
        
        return $next($request, $response);
    }
}
```

**CORS Middleware (Allow cross-origin requests):**

```php
class CorsMiddleware implements MiddlewareInterface {
    public function handle(Request $request, Response $response, callable $next) {
        $response->header('Access-Control-Allow-Origin', '*');
        $response->header('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');
        $response->header('Access-Control-Allow-Headers', 'Content-Type, Authorization');
        
        // Handle preflight requests
        if ($request->getMethod() === 'OPTIONS') {
            return $response->status(200)->send('');
        }
        
        return $next($request, $response);
    }
}
```

### Applying Middleware to Routes

**In Router:**

```php
class Router {
    public function get(string $uri, string $action, array $middleware = []) {
        $this->addRoute('GET', $uri, $action, $middleware);
    }
    
    private function addRoute(string $method, string $uri, string $action, array $middleware) {
        // Store middleware with route
        $this->routes[$method][] = [
            'pattern' => $this->uriToPattern($uri),
            'action' => $action,
            'middleware' => $middleware
        ];
    }
    
    public function dispatch(Request $request, Response $response) {
        // Find matching route
        $route = $this->findRoute($request);
        
        if (!$route) {
            return $response->status(404)->send('Page not found');
        }
        
        // Build middleware pipeline
        $middleware = $route['middleware'];
        $controller = $this->resolveController($route['action']);
        
        // Execute middleware chain
        return $this->executeMiddleware($middleware, $request, $response, $controller);
    }
    
    private function executeMiddleware(array $middleware, Request $request, Response $response, callable $controller) {
        $next = function($request, $response) use ($controller) {
            return $controller($request, $response);
        };
        
        // Build chain in reverse order
        foreach (array_reverse($middleware) as $mw) {
            $next = function($request, $response) use ($mw, $next) {
                return $mw->handle($request, $response, $next);
            };
        }
        
        return $next($request, $response);
    }
}
```

**Route Definition:**

```php
// Public route (no middleware)
$router->get('/home', 'HomeController@index');

// Protected route (requires authentication)
$router->get('/dashboard', 'DashboardController@index', [
    new AuthMiddleware()
]);

// Admin route (requires authentication + admin role)
$router->get('/admin/users', 'AdminController@users', [
    new AuthMiddleware(),
    new RoleMiddleware('admin', $userDAO)
]);
```

### Common Middleware Examples

- **AuthMiddleware:** Verify user is logged in
- **RoleMiddleware:** Check user permissions
- **CorsMiddleware:** Handle cross-origin requests
- **RateLimitMiddleware:** Prevent abuse (limit requests per minute)
- **LoggingMiddleware:** Log all requests
- **CsrfMiddleware:** Prevent CSRF attacks
- **CompressionMiddleware:** Compress responses

<aside>
🛡️

**Security First:** Always use middleware for authentication and authorization. Never trust user input or assume a user is who they claim to be.

</aside>

---

# Phase 6: Design Patterns Reference

## 15. Summary of Patterns Used

This framework uses several classic design patterns. Here's a complete reference of where and why each pattern is used.

---

### Singleton Pattern

**Used in:** Database, Config

**Purpose:** Ensure only one instance exists throughout the application.

**Example:**

```php
class Database {
    private static ?Database $instance = null;
    
    private function __construct() {
        // Single database connection
    }
    
    public static function getInstance(): Database {
        if (self::$instance === null) {
            self::$instance = new Database();
        }
        return self::$instance;
    }
}
```

**Why:**

- **Database:** Only one connection needed to prevent resource waste
- **Config:** Configuration should be loaded once and shared globally

**Benefits:**

- Memory efficiency
- Consistent state
- Global access point

---

### Factory Method Pattern

**Used in:** View rendering, Dependency Injection Container

**Purpose:** Create objects without specifying their exact class.

**Example (View Factory):**

```php
class ViewFactory {
    public static function render(string $template, array $data = []): string {
        // Factory decides which view engine to use
        if (file_exists("views/$template.blade.php")) {
            return (new BladeEngine())->render($template, $data);
        } elseif (file_exists("views/$template.twig")) {
            return (new TwigEngine())->render($template, $data);
        } else {
            return (new PhpEngine())->render($template, $data);
        }
    }
}
```

**Example (Container as Factory):**

```php
// Container is a smart factory
$controller = $container->resolve(UserController::class);
// Container automatically creates UserController with all dependencies
```

**Why:**

- **Flexibility:** Switch implementations without changing client code
- **Encapsulation:** Hide complex creation logic
- **Extensibility:** Easy to add new types

---

### DAO (Data Access Object) Pattern

**Used in:** All database interactions

**Purpose:** Abstract and encapsulate all database access.

**Example:**

```php
class UserDAO {
    public function findById(int $id): ?User { /* SQL logic */ }
    public function insert(User $user): int { /* SQL logic */ }
    public function update(User $user): bool { /* SQL logic */ }
    public function delete(int $id): bool { /* SQL logic */ }
}
```

**Why:**

- **Separation of Concerns:** Database logic separate from business logic
- **Flexibility:** Can swap database engines without affecting Services
- **Testability:** Can mock DAO in tests

<aside>
⚠️

**Important:** We chose DAO over Repository Pattern to avoid over-engineering. DAO gives us direct SQL control while still maintaining clean architecture.

</aside>

---

### DTO (Data Transfer Object) Pattern

**Used in:** Complex queries, API responses

**Purpose:** Transfer data between layers without using Entities.

**Entity (Database structure):**

```php
class User {
    private int $id;
    private string $username;
    private string $email;
    private string $passwordHash;
    private DateTime $createdAt;
    // Getters and setters...
}
```

**DTO (View requirement):**

```php
class UserDTO {
    public int $id;
    public string $username;
    public string $email;
    public string $displayName;
    public int $postCount;
    // No password hash (security)
    // Includes computed fields (displayName, postCount)
}
```

**Why:**

- **Security:** Don't expose sensitive fields (like passwordHash)
- **Flexibility:** Include computed or joined data
- **Performance:** Only transfer what the view needs
- **Clean Separation:** View requirements separate from database structure

**When to Use:**

- API responses (don't expose internal Entities)
- Complex JOIN queries (Entities can't represent joined data)
- Forms with nested data

---

### Dependency Injection Pattern

**Used in:** All Controllers, Services, DAOs

**Purpose:** Pass dependencies through constructors instead of creating them internally.

❌ **Without DI:**

```php
class UserController {
    private $userService;
    
    public function __construct() {
        $db = new Database();
        $dao = new UserDAO($db);
        $this->userService = new UserService($dao);
    }
}
```

✅ **With DI:**

```php
class UserController {
    private UserService $userService;
    
    public function __construct(UserService $userService) {
        $this->userService = $userService;
    }
}
```

**Why:**

- **Testability:** Easy to mock dependencies in tests
- **Flexibility:** Swap implementations without changing code
- **Loose Coupling:** Classes don't depend on concrete implementations

---

### MVC (Model-View-Controller) Pattern

**Used in:** Entire application structure

**Purpose:** Separate application into three interconnected components.

```
┌─────────┐      ┌────────────┐      ┌───────┐
│  View   │◄─────│ Controller │◄─────│ Model │
└─────────┘      └────────────┘      └───────┘
   (HTML)         (Routing,            (Business
                   HTTP)                Logic, Data)
```

**Example:**

```php
// Model (Service + DAO + Entity)
$users = $userService->findAll();

// Controller (orchestration)
class UserController {
    public function index(Request $request, Response $response) {
        $users = $this->userService->findAll();
        return ViewFactory::render('users/index', ['users' => $users]);
    }
}

// View (template)
foreach ($users as $user) {
    echo "<li>" . $user->getUsername() . "</li>";
}
```

---

### Middleware Pattern (Chain of Responsibility)

**Used in:** Request/response pipeline

**Purpose:** Pass a request through a chain of handlers, each performing a specific task.

```
Request → Auth → Role → CORS → Controller → Response
           ↓      ↓      ↓
         Block  Block   Add Headers
```

**Example:**

```php
interface MiddlewareInterface {
    public function handle(Request $request, Response $response, callable $next);
}

class AuthMiddleware implements MiddlewareInterface {
    public function handle(Request $request, Response $response, callable $next) {
        if (!$this->isAuthenticated($request)) {
            return $response->status(401)->json(['error' => 'Unauthorized']);
        }
        return $next($request, $response); // Pass to next in chain
    }
}
```

**Why:**

- **Separation of Concerns:** Each middleware has one job
- **Flexibility:** Add/remove middleware without affecting others
- **Reusability:** Same middleware can be applied to multiple routes

---

## Pattern Summary Table

| Pattern | Where Used | Primary Benefit |
| --- | --- | --- |
| **Singleton** | Database, Config | Single instance, resource efficiency |
| **Factory Method** | Container, ViewFactory | Flexible object creation |
| **DAO** | Data Access Layer | Database abstraction |
| **DTO** | Data Transfer Layer | Clean separation, security |
| **Dependency Injection** | All classes | Testability, loose coupling |
| **MVC** | Application Structure | Separation of concerns |
| **Middleware (Chain of Responsibility)** | Request Pipeline | Modular request processing |

<aside>
🎓

**Learning Tip:** You don't need to memorize pattern names. Focus on understanding *why* each pattern solves a specific problem. The names will become natural over time.

</aside>

---

# Conclusion

You now have a complete roadmap for building a custom PHP MVC framework from scratch. This documentation covers:

✅ **Foundation:** MVC architecture, folder structure, entry point, autoloading

✅ **Core Engine:** Router, DI Container, Request/Response abstraction

✅ **Database Layer:** Singleton connection, DAO pattern, Entities, DTOs, Mini-ORM

✅ **Business Logic:** Services, Mappers, Helpers

✅ **Security:** Middleware for authentication and authorization

✅ **Design Patterns:** Practical application of Singleton, Factory, DAO, DTO, DI, and Middleware

<aside>
🚀

**Next Steps:** Start implementing each component step by step. Begin with the Router and Request/Response objects, then build up to the Database layer and Services. Don't try to build everything at once—take it one phase at a time.

</aside>

<aside>
📚

**Remember:** This framework prioritizes **simplicity and learning** over enterprise complexity. We chose DAO over Repository, avoided heavy abstractions, and kept direct SQL control. As your needs grow, you can always refactor toward more complex patterns.

</aside>
