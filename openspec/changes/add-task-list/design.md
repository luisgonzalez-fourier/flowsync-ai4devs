# Design

## Context

Hoy el backend solo tiene el vertical de auth:

- un modelo de usuario que extiende el esquema generado desde las migraciones;
- validadores VineJS creados con `vine.create`;
- transformers `BaseTransformer` que se sirven con `serialize()`, que envuelve la respuesta en `{ data }`;
- rutas en `start/routes.ts` que apuntan al mapa de controladores generado en `.adonisjs/`;
- un grupo protegido con `middleware.auth()`.

El body parser convierte las cadenas vacías en `null`, pero una cadena hecha solo de espacios llega intacta.

El frontend centraliza toda llamada a la API en `lib/api.ts`. Ahí se traducen los errores de VineJS a castellano según el campo y la regla.

Las páginas siguen un patrón:

- un layout de tarjeta;
- los componentes `Card`, `Button`, `Input`, `Label` y `Alert` de `components/ui/`;
- `FieldError` para el error de cada campo.

Los guards `ProtectedRoute` y `PublicOnlyRoute` redirigen a `/profile`, y la ruta comodín también. La sesión vive en `AuthProvider`, que solo reacciona a un `401` al rehidratar la sesión al cargar.

El alcance y los puntos abiertos están en `proposal.md`, y el comportamiento en `specs/tasks/spec.md` y `specs/auth/spec.md`.

## Goals / Non-Goals

**Goals:**

- Seguir al pie de la letra las convenciones existentes en las dos capas: migración → esquema generado, transformers, `serialize()`, `lib/api.ts` como único cliente, y componentes de `ui/` ya instalados.
- Que el responsable salga de la API con el mínimo de datos (`id` y `fullName`) desde el primer día, porque recortar después rompería a quien ya los use.

**Non-Goals:**

- Ningún cambio en cómo se autentica, ni en la forma de las respuestas de auth.
- Ninguna capa de estado global ni caché para las tareas: la página tiene su propio estado.
- No se deja preparado nada para la fecha de vencimiento, ni columnas, ni campos, ni tipos.
- No se añaden tests ni base de pruebas.

## Decisions

### Modelo de datos: una tabla `tasks`

La migración nueva tiene estas columnas:

- `id` autoincremental;
- `title`, `string(120)`, no nulo;
- `status`, no nulo y por defecto `'pending'`, restringido a `pending`, `in_progress` y `done` con `table.enum`, que en SQLite genera un `CHECK`;
- `assignee_id`, entero no nulo con referencia a `users.id`;
- `created_at` y `updated_at`.

`created_at` y `updated_at` se rellenan pero no se exponen: la spec prohíbe fechas en la lista y no hacen falta.

No se guarda quién creó la tarea, aparte de usarlo como responsable inicial. Ninguna historia de este change lo usa y añadirlo sería preparar trabajo futuro.

El modelo `Task` extiende el esquema generado y declara solo la relación `belongsTo` `assignee` hacia `User`. Los estados se definen una sola vez en el backend, como una constante `TASK_STATUSES` en un módulo propio, `app/enums/task_status.ts`, sin dependencias del esquema generado, que reutilizan el modelo, el validador y la migración. No puede vivir en el modelo: el modelo importa `#database/schema`, y ese fichero no contiene `TaskSchema` hasta después del primer `migration:run`, así que una migración que importase el modelo fallaría en su primera ejecución.

*Alternativa descartada:* una tabla de estados. Los estados son fijos por requisito (RF-8), y una tabla invitaría a editarlos.

### Validación

Hay dos validadores en un fichero nuevo de validadores de tareas.

- **`createTaskValidator`:** `{ title: vine.string().trim().minLength(1).maxLength(120) }`.
  - El `trim()` hace que `"    "` se quede en `""` y lo rechace `minLength(1)`, porque `required` por sí solo no lo detecta: la cadena llega presente.
  - Como VineJS solo devuelve las claves declaradas, los campos extra (`status`, `assigneeId`) se ignoran sin código adicional.
- **`updateTaskValidator`:** `{ status: vine.enum(TASK_STATUSES).optional(), assigneeId: vine.number().exists({ table: 'users', column: 'id' }).optional() }`.
  - El body parser convierte `""` en `null`. Si `.optional()` de VineJS 4 acepta `null` como ausente, `{ "status": null }` pasaría como no-op, y la spec exige `422`. Hay que comprobarlo en sus `.d.ts` y en su comportamiento. Si ocurre, se rechaza de forma explícita: el controlador comprueba con `request.input` si la clave viene con `null` y responde `422` con el mismo formato de error, o bien se usa una regla propia de VineJS. Lo mismo vale para `assigneeId: null`.
  - `title` no se declara, así que se ignora.
  - Un cuerpo vacío valida y deja la tarea como está.

Antes de escribir `exists` y `enum` hay que comprobar sus firmas en los `.d.ts` de VineJS 4 y de Lucid 22, igual que con `unique` en auth.

### API: controlador `TasksController` con `index`, `store` y `update`

Las rutas van en un grupo nuevo, `router.group(...).prefix('tasks').use(middleware.auth())`, dentro del prefijo `/api/v1`:

- `GET /` → `index`;
- `POST /` → `store`;
- `PATCH /:id` → `update`.

Un `GET /:id` o un `DELETE /:id` no casan con ninguna ruta, así que el router responde `404` por sí solo, sin código adicional.

- **`index`:** `Task.query().preload('assignee')`, **sin `orderBy`**. Es una decisión explícita mientras PA-3 siga abierto, y se deja con un comentario que lo diga para que nadie añada un orden "por limpieza".
- **`store`:** valida, hace `Task.create({ title, assigneeId: auth.user.id })` con el estado por defecto, carga el responsable y responde `201` con `response.status(201)` y el `serialize` de la tarea.
- **`update`:**
  - hace `Task.findOrFail(params.id)`, así que un id que no existe, o que no es numérico, da `404`;
  - valida, hace `merge` solo de las claves presentes y guarda;
  - recarga el responsable y responde.

Cualquier persona con sesión puede actualizar cualquier tarea. No hay comprobaciones de propiedad, y eso es un requisito, no un descuido.

*Alternativa descartada:* `PUT` con la tarea entera. Obligaría a mandar el título, que este change no deja editar.

### Transformers: el responsable sale recortado

`TaskTransformer` hace `pick` de `id`, `title` y `status`, y añade `assignee: AssigneeTransformer.transform(this.whenLoaded(this.resource.assignee))`. `AssigneeTransformer` es un transformer nuevo que solo hace `pick` de `id` y `fullName`.

No se reutiliza `UserTransformer`, porque expone `email`, `createdAt`, `updatedAt` e `initials`, y la nota de E3-1 pide exactamente lo contrario.

Al tocar rutas y controladores hay que regenerar `.adonisjs/`, arrancando el dev server, y commitear el diff.

### Frontend: cliente, tipos y página

- **`lib/types.ts`:** añade los tipos `TaskStatus = 'pending' | 'in_progress' | 'done'`, `Task` y `CreateTaskPayload`.
- **`lib/api.ts`:** amplía el tipo `method` de `request()` para que admita `'PATCH'` y añade `listTasks(token)`, `createTask(token, { title })` y `updateTaskStatus(token, id, status)`. Esta última manda solo `{ status }`, porque la UI no reasigna. En la traducción de errores:
  - `FIELD_LABELS` gana `title: 'el título'`;
  - `translate` trata `title` de forma específica: `required`/`minLength` → "Escribe un título para la tarea." y `maxLength` → "El título no puede superar los 120 caracteres.".

  Así la frase sale en mayúscula y como pide la spec, en vez de usar la plantilla genérica.
- **`lib/task-status.ts`** (o una constante en la página): la correspondencia entre identificador y etiqueta, `pending` → "Pendiente", `in_progress` → "En curso" y `done` → "Hecho". Es el único sitio del frontend donde aparecen las etiquetas en castellano. Los identificadores nunca se traducen ni se usan las etiquetas como valor.
- **`pages/tasks-page.tsx`:** una página nueva con el mismo marco visual que el perfil (`bg-muted/40`, `Card`), más ancha. Contiene:
  - una cabecera con "FlowSync", el título de la pantalla y el enlace "Mi perfil";
  - el formulario de creación, con `Input` y `Label` "Título" y `Button` "Añadir tarea", que comprueba en el navegador si el título va vacío o en blanco antes de llamar;
  - un estado de carga, que reutiliza `FullScreenLoader` o un spinner equivalente dentro de la tarjeta;
  - un `Alert` destructivo si la carga o un cambio de estado fallan;
  - el estado vacío, un texto explicativo dentro de la tarjeta que señala el formulario;
  - la lista, en un `<ul>`: cada `<li>` muestra el título, `assignee.fullName?.trim() || 'Sin nombre'` (la API admite nombres solo con espacios) y un grupo de tres `Button` (`variant="default"` para el estado actual y `"outline"` para los demás, con `aria-pressed`).

  Los tres botones hacen a la vez de indicador de estado y de control de un solo clic. Así no hace falta ningún componente nuevo, como `Select` o `ToggleGroup`, ni dependencias nuevas.
- **Cambio de estado optimista:** se actualiza la fila en el estado local, se deshabilitan los botones de esa fila mientras la petición está en curso, se llama a la API y, si falla, se restaura el estado anterior de esa fila y se muestra el `Alert`. Al bloquear la fila, dos clics rápidos no pueden hacer que una reversión pise el estado bueno. Con éxito, la fila se sustituye por la tarea que devuelve el servidor.
- **Creación:** la tarea devuelta se añade **al final** del array local. No se ordena, y es coherente con "no hay regla de orden".
- **`401` durante el uso:** `AuthContext` gana `expireSession(message)`, que reutiliza `clearSession` y fija `sessionError`. La página la llama ante un `ApiError` con `status === 401`, y `ProtectedRoute` lleva a `/login`, donde el aviso ya se pinta hoy.

  *Alternativa descartada:* gestionar el `401` dentro de `lib/api.ts`. Acoplaría el cliente HTTP al estado de React.
- **Rutas:**
  - `app-routes.tsx` añade `/tasks` dentro de `ProtectedRoute` y cambia el comodín a `/tasks`;
  - `PublicOnlyRoute` redirige a `/tasks`;
  - `profile-page.tsx` añade un `Link` "Ver tareas".

  Login y registro no necesitan cambios propios, porque su redirección la hace `PublicOnlyRoute` al pasar a `authenticated`.

## Risks / Trade-offs

- **[Orden implícito]** Sin `orderBy`, SQLite devuelve hoy las filas por `rowid`, casi siempre en orden de creación. Alguien puede acabar dependiendo de eso. → Comentario explícito en la consulta, el punto abierto PA-3 en el proposal y la frase de la spec "no dar por estable el orden".
- **[Lista desactualizada]** Sin refresco automático, dos personas pueden ver estados distintos hasta recargar, y un cambio optimista puede pisar el de otra persona. → Aceptado: E3-2 lo resuelve, y la spec define la lista como correcta en el momento en que se pide.
- **[Marcar hecho por error]** Un solo clic y sin confirmación (PA-7). → Mitigado porque "Hecho" no oculta la tarea y se puede volver atrás con otro clic.
- **[Reasignación sin UI]** La API acepta `assigneeId`, pero nada en pantalla lo usa, así que solo se puede comprobar con peticiones manuales. → Aceptado por alcance: la UI llega con la historia de reasignar.
- **[Longitud del título]** VineJS mide en unidades UTF-16 (un emoji cuenta 2) y SQLite no impone el `string(120)`, así que el único guardián real es el validador. → El cliente mide igual (`title.trim().length`) y el servidor manda.
- **[Volumen]** La lista se pide entera, sin paginar. Con el volumen previsto por el PRD (unas 200 tareas) cabe sin problema. → No se pagina hasta que haga falta.
- **[Sin tests]** No hay red de seguridad automática. → La verificación de tasks.md se apoya en el typecheck, el lint y el build de las dos capas, y en una comprobación manual con el servidor de desarrollo.

## Migration Plan

- Desplegar es ejecutar `node ace migration:run`, que crea la tabla y regenera `database/schema.ts`.
- El rollback es `node ace migration:rollback`, que borra la tabla, y revertir el commit.
- No hay datos previos que migrar.
- El cambio de portada de `/profile` a `/tasks` no rompe nada guardado: quien tenga `/profile` en marcadores sigue llegando a su perfil.
