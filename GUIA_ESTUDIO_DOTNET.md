# Mini guía de estudio .NET aplicada a este proyecto

Esta API es un proyecto monolítico de ASP.NET Core para gestionar categorías, productos, inventario, usuarios, autenticación y carga de imágenes. No incluye carrito, pedidos, pagos ni envíos.

## 1. .NET y ASP.NET Core

- **.NET**: plataforma para crear aplicaciones con C#.
- **ASP.NET Core**: framework de .NET para crear aplicaciones web y APIs.
- Este proyecto es una API: recibe peticiones HTTP y devuelve normalmente JSON.
- El punto de inicio es `Program.cs`.
- El framework objetivo es `net8.0`.

## 2. HTTP y API REST

**HTTP** es el protocolo que usa un cliente para comunicarse con la API.

| Método | Uso | Ejemplo |
| --- | --- | --- |
| `GET` | Consultar datos | Obtener productos |
| `POST` | Crear datos | Crear producto o usuario |
| `PUT` | Actualizar datos | Actualizar categoría |
| `PATCH` | Actualizar parcialmente o ejecutar una acción | Descontar stock |
| `DELETE` | Eliminar datos | Eliminar producto |

Ejemplo:

```http
GET /api/v1/Products/1
```

Significa: "dame el producto cuyo ID es 1".

| Código | Significado |
| --- | --- |
| `200 OK` | Operación correcta |
| `201 Created` | Recurso creado |
| `204 No Content` | Operación correcta sin contenido de respuesta |
| `400 Bad Request` | Datos inválidos |
| `401 Unauthorized` | Falta autenticación |
| `403 Forbidden` | No tienes permisos |
| `404 Not Found` | No existe |
| `500 Internal Server Error` | Error interno |

Puedes probar las rutas desde Swagger o `ApiEcommerce.http`.

## 3. Controllers

Los **controllers** reciben las peticiones HTTP, coordinan la operación y devuelven respuestas.

Ubicación: `Controllers/`

- `ProductsController`: CRUD, búsqueda, paginación, imágenes y descuento de stock.
- `UsersController`: registro, login y consulta de usuarios.
- `V1/CategoriesController`: categorías versión 1.
- `V2/CategoriesController`: categorías versión 2.

Un controller suele tener atributos como:

```csharp
[ApiController]
[Route("api/v{version:apiVersion}/[controller]")]
```

- `[ApiController]`: activa comportamientos útiles para APIs, como validación automática.
- `[Route(...)]`: define la URL base del controller.
- `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpPatch]` y `[HttpDelete]`: indican el método HTTP aceptado.

Ejemplo conceptual:

```csharp
[HttpGet("{productId:int}")]
public IActionResult GetProduct(int productId)
```

Lee un parámetro de la URL y devuelve un producto.

## 4. IActionResult y respuestas

`IActionResult` permite que un endpoint devuelva distintos resultados HTTP:

```csharp
return Ok(product);              // 200
return CreatedAtRoute(...);      // 201
return NoContent();              // 204
return BadRequest("Error");      // 400
return NotFound();               // 404
return Unauthorized();           // 401
```

Una API no devuelve solo datos: también comunica el resultado mediante códigos HTTP.

## 5. Models o entidades

Los **models** representan datos del dominio y, en muchos casos, tablas de la base de datos.

Ubicación: `Models/`

- `Category`
- `Product`
- `ApplicationUser`

`Product` representa un producto con propiedades como:

```text
ProductId, Name, Description, Price, SKU,
Stock, CategoryId, ImgUrl, CreationDate
```

Relación principal:

```text
Una Category tiene muchos Product.
Un Product pertenece a una Category.
```

## 6. DTO

**DTO** significa *Data Transfer Object*. Es una clase que define exactamente qué recibe o devuelve la API.

Ubicación: `Models/Dtos/`

Ejemplos:

- `CreateProductDto`: datos para crear un producto.
- `UpdateProductDto`: datos para actualizarlo.
- `ProductDto`: datos que devuelve la API.
- `CreateUserDto`: datos de registro.
- `UserLoginDto`: usuario y contraseña para iniciar sesión.

No se devuelve directamente una entidad como `Product` porque los DTOs permiten:

- Evitar exponer columnas internas.
- Adaptar nombres y formatos de los datos.
- Separar el modelo de base de datos del contrato público.
- Usar validaciones distintas para crear y editar.

Flujo:

```text
JSON recibido
-> CreateProductDto
-> Product
-> Base de datos

Base de datos
-> Product
-> ProductDto
-> JSON de respuesta
```

## 7. Mapster

**Mapster** es la librería que convierte objetos entre entidades y DTOs.

Configuración: `Mapping/MapsterConfig.cs`.

Ejemplo:

```csharp
var productDto = product.Adapt<ProductDto>();
```

Convierte un `Product` en un `ProductDto`, evitando asignar manualmente cada propiedad. También permite mapear propiedades con nombres distintos, por ejemplo el nombre de la categoría dentro de un DTO de producto.

## 8. Repository Pattern

Un **repository** centraliza el acceso a datos.

Ubicación:

```text
Repository/
Repository/IRepository/
```

Flujo:

```text
ProductsController
-> IProductRepository
-> ProductRepository
-> ApplicationDbContext
-> SQL Server
```

Las interfaces definen el contrato, por ejemplo `IProductRepository`. Las implementaciones contienen las consultas reales, por ejemplo `ProductRepository`.

Ventajas:

- El controller conoce menos detalles de la base de datos.
- Las consultas se concentran en una capa.
- Es más fácil sustituir el acceso a datos en pruebas.

En este proyecto, los repositories son la capa entre los controllers y Entity Framework Core.

## 9. Dependency Injection

La **inyección de dependencias** permite que ASP.NET cree y entregue los objetos que una clase necesita.

En `Program.cs` se registran los repositories:

```csharp
builder.Services.AddScoped<IProductRepository, ProductRepository>();
```

Después el controller lo recibe por constructor:

```csharp
public ProductsController(IProductRepository productRepository)
```

| Registro | Vida |
| --- | --- |
| `AddSingleton` | Una instancia para toda la aplicación |
| `AddScoped` | Una instancia por petición HTTP |
| `AddTransient` | Una instancia nueva cada vez que se solicita |

El proyecto usa `AddScoped` para los repositories.

## 10. Entity Framework Core

**Entity Framework Core** es un ORM: permite trabajar con una base de datos mediante objetos C# y LINQ.

Configuración: `Data/ApplicationDbContext.cs`.

```text
Objeto C# Product
<-> Entity Framework Core
<-> Tabla Products
```

Ejemplos de operaciones:

```csharp
_db.Products.ToList();
_db.Products.Find(id);
_db.Products.Add(product);
_db.Products.Update(product);
_db.Products.Remove(product);
_db.SaveChanges();
```

En la mayoría de los casos no se escribe SQL directamente: EF Core lo genera.

## 11. DbContext y DbSet

El **DbContext** representa una sesión de trabajo con la base de datos.

En este proyecto es `ApplicationDbContext`, que hereda de `IdentityDbContext<ApplicationUser>`.

Contiene `DbSet`, que representan tablas:

```csharp
DbSet<Product> Products
DbSet<Category> Categories
```

Puedes verlo como la puerta de acceso a la base de datos desde C#.

## 12. LINQ

**LINQ** permite consultar colecciones y bases de datos con sintaxis C#.

Ejemplos:

```csharp
_db.Products.Where(p => p.Stock > 0)
_db.Products.FirstOrDefault(p => p.ProductId == id)
_db.Products.Include(p => p.Category)
_db.Products.OrderBy(p => p.Name)
```

- `Where`: filtra.
- `FirstOrDefault`: devuelve el primero o `null`.
- `Include`: carga una relación, por ejemplo la categoría del producto.
- `OrderBy`: ordena.
- `Skip` y `Take`: permiten paginación.

## 13. Migraciones

Las **migraciones** guardan cambios de estructura de la base de datos como código.

Ubicación: `Migrations/`.

Ejemplos de cambios en este proyecto:

- Crear tabla `Categories`.
- Crear tabla `Products`.
- Añadir las tablas de Identity.
- Añadir la URL local de una imagen al producto.

Comandos clave:

```bash
dotnet ef migrations add NombreMigracion
dotnet ef database update
```

- `migrations add`: genera una migración desde cambios en entidades o `DbContext`.
- `database update`: aplica las migraciones a SQL Server.

## 14. SQL Server y Docker

La base de datos es SQL Server 2022 y se levanta con Docker.

Archivo: `docker-compose.yaml`.

```text
API .NET
-> EF Core
-> SQL Server en Docker
-> Volumen Docker para persistir datos
```

Docker permite ejecutar SQL Server en un contenedor sin instalarlo directamente en Windows.

## 15. Seeding

El **seeding** inserta datos iniciales en la base de datos.

Archivo: `Data/DataSeeder.cs`.

El proyecto crea, si no existen:

- Roles `Admin` y `User`.
- Usuarios iniciales.
- Categorías.
- Productos de ejemplo.

Es útil para probar la API nada más crear la base de datos.

## 16. ASP.NET Identity

**Identity** es el sistema de ASP.NET Core para gestionar usuarios, contraseñas hasheadas y roles.

Modelo principal:

```text
ApplicationUser : IdentityUser
```

`IdentityUser` ya incluye campos como usuario, correo, contraseña hasheada y otros datos de seguridad. `ApplicationUser` añade el campo `Name`.

Servicios importantes:

```csharp
UserManager<ApplicationUser>
RoleManager<IdentityRole>
```

- `UserManager`: crea usuarios, busca usuarios y valida contraseñas.
- `RoleManager`: crea y consulta roles.

## 17. JWT

**JWT** significa *JSON Web Token*. Es un token firmado que identifica al usuario después del login.

Flujo:

```text
Usuario envía username + password
-> API valida con Identity
-> API crea un JWT
-> Cliente guarda el token
-> Cliente envía Authorization: Bearer <token>
-> API valida token y permisos
```

Ejemplo de cabecera:

```http
Authorization: Bearer eyJhbGciOi...
```

La configuración está en `Program.cs` y `appsettings.json`. El token contiene, entre otros datos, el identificador, nombre de usuario, rol y fecha de expiración.

## 18. Authentication y Authorization

| Concepto | Pregunta |
| --- | --- |
| Authentication | ¿Quién eres? |
| Authorization | ¿Qué puedes hacer? |

En este proyecto:

```csharp
[Authorize]
```

Exige un usuario autenticado.

```csharp
[Authorize(Roles = "Admin")]
```

Exige además que tenga el rol `Admin`.

```csharp
[AllowAnonymous]
```

Permite llamar al endpoint sin token, por ejemplo en registro, login o consultas públicas.

## 19. API Versioning

El proyecto usa versionado de API:

```text
/api/v1/Categories
/api/v2/Categories
```

Sirve para cambiar una API sin romper a los clientes que todavía dependen de una versión anterior.

```text
v1 sigue funcionando
v2 introduce cambios
```

Aquí se versionan principalmente las categorías. Productos y usuarios son neutrales a versión.

## 20. Swagger y OpenAPI

Swagger documenta y permite probar la API desde el navegador.

Ruta local:

```text
http://localhost:5176/swagger
```

Swagger muestra:

- Endpoints.
- Parámetros.
- DTOs.
- Códigos de respuesta.
- Esquema de autenticación Bearer.

**OpenAPI** es el estándar que describe la API; **Swagger UI** es la interfaz visual para usar esa descripción.

## 21. Middleware

Un **middleware** es una pieza del pipeline que procesa cada petición antes de llegar al controller.

En `Program.cs` hay componentes como:

```csharp
app.UseStaticFiles();
app.UseHttpsRedirection();
app.UseCors(...);
app.UseResponseCaching();
app.UseAuthentication();
app.UseAuthorization();
```

Orden simplificado:

```text
Petición HTTP
-> archivos estáticos
-> HTTPS
-> CORS
-> caché
-> autenticación
-> autorización
-> controller
```

El orden importa: primero se autentica al usuario y después se comprueba si tiene permisos.

## 22. CORS

**CORS** controla qué aplicaciones web pueden llamar a la API desde otro origen.

Ejemplo:

```text
Frontend en http://localhost:3000
-> quiere llamar
-> API en http://localhost:5176
```

El navegador bloquea esa comunicación si la API no permite ese origen mediante CORS.

La configuración está centralizada en `Program.cs` y `Constants/PolicyNames.cs`.

## 23. Response Caching

La caché guarda temporalmente respuestas para evitar repetir consultas iguales.

Ventaja:

```text
Menos consultas a SQL Server
-> respuesta potencialmente más rápida
```

En este proyecto hay perfiles de caché en `Constants/CacheProfiles.cs`. La caché no reemplaza la base de datos: reutiliza respuestas durante un período corto.

## 24. Archivos estáticos e imágenes

Las imágenes se guardan en:

```text
wwwroot/ProductsImages/
```

`wwwroot` es la carpeta pública de archivos estáticos de ASP.NET Core.

Flujo:

```text
Cliente envía imagen multipart/form-data
-> ProductsController recibe un IFormFile
-> se guarda el archivo en wwwroot/ProductsImages
-> UseStaticFiles permite acceder mediante una URL
```

`IFormFile` representa un archivo recibido por HTTP.

## 25. Validación

La validación comprueba que los datos recibidos tengan el formato correcto.

Ejemplos:

```csharp
[Required]
[StringLength(50)]
[Range(1, 100)]
```

Si un DTO no cumple estos atributos, `[ApiController]` normalmente devuelve `400 Bad Request` automáticamente.

Ejemplo:

```csharp
public class CreateCategoryDto
{
  [Required]
  [StringLength(50, MinimumLength = 3)]
  public string Name { get; set; }
}
```

## 26. Configuración

Archivos principales:

```text
appsettings.json
appsettings.Development.json
Properties/launchSettings.json
```

- `appsettings.json`: configuración general.
- `appsettings.Development.json`: configuración exclusiva de desarrollo.
- `launchSettings.json`: perfiles, puertos y URLs locales.

Aquí se guardan valores como la cadena de conexión, configuración JWT, logs y puertos de desarrollo. En producción, secretos como claves JWT y contraseñas no deberían quedar en Git.

## 27. Nullable Reference Types

El proyecto tiene nullable activado:

```xml
<Nullable>enable</Nullable>
```

Esto ayuda a evitar errores por valores `null`.

```csharp
string? imageUrl // Puede ser null.
string name      // Idealmente no debe ser null.
```

## 28. Dependencias NuGet principales

| Paquete | Uso |
| --- | --- |
| `Asp.Versioning.Mvc` | Versionado de endpoints |
| `Asp.Versioning.Mvc.ApiExplorer` | Swagger por versión |
| `Mapster` | Conversión entre entidades y DTOs |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | Autenticación JWT |
| `Microsoft.AspNetCore.Identity.EntityFrameworkCore` | Usuarios, contraseñas y roles |
| `Microsoft.EntityFrameworkCore.SqlServer` | Acceso a SQL Server mediante EF Core |
| `Microsoft.EntityFrameworkCore.Design` | Soporte de diseño y migraciones |
| `Microsoft.EntityFrameworkCore.Tools` | Herramientas de EF Core |
| `Swashbuckle.AspNetCore` | Swagger/OpenAPI |
| `BCrypt.Net-Next` | Está referenciado, pero no se observa uso activo; Identity gestiona las contraseñas |

## 29. Flujo completo de un endpoint

Ejemplo: consultar un producto.

```text
GET /api/v1/Products/1
-> Middleware de ASP.NET Core
-> Enrutamiento y versionado
-> ProductsController.GetProduct(1)
-> IProductRepository.GetProduct(1)
-> ProductRepository
-> ApplicationDbContext.Products
-> SQL Server
-> Product
-> Mapster convierte Product a ProductDto
-> Respuesta JSON con HTTP 200
```

## 30. Ruta recomendada de estudio

1. HTTP, JSON y REST.
2. Controllers, rutas y respuestas HTTP.
3. DTOs y validación.
4. Inyección de dependencias.
5. Entity Framework Core, `DbContext` y LINQ.
6. Repository Pattern.
7. Migraciones y SQL Server.
8. Identity, JWT, autenticación y roles.
9. Middleware, CORS, caché y archivos estáticos.
10. Swagger, versionado y configuración.

La práctica más útil es seguir un endpoint completo, por ejemplo `POST /api/v1/Products`, desde `ProductsController`, pasando por `ProductRepository`, hasta `ApplicationDbContext` y SQL Server.
