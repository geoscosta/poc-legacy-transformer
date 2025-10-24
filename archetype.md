# .Net Application Archetype Guide

## Overview

This is an ASP.NET Core 8.0 Web API archetype using C#, PostgreSQL, Entity Framework Core, FluentMigrator (or EF Migrations), and a clean layered architecture. It follows best practices for separation of concerns and scalability.

## Project Structure

```text
src/
├── DemoApi/				# Presentation Layer (Controllers)
├── DemoApi.Application/	# Application Layer (Services, DTOs)
├── DemoApi.Domain/			# Domain Layer (Entities, Enums, Interfaces)
├── DemoApi.Infrastructure/	# Infrastructure Layer (EF, Repositories)
├── DemoApi.Tests/          # Unit and integration tests
```
## Architecture Layers

### 1. Controller Layer (`/controller`)

- **Purpose**: Handle HTTP requests and responses
- **Responsibilities**:
  - Define REST endpoints
  - Validate request data
  - Call service layer methods
  - Return appropriate HTTP responses

**Example Structure:**

```C#
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Swashbuckle.AspNetCore.Annotations;
using System.Collections.Generic;
using System.Threading.Tasks;

namespace DemoApi.Controllers
{
    [ApiController]
    [Route("api/users")]
    [Produces("application/json")]
    [SwaggerTag("User Management", Description = "APIs for managing users")]
    public class UserController : ControllerBase
    {
        private readonly IUserService _userService;
        private readonly ILogger<UserController> _logger;

        public UserController(IUserService userService, ILogger<UserController> logger)
        {
            _userService = userService;
            _logger = logger;
        }

        [HttpGet]
        [SwaggerOperation(Summary = "Get all users", Description = "Retrieve a paginated list of all users")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<ActionResult<PagedResult<UserResponse>>> GetAllUsers([FromQuery] PagingParameters paging)
        {
            var users = await _userService.GetAllUsersAsync(paging);
            return Ok(users);
        }

        [HttpGet("{id}")]
        [SwaggerOperation(Summary = "Get user by ID", Description = "Retrieve a user by their ID")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<ActionResult<UserResponse>> GetUserById(long id)
        {
            var user = await _userService.GetUserByIdAsync(id);
            if (user == null) return NotFound();
            return Ok(user);
        }

        [HttpPost]
        [SwaggerOperation(Summary = "Create a new user", Description = "Create a new user")]
        [ProducesResponseType(StatusCodes.Status201Created)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<ActionResult<UserResponse>> CreateUser([FromBody] CreateUserRequest request)
        {
            var createdUser = await _userService.CreateUserAsync(request);
            return CreatedAtAction(nameof(GetUserById), new { id = createdUser.Id }, createdUser);
        }

        [HttpPut("{id}")]
        [SwaggerOperation(Summary = "Update an existing user", Description = "Update user details by ID")]
        [ProducesResponseType(StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<ActionResult<UserResponse>> UpdateUser(long id, [FromBody] UpdateUserRequest request)
        {
            var updatedUser = await _userService.UpdateUserAsync(id, request);
            if (updatedUser == null) return NotFound();
            return Ok(updatedUser);
        }

        [HttpDelete("{id}")]
        [SwaggerOperation(Summary = "Delete a user", Description = "Delete a user by ID")]
        [ProducesResponseType(StatusCodes.Status204NoContent)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        [ProducesResponseType(StatusCodes.Status500InternalServerError)]
        public async Task<IActionResult> DeleteUser(long id)
        {
            var success = await _userService.DeleteUserAsync(id);
            if (!success) return NotFound();
            return NoContent();
        }
    }
}
```

### 2. DTO Layer (`/dto`)

- **Purpose**: Data Transfer Objects for API communication
- **Responsibilities**:
  - Request DTOs for incoming data
  - Response DTOs for outgoing data
  - Data validation annotations (via System.ComponentModel.DataAnnotations)
  - Swagger documentation (via Swashbuckle.AspNetCore.Annotations)

**Example Structure:**

```C#
// Request DTO
using System.ComponentModel.DataAnnotations;
using Swashbuckle.AspNetCore.Annotations;

namespace DemoApi.Dtos
{
    public class CreateUserRequest
    {
        [Required]
        [SwaggerSchema("First name of the user", Example = "John")]
        public string FirstName { get; set; } = string.Empty;

        [Required]
        [SwaggerSchema("Last name of the user", Example = "Doe")]
        public string LastName { get; set; } = string.Empty;

        [Required]
        [EmailAddress]
        [SwaggerSchema("Email address of the user", Example = "john.doe@example.com")]
        public string Email { get; set; } = string.Empty;

        [SwaggerSchema("Phone number of the user", Example = "+1234567890")]
        public string? PhoneNumber { get; set; }
    }
}

// Update DTO
using Swashbuckle.AspNetCore.Annotations;

namespace DemoApi.Dtos
{
    public class UpdateUserRequest
    {
        [SwaggerSchema("First name of the user", Example = "John")]
        public string? FirstName { get; set; }

        [SwaggerSchema("Last name of the user", Example = "Doe")]
        public string? LastName { get; set; }

        [EmailAddress]
        [SwaggerSchema("Email address of the user", Example = "john.doe@example.com")]
        public string? Email { get; set; }

        [SwaggerSchema("Phone number of the user", Example = "+1234567890")]
        public string? PhoneNumber { get; set; }

        [SwaggerSchema("Status of the user", Example = "ACTIVE")]
        public UserStatus? Status { get; set; }
    }
}

// Response DTO
using Swashbuckle.AspNetCore.Annotations;

namespace DemoApi.Dtos
{
    public class UserResponse
    {
        [SwaggerSchema("Unique identifier of the user", Example = "1")]
        public long Id { get; set; }

        [SwaggerSchema("First name of the user", Example = "John")]
        public string FirstName { get; set; } = string.Empty;

        [SwaggerSchema("Last name of the user", Example = "Doe")]
        public string LastName { get; set; } = string.Empty;

        [SwaggerSchema("Full name of the user", Example = "John Doe")]
        public string FullName => $"{FirstName} {LastName}";

        [SwaggerSchema("Email address of the user", Example = "john.doe@example.com")]
        public string Email { get; set; } = string.Empty;

        [SwaggerSchema("Phone number of the user", Example = "+1234567890")]
        public string? PhoneNumber { get; set; }

        [SwaggerSchema("Status of the user", Example = "ACTIVE")]
        public UserStatus Status { get; set; }

        [SwaggerSchema("Display name of the user status", Example = "Active")]
        public string StatusDisplayName => Status.ToString(); // You can customize this

        [SwaggerSchema("Timestamp when the user was created", Example = "2023-10-01T12:00:00")]
        public DateTime CreatedAt { get; set; }

        [SwaggerSchema("Timestamp when the user was last updated", Example = "2023-10-01T12:00:00")]
        public DateTime UpdatedAt { get; set; }
    }
}
```

### 3. Service Layer (`/service`)

- **Purpose**: Business logic and orchestration
- **Responsibilities**:
  - Implement business rules
  - Coordinate between repositories
  - Handle transactions
  - Data transformation

**Example Structure:**

```C#
using DemoApi.Dtos;
using DemoApi.Entities;
using DemoApi.Enums;
using DemoApi.Repositories;
using Microsoft.Extensions.Logging;


namespace DemoApi.Services
{
    public interface IUserService
    {
        Task<UserResponse> CreateUserAsync(CreateUserRequest request);
        Task<UserResponse?> GetUserByIdAsync(long id);
        Task<UserResponse> UpdateUserAsync(long id, UpdateUserRequest request);
        Task DeleteUserAsync(long id);
        Task<PagedResult<UserResponse>> GetAllUsersAsync(PagingParameters paging);
    }
}


namespace DemoApi.Services
{
    public class UserService : IUserService
    {
        private readonly IUserRepository _userRepository;
        private readonly ILogger<UserService> _logger;

        public UserService(IUserRepository userRepository, ILogger<UserService> logger)
        {
            _userRepository = userRepository;
            _logger = logger;
        }

        public async Task<UserResponse> CreateUserAsync(CreateUserRequest request)
        {
            _logger.LogInformation("Creating new user with email: {Email}", request.Email);

            if (await _userRepository.ExistsByEmailAsync(request.Email))
                throw new ArgumentException("User with email already exists");

            var user = new User
            {
                FirstName = request.FirstName,
                LastName = request.LastName,
                Email = request.Email,
                PhoneNumber = request.PhoneNumber,
                Status = UserStatus.ACTIVE,
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow
            };

            var savedUser = await _userRepository.SaveAsync(user);
            return ConvertToResponse(savedUser);
        }

        public async Task<UserResponse?> GetUserByIdAsync(long id)
        {
            var user = await _userRepository.FindByIdAsync(id);
            return user is null ? null : ConvertToResponse(user);
        }

        public async Task<UserResponse> UpdateUserAsync(long id, UpdateUserRequest request)
        {
            var user = await _userRepository.FindByIdAsync(id)
                ?? throw new ArgumentException("User not found");

            if (!string.IsNullOrEmpty(request.FirstName)) user.FirstName = request.FirstName;
            if (!string.IsNullOrEmpty(request.LastName)) user.LastName = request.LastName;
            if (!string.IsNullOrEmpty(request.PhoneNumber)) user.PhoneNumber = request.PhoneNumber;
            if (request.Status.HasValue) user.Status = request.Status.Value;

            user.UpdatedAt = DateTime.UtcNow;

            var updatedUser = await _userRepository.SaveAsync(user);
            return ConvertToResponse(updatedUser);
        }

        public async Task DeleteUserAsync(long id)
        {
            if (!await _userRepository.ExistsByIdAsync(id))
                throw new ArgumentException("User not found");

            await _userRepository.DeleteByIdAsync(id);
        }

        public async Task<PagedResult<UserResponse>> GetAllUsersAsync(PagingParameters paging)
        {
            var pagedUsers = await _userRepository.FindAllAsync(paging);
            var responseItems = pagedUsers.Items.Select(ConvertToResponse).ToList();

            return new PagedResult<UserResponse>
            {
                Items = responseItems,
                TotalCount = pagedUsers.TotalCount,
                PageNumber = paging.PageNumber,
                PageSize = paging.PageSize
            };
        }

        private UserResponse ConvertToResponse(User user)
        {
            return new UserResponse
            {
                Id = user.Id,
                FirstName = user.FirstName,
                LastName = user.LastName,
                Email = user.Email,
                PhoneNumber = user.PhoneNumber,
                Status = user.Status,
                CreatedAt = user.CreatedAt,
                UpdatedAt = user.UpdatedAt
            };
        }
    }
}
```
### 4. Entity Layer (`/entity`)

- **Purpose**: Database model representation
- **Responsibilities**:
  - EF Core Entity Definitions
  - Table and Column Mapping
  - Entity Relationships
  - Annotations for Persistence and Validation Control

**Example Structure:**

```C#
using DemoApi.Enums;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace DemoApi.Entities
{
    [Table("users")]
    public class User
    {
        [Key]
        [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
        [Column("id")]
        public long Id { get; set; }

        [Required]
        [MaxLength(100)]
        [Column("first_name")]
        public string FirstName { get; set; } = string.Empty;

        [Required]
        [MaxLength(100)]
        [Column("last_name")]
        public string LastName { get; set; } = string.Empty;

        [Required]
        [MaxLength(255)]
        [EmailAddress]
        [Column("email")]
        public string Email { get; set; } = string.Empty;

        [MaxLength(20)]
        [Column("phone_number")]
        public string? PhoneNumber { get; set; }

        [Required]
        [Column("status")]
        [EnumDataType(typeof(UserStatus))]
        public UserStatus Status { get; set; } = UserStatus.ACTIVE;

        [Required]
        [Column("created_at")]
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

        [Required]
        [Column("updated_at")]
        public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;

        [NotMapped]
        public string FullName => $"{FirstName} {LastName}";
    }
}
```

### 5. Enums (`/enums`)

- **Purpose**: Define constant values and types
- **Responsibilities**:
  - Status enums
  - Category types
  - Configuration constants

**Example Structure:**

```C#
namespace DemoApi.Enums
{
    public enum UserStatus
    {
        ACTIVE,
        SUSPENDED,
        INACTIVE,
        PENDING
    }

    public static class UserStatusExtensions
    {
        public static string GetDisplayName(this UserStatus status)
        {
            return status switch
            {
                UserStatus.ACTIVE => "Active",
                UserStatus.SUSPENDED => "Suspended",
                UserStatus.INACTIVE => "Inactive",
                UserStatus.PENDING => "Pending",
                _ => "Unknown"
            };
        }

        public static bool CanLogin(this UserStatus status)
        {
            return status == UserStatus.ACTIVE;
        }

        public static bool CanModify(this UserStatus status)
        {
            return status == UserStatus.ACTIVE || status == UserStatus.SUSPENDED;
        }
    }
}
```

### 6. Repository Layer (`/repository`)

- **Purpose**: Data access abstraction
- **Responsibilities**:
  - Database operations
  - Custom queries
  - Repository interfaces

**Example Structure:**

```C#
using DemoApi.Data;
using DemoApi.Dtos;
using DemoApi.Entities;
using DemoApi.Enums;
using Microsoft.EntityFrameworkCore;


namespace DemoApi.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options) { }

        public DbSet<User> Users { get; set; }
    }
}


namespace DemoApi.Repositories
{
    public interface IUserRepository
    {
        Task<User?> FindByIdAsync(long id);
        Task<User?> FindByEmailAsync(string email);
        Task<bool> ExistsByEmailAsync(string email);
        Task<bool> ExistsByIdAsync(long id);
        Task<User> SaveAsync(User user);
        Task DeleteByIdAsync(long id);
        Task<PagedResult<User>> FindAllAsync(PagingParameters paging);
        Task<PagedResult<User>> FindByStatusAsync(UserStatus status, PagingParameters paging);
        Task<PagedResult<User>> FindByNameContainingAsync(string searchTerm, PagingParameters paging);
        Task<List<User>> FindRecentUsersAsync(DateTime since);
        Task<long> CountByStatusAsync(UserStatus status);
    }
}


namespace DemoApi.Repositories
{
    public class UserRepository : IUserRepository
    {
        private readonly AppDbContext _context;

        public UserRepository(AppDbContext context)
        {
            _context = context;
        }

        public async Task<User?> FindByIdAsync(long id)
        {
            return await _context.Users.FindAsync(id);
        }

        public async Task<User?> FindByEmailAsync(string email)
        {
            return await _context.Users.FirstOrDefaultAsync(u => u.Email == email);
        }

        public async Task<bool> ExistsByEmailAsync(string email)
        {
            return await _context.Users.AnyAsync(u => u.Email == email);
        }

        public async Task<bool> ExistsByIdAsync(long id)
        {
            return await _context.Users.AnyAsync(u => u.Id == id);
        }

        public async Task<User> SaveAsync(User user)
        {
            _context.Users.Update(user);
            await _context.SaveChangesAsync();
            return user;
        }

        public async Task DeleteByIdAsync(long id)
        {
            var user = await FindByIdAsync(id);
            if (user != null)
            {
                _context.Users.Remove(user);
                await _context.SaveChangesAsync();
            }
        }

        public async Task<PagedResult<User>> FindAllAsync(PagingParameters paging)
        {
            var query = _context.Users.AsQueryable();
            var totalCount = await query.CountAsync();
            var items = await query
                .Skip((paging.PageNumber - 1) * paging.PageSize)
                .Take(paging.PageSize)
                .ToListAsync();

            return new PagedResult<User>
            {
                Items = items,
                TotalCount = totalCount,
                PageNumber = paging.PageNumber,
                PageSize = paging.PageSize
            };
        }

        public async Task<PagedResult<User>> FindByStatusAsync(UserStatus status, PagingParameters paging)
        {
            var query = _context.Users.Where(u => u.Status == status);
            var totalCount = await query.CountAsync();
            var items = await query
                .Skip((paging.PageNumber - 1) * paging.PageSize)
                .Take(paging.PageSize)
                .ToListAsync();

            return new PagedResult<User>
            {
                Items = items,
                TotalCount = totalCount,
                PageNumber = paging.PageNumber,
                PageSize = paging.PageSize
            };
        }

        public async Task<PagedResult<User>> FindByNameContainingAsync(string searchTerm, PagingParameters paging)
        {
            var query = _context.Users.Where(u =>
                EF.Functions.Like(u.FirstName.ToLower(), $"%{searchTerm.ToLower()}%") ||
                EF.Functions.Like(u.LastName.ToLower(), $"%{searchTerm.ToLower()}%"));

            var totalCount = await query.CountAsync();
            var items = await query
                .Skip((paging.PageNumber - 1) * paging.PageSize)
                .Take(paging.PageSize)
                .ToListAsync();

            return new PagedResult<User>
            {
                Items = items,
                TotalCount = totalCount,
                PageNumber = paging.PageNumber,
                PageSize = paging.PageSize
            };
        }

        public async Task<List<User>> FindRecentUsersAsync(DateTime since)
        {
            return await _context.Users
                .Where(u => u.CreatedAt >= since)
                .ToListAsync();
        }

        public async Task<long> CountByStatusAsync(UserStatus status)
        {
            return await _context.Users.LongCountAsync(u => u.Status == status);
        }
    }
}
```

#### 7. Security Configuration (`/config`)

- **Purpose**: Application security settings
- **Responsibilities**:
  - Define security policies
  - Configure authentication and authorization
**Example Structure:**

```C#
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace DemoApi.Config
{
    public static class SecurityConfig
    {
        public static void ConfigureSecurity(this IServiceCollection services)
        {
            // Adiciona autenticação e autorização se necessário
            services.AddAuthorization();
        }

        public static void UseSecurity(this IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseHttpsRedirection();

            app.UseRouting();

            // Desabilita CSRF (não necessário para APIs REST)
            // Se usar autenticação baseada em cookies, reconsidere isso
            app.Use((context, next) =>
            {
                context.Request.Headers.Remove("X-CSRF-TOKEN");
                return next();
            });

            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers(); // Permite todas as rotas
            });
        }
    }
}
```

#### 8. Program (`/program`)

- **Purpose**: Application authentication
- **Responsibilities**:
  - Configure authentication and authorization
**Example Structure:**

```C#
var builder = WebApplication.CreateBuilder(args);

// Configura serviços
builder.Services.ConfigureSecurity();
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configura pipeline
app.UseSecurity(app.Environment);

app.Run();
```


## Getting Started

### Stack Requirements

- .Net 8

## Development Guidelines

### Creating a New Feature

#### 1. Entity First (`/entity`)

Create your database model:

```C#
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace DemoApi.Entities
{
    [Table("users")]
    public class User
    {
        [Key]
        [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
        [Column("id")]
        public long Id { get; set; }

        [Required]
        [Column("name")]
        public string Name { get; set; } = string.Empty;

        [Required]
        [EmailAddress]
        [Column("email")]
        public string Email { get; set; } = string.Empty;
    }
}
```

#### 2. Repository (`/repository`)

Define data access interface:

```C#
using DemoApi.Entities;

namespace DemoApi.Repositories
{
    public interface IUserRepository
    {
        Task<User?> FindByEmailAsync(string email);
        Task<User?> FindByIdAsync(long id);
        Task<User> SaveAsync(User user);
        Task<bool> ExistsByEmailAsync(string email);
        Task<bool> ExistsByIdAsync(long id);
        Task DeleteByIdAsync(long id);
    }
}

using DemoApi.Data;
using DemoApi.Entities;
using Microsoft.EntityFrameworkCore;

namespace DemoApi.Repositories
{
    public class UserRepository : IUserRepository
    {
        private readonly AppDbContext _context;

        public UserRepository(AppDbContext context)
        {
            _context = context;
        }

        public async Task<User?> FindByEmailAsync(string email)
        {
            return await _context.Users.FirstOrDefaultAsync(u => u.Email == email);
        }

        public async Task<User?> FindByIdAsync(long id)
        {
            return await _context.Users.FindAsync(id);
        }
    }
}

using DemoApi.Entities;
using Microsoft.EntityFrameworkCore;

namespace DemoApi.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options)
            : base(options) { }

        public DbSet<User> Users { get; set; }
    }
}
```

#### 3. DTOs (`/dto`)

Create request and response objects:

```C#
// Request DTO
using System.ComponentModel.DataAnnotations;
using Swashbuckle.AspNetCore.Annotations;

namespace DemoApi.Dtos
{
    public class CreateUserRequest
    {
        [Required]
        [SwaggerSchema(Description = "Name of the user", Example = "John Doe")]
        public string Name { get; set; } = string.Empty;

        [Required]
        [EmailAddress]
        [SwaggerSchema(Description = "Email of the user", Example = "john.doe@example.com")]
        public string Email { get; set; } = string.Empty;
    }
}


// Response DTO
using Swashbuckle.AspNetCore.Annotations;

namespace DemoApi.Dtos
{
    public class UserResponse
    {
        [SwaggerSchema(Description = "ID of the user", Example = "1")]
        public long Id { get; set; }

        [SwaggerSchema(Description = "Name of the user", Example = "John Doe")]
        public string Name { get; set; } = string.Empty;

        [SwaggerSchema(Description = "Email of the user", Example = "john.doe@example.com")]
        public string Email { get; set; } = string.Empty;
    }
}
```

#### 5. Service (`/service`)

Implement business logic following your business rules:

```C#
using DemoApi.Dtos;

namespace DemoApi.Services
{
    public interface IUserService
    {
        Task<UserResponse> CreateUserAsync(CreateUserRequest request);
    }
}

using DemoApi.Dtos;
using DemoApi.Entities;
using DemoApi.Repositories;

namespace DemoApi.Services
{
    public class UserService : IUserService
    {
        private readonly IUserRepository _userRepository;

        public UserService(IUserRepository userRepository)
        {
            _userRepository = userRepository;
        }

        public async Task<UserResponse> CreateUserAsync(CreateUserRequest request)
        {
            var user = new User
            {
                Name = request.Name,
                Email = request.Email
            };

            var savedUser = await _userRepository.SaveAsync(user);

            return new UserResponse
            {
                Id = savedUser.Id,
                Name = savedUser.Name,
                Email = savedUser.Email
            };
        }
    }
}
```

#### 6. Controller (`/controller`)

Create REST endpoints:

```C#
using DemoApi.Dtos;
using DemoApi.Services;
using Microsoft.AspNetCore.Mvc;
using Swashbuckle.AspNetCore.Annotations;

namespace DemoApi.Controllers
{
    [ApiController]
    [Route("api/users")]
    [Produces("application/json")]
    [SwaggerTag("User Management", Description = "APIs for managing users")]
    public class UserController : ControllerBase
    {
        private readonly IUserService _userService;

        public UserController(IUserService userService)
        {
            _userService = userService;
        }

        [HttpPost]
        [SwaggerOperation(Summary = "Create a new user", Description = "Create a new user")]
        [SwaggerResponse(201, "User created successfully", typeof(UserResponse))]
        [SwaggerResponse(400, "Invalid request data")]
        [SwaggerResponse(500, "Internal server error")]
        public async Task<ActionResult<UserResponse>> CreateUser([FromBody] CreateUserRequest request)
        {
            if (!ModelState.IsValid)
                return BadRequest(ModelState);

            var response = await _userService.CreateUserAsync(request);
            return CreatedAtAction(nameof(CreateUser), new { id = response.Id }, response);
        }
    }
}
```

### Best Practices

1. **Separation of Concerns**: Each layer should have a single responsibility
2. **DTO Usage**: Never expose entities directly in controllers
3. **Validation**: Use Bean Validation annotations in DTOs
4. **Error Handling**: Implement proper exception handling if needed

## Code Generation Patterns for AI Agents

### Required Imports by Layer

**Entity Layer:**

```C#
using System;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
```

**Repository Layer:**

```C#
using System.Collections.Generic;
using System.Threading.Tasks;
using DemoApi.Entities;
using DemoApi.Enums;
using DemoApi.Dtos;
using Microsoft.EntityFrameworkCore;
using System.Linq;
```

**Service Layer:**

```C#
using System.Threading.Tasks;
using DemoApi.Dtos;
using DemoApi.Entities;
using DemoApi.Repositories;
using Microsoft.Extensions.Logging;
```

**Controller Layer:**

```C#
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;
using DemoApi.Dtos;
using DemoApi.Services;
using Swashbuckle.AspNetCore.Annotations;
```

**DTO Layer:**

```C#
using System.ComponentModel.DataAnnotations;
using Swashbuckle.AspNetCore.Annotations;
```

### Naming Conventions

- **Entities**: PascalCase (e.g., `User`, `Order`, `Product`)
- **Tables**: snake_case (e.g., `users`, `orders`, `products`)
- **Columns**: snake_case (e.g., `first_name`, `created_at`)
- **Request DTOs**: `Create{Entity}Request`, `Update{Entity}Request`
- **Response DTOs**: `{Entity}Response`
- **Controllers**: `{Entity}Controller`
- **Services**: `{Entity}Service`
- **Repositories**: `{Entity}Repository`
- **Enums**: PascalCase (e.g., `UserStatus`, `OrderStatus`)


### Standard Annotations

- **Entity** + **@Table(name = "table_name")**: For JPA entities
- **Service**: For service classes
- **Repository**: For repository interfaces
- **[ApiController] + [Route("api/entity")]**: For controllers
- **Data** + **NoArgsConstructor** + **AllArgsConstructor**: For DTOs and entities
- **RequiredArgsConstructor**: For dependency injection
- **ILogger**: For logging

## Available Dependencies

- **Microsoft.AspNetCore.App (ASP.NET Core Web API)**: REST API development

