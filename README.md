# Práctica Final — Alta de dispositivos (Devices)

Práctica final del curso **[Claude Code .NET](https://www.trainingit.es/producto/claude-code-net/)** de TrainingIT.

📄 Tienes este mismo enunciado en Word para descargar: [`enunciado-practica-final.docx`](enunciado-practica-final.docx).

## Objetivo

Implementar el caso de uso completo de **creación de dispositivos** en MasterNet: el slice CQRS en `Application`, el endpoint `POST /api/devices` en el controller existente, se deben hacer las pruebas en los dos niveles del proyecto y la documentación del cambio.

Esta práctica no evalúa si sabes escribir un `INSERT`. Evalúa si eres capaz de **incorporarte a un código ajeno, descubrir sus convenciones y respetarlas**, apoyándote en Claude Code para explorar el repositorio. El proyecto ya tiene una forma muy concreta de hacer las cosas: tu trabajo debe ser indistinguible del código que ya está escrito.

Puedes usar Claude Code como quieras y con las herramientas que prefieras. Se evalúa el resultado.

## Antes de empezar

1. Pulsa **Use this template → Create a new repository** en esta página y crea el repositorio en tu cuenta, como **público**.
2. Clona ese repositorio en tu equipo y copia dentro tu proyecto MasterNet tal como lo tienes al terminar el curso.
3. Haz un primer commit con el proyecto, antes de empezar la práctica, para que tu trabajo quede separado del punto de partida.

## Punto de partida

`Device` es la entidad más atípica del dominio, y por eso es la elegida. Antes de escribir nada, léela.

**Lo que ya existe:**

| Elemento | Ruta |
|---|---|
| Entidad `Device` (sealed, constructor privado para EF) | `src/MasterNet.Domain/Devices/Device.cs` |
| Value Object `DeviceName` | `src/MasterNet.Domain/Devices/DeviceName.cs` |
| Mapeo del Value Object con `OwnsOne` → columna `name`, `nvarchar(200)`, `NOT NULL` | `src/MasterNet.Persistence/MasterNetDbContext.cs` |
| Tabla `devices` ya creada | `src/MasterNet.Persistence/Migrations/20251002092243_InitialMigration.cs` |
| Seed con nombres fijos | `src/MasterNet.Persistence/SeedData/device.json` |
| Specification de búsqueda por nombre | `src/MasterNet.Persistence/Specifications/DeviceByNameContainsSpec.cs` |
| Endpoint `GET /api/devices` | `src/MasterNet.WebApi/Controllers/DeviceController.cs` |

**Lo que no existe y tendrás que crear:**

- La carpeta `src/MasterNet.Application/Devices/` — no hay ningún caso de uso de dispositivos.
- El DTO de entrada en `src/MasterNet.WebApi/Models/`.
- Cualquier prueba sobre `Device`: no hay ni una.

> **Importante:** la tabla `devices` ya está creada desde la migración inicial. **No necesitas generar una migración nueva.**

Fíjate también en que el `DeviceController` actual **no sigue las convenciones del proyecto**: inyecta el `MasterNetDbContext` directamente, no usa el mediator ni `this.FromResult`, y devuelve la entidad de dominio serializada en crudo. Tu endpoint nuevo sí debe seguirlas. Arreglar el `GET` no es parte del trabajo obligatorio.

## Entregables

| # | Entregable | Dónde |
|---|---|---|
| 1 | Slice CQRS de creación (request + handler + validador) | `src/MasterNet.Application/Devices/DeviceCreate/` |
| 2 | Endpoint `POST /api/devices` | `src/MasterNet.WebApi/Controllers/DeviceController.cs` |
| 3 | Pruebas unitarias y de integración | `tests/MasterNet.Application.UnitTests/` y `tests/MasterNet.WebApi.IntegrationTests/` |
| 4 | Documentación actualizada y commit | `docs/`, historial de git |

## Contrato del endpoint

`POST /api/devices`, cuerpo JSON:

```json
{ "name": "Router TP-Link AX55" }
```

Respuestas esperadas:

| Caso | Código | Cuerpo |
|---|---|---|
| Nombre válido | `201 Created` | el `Guid` del dispositivo creado |
| Nombre ausente, vacío o solo espacios | `400 Bad Request` | `ValidationProblemDetails` con el campo `Name` |
| Nombre de más de 200 caracteres | `400 Bad Request` | `ValidationProblemDetails` con el campo `Name` |
| Nombre ya existente (ignorando mayúsculas) | `409 Conflict` | `ProblemDetails` con un mensaje explicativo |

No construyas ninguna de estas respuestas a mano en el controller. Los cuatro formatos salen ya resueltos de combinar `Result<T>` (`src/MasterNet.Application/Core/Result.cs`) con el traductor `this.FromResult(...)` (`src/MasterNet.WebApi/Extensions/ResultExtensions.cs`). Si estás escribiendo un `BadRequest(new { ... })`, vas por el camino equivocado.

## Reglas de negocio

- **Nombre obligatorio** — no puede faltar, ni ser cadena vacía, ni contener solo espacios en blanco.
- **Longitud máxima de 200 caracteres** — es el límite real de la columna `name`. Si dejas que lo valide la base de datos, el usuario recibe un `500` en vez de un `400`.
- **Normalización** — el nombre se guarda recortado: enviar `"  Router  "` debe persistir `"Router"`.
- **Sin duplicados** — si ya existe un dispositivo con ese nombre, la petición se rechaza con `409`. La comparación ignora mayúsculas y minúsculas, de forma coherente con cómo `DeviceName` define su propia igualdad (`OrdinalIgnoreCase`). Enviar `"router"` cuando ya existe `"Router"` debe dar conflicto.
- **Nunca debe escapar una excepción** — y aquí está el punto interesante de la práctica. El constructor de `DeviceName` lanza `ArgumentException` cuando recibe un valor vacío. El proyecto prohíbe usar excepciones para flujo de negocio esperado: si esa excepción llega al `ExceptionMiddleware`, el cliente recibe un `500` y la entrega queda inválida. Además, fíjate en que `DeviceName` valida `IsNullOrEmpty`, no `IsNullOrWhiteSpace`, y no valida longitud: el Value Object no te protege de todo. **Tu diseño debe garantizar que ningún `DeviceName` se construya nunca con un valor que no haya pasado antes por la validación.**

## Restricciones de arquitectura

Estas reglas salen de `CLAUDE.md` y `ARCHITECTURE.md`. Léelos: son la fuente de verdad y se evalúan.

- **Un único archivo por caso de uso.** Todo el slice va en `src/MasterNet.Application/Devices/DeviceCreate/CreateDeviceCommand.cs`, dentro de una clase contenedora que anida el `record` del request (que implementa `IRequest<Result<Guid>>` e `ICommandBase`), su `IRequestHandler<,>` y su `IValidator<T>`. Nada de carpetas separadas por tipo técnico (`Commands/`, `Handlers/`, `Validators/`).
- **El validador no se invoca a mano.** Ni desde el handler ni desde el controller. `ValidationBehavior<,>` (`src/MasterNet.Application/Core/ValidationBehavior.cs`) lo localiza y lo ejecuta antes del handler.
- **No registres nada en `Program.cs`.** Handlers y validadores se descubren por reflexión en `AddApplication()` (`src/MasterNet.Application/DependencyInjection.cs`). Si tu código necesita una línea nueva de DI, revisa qué convención estás rompiendo.
- **Los errores de negocio son `Result<T>`, no excepciones.**
- **El controller usa el mediator propio.** Inyecta `IMediatOR`, envía el comando y traduce con `this.FromResult(result, StatusCodes.Status201Created)`.
- **Escrituras por `IUnitOfWork`, lecturas por `DbContext`.** El alta pasa por `IUnitOfWork.Repository<Device>()`; la consulta de duplicado, por el `MasterNetDbContext` inyectado directamente en el handler. No crees un repositorio genérico nuevo para leer.
- **Prohibido añadir MediatR, AutoMapper, Mapster o FluentValidation.** El proyecto usa sus propias implementaciones (`libs/Core.MediatOR`, `libs/Core.Mappy`) y su propio `IValidator<T>`. Añadir cualquiera de esos paquetes invalida la entrega.
- **El endpoint queda `[AllowAnonymous]`**, igual que el `GET` del mismo controller. La autorización por políticas está fuera del alcance de esta práctica.
- **No rompas el `GET` existente.** Debe seguir funcionando exactamente igual.

## Pruebas

Las pruebas son obligatorias y **se escriben antes que el código**. El orden exigido es:

1. Escribe el test.
2. Ejecútalo y **compruébalo fallando**. Un test que nunca falló no demuestra nada.
3. Implementa lo mínimo para que pase.
4. Vuelve a ejecutarlo.

Repite el ciclo regla a regla. No escribas el handler completo y los tests después.

### Pruebas unitarias — `tests/MasterNet.Application.UnitTests/Devices/`

Deben cubrir:

- El **validador**, con un caso por cada regla de formato (ausente, vacío, solo espacios, más de 200 caracteres) y un caso válido que no produzca errores.
- El **handler**: alta correcta, fallo al guardar y nombre duplicado.

Como referencia de estilo tienes `tests/MasterNet.Application.UnitTests/Prices/CreatePriceCommandTests.cs` (handler de creación con `IUnitOfWork` mockeado con Moq).

> **Decisión de diseño que tienes que resolver tú:** el handler necesita consultar la base de datos para detectar el duplicado, así que mockear `IUnitOfWork` no basta. Tendrás que decidir cómo probarlo — por ejemplo con un proveedor EF Core en memoria dentro del proyecto de unitarias, o dejando ese caso concreto cubierto solo por las pruebas de integración. Elige una opción y **justifícala por escrito** en el cuerpo del commit o en un comentario del propio test. No hay una única respuesta correcta; sí hay que saber defender la que tomaste.

### Pruebas de integración — `tests/MasterNet.WebApi.IntegrationTests/Devices/`

Prueba los cuatro casos del contrato sobre HTTP real, con `MasterNetApiFactory` y `[Collection(ApiCollection.Name)]`. Usa `tests/MasterNet.WebApi.IntegrationTests/Students/StudentsEndpointsTests.cs` como modelo.

Tres avisos que te ahorrarán tiempo:

- **Requieren Docker en ejecución.** La factory levanta un SQL Server efímero con Testcontainers; sin Docker, toda la suite falla.
- **La base viene sembrada** con los nombres fijos de `device.json`, y todas las clases de test comparten el mismo contenedor. Genera nombres únicos en cada test (mira el helper `UniqueSuffix()` de `StudentsEndpointsTests.cs`) salvo, claro, cuando estés provocando el conflicto a propósito.
- Si verificas el alta releyendo con el `GET`, recuerda que ese endpoint devuelve la **entidad cruda** — `{"deviceName":{"value":"..."},"id":"..."}` — y no un DTO plano.

## Documentación y entrega

- Si tu implementación cambia el modelo de dominio, actualiza la entrada correspondiente en `docs/domain-glossary.md`. Si no lo cambia, déjalo intacto.
- Trabaja en una rama propia.
- El commit sigue **Conventional Commits y se escribe en español**, por ejemplo: `feat(devices): alta de dispositivos vía POST /api/devices`. Añade un cuerpo que explique **por qué** resolviste así los puntos no evidentes (el manejo del `ArgumentException` de `DeviceName`, la estrategia de pruebas del duplicado).
- Revisa el `diff` antes de commitear: no deben colarse archivos no relacionados.
- Cuando termines, sube tu rama a GitHub con `git push`.

**Cómo entregar:** copia el enlace a tu rama (por ejemplo, `https://github.com/tu-usuario/tu-repositorio/tree/nombre-de-tu-rama`) y envíalo en la tarea de la práctica final en Moodle.

## Criterios de aceptación

- [ ] `dotnet build MasterNet.sln` compila sin errores ni advertencias nuevas.
- [ ] `dotnet test MasterNet.sln` pasa en verde, incluidas las pruebas nuevas.
- [ ] Los cuatro casos del contrato responden con el código y el formato indicados.
- [ ] El nombre se persiste recortado.
- [ ] El duplicado se detecta ignorando mayúsculas y minúsculas.
- [ ] Ninguna petición produce un `500`, ni siquiera enviando `{ "name": "" }` o un cuerpo vacío.
- [ ] El slice completo vive en un solo archivo, con request, handler y validador anidados.
- [ ] El validador no se invoca manualmente en ningún punto.
- [ ] No se añadió ningún registro manual en `Program.cs`.
- [ ] No se añadió ningún paquete NuGet de mediator, mapeo automático o validación.
- [ ] `GET /api/devices` sigue funcionando igual que antes.
- [ ] El commit usa Conventional Commits, está en español y explica el porqué.

---

Curso **Claude Code .NET** · [TrainingIT](https://www.trainingit.es/producto/claude-code-net/)
