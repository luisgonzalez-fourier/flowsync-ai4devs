# Tasks

## 1. Modelo de datos (backend)

- [ ] 1.1 Crear `app/enums/task_status.ts` con `TASK_STATUSES` (`pending`, `in_progress` y `done`), sin imports del esquema generado. Crear con `node ace make:migration tasks` la tabla `tasks`:
  - `id`;
  - `title` `string(120)` no nulo;
  - `status` `enum(TASK_STATUSES)` no nulo y por defecto `pending`;
  - `assignee_id` no nulo con referencia a `users.id`;
  - `created_at` y `updated_at`.

  Ninguna columna de vencimiento. Se verifica con `node ace migration:run`: termina sin errores y `database/schema.ts` regenerado incluye `TaskSchema` con esas columnas.
- [ ] 1.2 Crear el modelo `Task`, que extiende `TaskSchema` con la relación `belongsTo` `assignee` hacia `User`, con `TASK_STATUSES` importada de `app/enums/task_status.ts`. Se verifica con `npm run typecheck` en `backend/`, sin errores.

## 2. API de tareas (backend)

- [ ] 2.1 Comprobar en los `.d.ts` de VineJS 4 y Lucid 22 las firmas de `vine.enum`, `.trim()`, `.optional()` y `.exists({ table, column })`. Crear `createTaskValidator` (`title`: `trim`, `minLength(1)`, `maxLength(120)`) y `updateTaskValidator` (`status` enum opcional, `assigneeId` número opcional que exista en `users`), y confirmar si `.optional()` acepta `null`; si es así, añadir el rechazo explícito con `422` que describe design.md. Se verifica con `npm run typecheck`, sin errores.
- [ ] 2.2 Crear `AssigneeTransformer` (solo `id` y `fullName`) y `TaskTransformer` (`id`, `title`, `status` y `assignee`, este último mediante `whenLoaded`). Se verifica con `npm run typecheck`, sin errores, y comprobando que ninguno de los dos hace `pick` de `email`.
- [ ] 2.3 Crear `TasksController` con tres acciones:
  - `index`, con `preload('assignee')` y **sin `orderBy`**, con un comentario que remita a PA-3;
  - `store`, con responsable `auth.user` y respuesta `201`;
  - `update`, con `findOrFail`, que solo aplica las claves presentes.

  Se verifica con `npm run typecheck`, sin errores.
- [ ] 2.4 Registrar en `start/routes.ts` el grupo `tasks` bajo `/api/v1` con `middleware.auth()`: `GET /`, `POST /` y `PATCH /:id` mediante `controllers.Tasks`. Arrancar `npm run dev` para regenerar `.adonisjs/`. Se verifica de dos formas:
  - `node ace list:routes` muestra exactamente esas tres rutas de tareas;
  - el diff de `.adonisjs/` incluye el controlador nuevo.
- [ ] 2.5 Actualizar `CLAUDE.md`: añadir las tres rutas de tareas a la tabla de rutas y mencionar `/tasks` como portada en la sección de frontend. Se verifica comprobando que la tabla coincide con la salida de `node ace list:routes`.
- [ ] 2.6 Comprobar a mano con `curl` contra el dev server cada escenario de API de `specs/tasks/spec.md`, haciendo antes una copia de `tmp/db.sqlite3` y restaurándola después:
  - `401` sin token;
  - creación con `201`, `pending` y quien crea como responsable, ignorando `status` y `assigneeId`;
  - título con espacios en los extremos, en blanco, de 120 y de 121 caracteres;
  - `422` por un estado desconocido, en castellano, `null` o `""`, y por `assigneeId: null`;
  - `PATCH` de `status` y de `assigneeId`, con un responsable inexistente (`422`) y una tarea inexistente (`404`);
  - `title` ignorado al actualizar;
  - `GET /api/v1/tasks/1` y `DELETE` devuelven `404`;
  - el responsable sin email en ninguna respuesta.

  No pegar tokens en el PR ni en los logs.

  Se verifica cuando todas las respuestas coinciden con la spec.
- [ ] 2.7 Ejecutar `npm run lint` y `npm run format` en `backend/`. Se verifica cuando el lint termina sin errores.

## 3. Cliente y sesión (frontend)

- [ ] 3.1 Añadir a `lib/types.ts` los tipos `TaskStatus`, `Task` (con `assignee: { id: number; fullName: string | null }`) y `CreateTaskPayload`, y crear la correspondencia de los estados con sus etiquetas "Pendiente", "En curso" y "Hecho". Se verifica con `npm run build` en `frontend/`, sin errores de tipos.
- [ ] 3.2 Ampliar el `method` de `request()` para que admita `'PATCH'` y añadir a `lib/api.ts` `listTasks`, `createTask` y `updateTaskStatus`, y la traducción de errores de `title`: `required`/`minLength` → "Escribe un título para la tarea." y `maxLength` → "El título no puede superar los 120 caracteres.". Se verifica con `npm run build`, sin errores.
- [ ] 3.3 Añadir `expireSession(message)` al `AuthContext` y al `AuthProvider`: limpia la sesión y fija `sessionError`. Se verifica con `npm run build` y `npm run lint`, sin errores.

## 4. Pantalla de lista (frontend)

- [ ] 4.1 Crear `pages/tasks-page.tsx` con:
  - la carga inicial, con indicador de carga y `Alert` de error;
  - el estado vacío explicativo;
  - la lista con título, `fullName?.trim() || 'Sin nombre'` y tres `Button` de estado con `aria-pressed`;
  - el enlace "Mi perfil".

  Solo se usan componentes de `components/ui/`, sin dependencias nuevas, y no se pinta ninguna fecha, email ni id. Se verifica con `npm run build` y `npm run lint`, sin errores, y con `git diff frontend/package.json` vacío.
- [ ] 4.2 Añadir el formulario de creación, con "Título" y "Añadir tarea" / "Añadiendo…":
  - comprueba en el navegador si el título va vacío o en blanco;
  - pinta el error de `title` con `FieldError`;
  - añade la tarea creada al final de la lista y vacía el campo;
  - conserva el texto si se rechaza.

  Se verifica con `npm run build`, sin errores.
- [ ] 4.3 Añadir el cambio de estado optimista, que deshabilita la fila mientras la petición está en curso, revierte la fila y muestra el `Alert` si falla, y llamar a `expireSession` ante cualquier `401` de la página. Se verifica con `npm run build`, sin errores.

## 5. Rutas y navegación (frontend)

- [ ] 5.1 Registrar `/tasks` dentro de `ProtectedRoute`, cambiar el comodín y la redirección de `PublicOnlyRoute` a `/tasks`, y añadir el `Link` "Ver tareas" en el perfil. Se verifica con `npm run build` y `npm run lint`, sin errores.

## 6. Comprobación integrada

- [ ] 6.1 Con backend y frontend en marcha, y con la BD copiada antes y restaurada después, recorrer en el navegador los escenarios web de `specs/tasks/spec.md` y del delta de `specs/auth/spec.md`:
  - el login y el registro llevan a `/tasks`;
  - una dirección desconocida lleva a `/tasks`;
  - `/tasks` sin sesión lleva a `/login`;
  - la lista vacía y la creación de la primera tarea;
  - los errores de título vacío, en blanco y de 130 caracteres;
  - el cambio de estado con un clic sobre una tarea propia y otra ajena;
  - el error con el backend parado;
  - "Sin nombre" para un responsable sin nombre;
  - los enlaces entre lista y perfil.

  Se verifica cuando todo se comporta como dicen las specs.
- [ ] 6.2 Ejecutar `openspec validate add-task-list --strict`. Se verifica cuando el change es válido.
