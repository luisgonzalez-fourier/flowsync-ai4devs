# Proposal

## Why

FlowSync todavía no tiene tareas: solo se puede crear una cuenta, entrar y ver el perfil. Este change trae la base de las épicas E2 y E3. Es una única lista compartida en la que cualquiera del equipo apunta una tarea escribiendo solo el título, ve quién lleva cada una y en qué estado está, y cambia ese estado sin salir de la lista. Sin esta base no hay dónde construir el resto del backlog: abrir, editar, reasignar desde la pantalla, filtrar, vencimientos o la lista viva.

Historias que cubre:

- E3-1 · La lista compartida del equipo.
- E2-1 · Crear tarea con solo el título.
- E2-2 · Título obligatorio.
- E2-3 · Nace mía y pendiente.
- E2-4 · Cambiar el estado desde la lista.

## What Changes

- **Tareas en la API.** Una tarea nueva tiene título, estado y responsable. El estado es uno de tres valores fijos que viajan como `pending`, `in_progress` y `done`. Cualquier otro valor se rechaza con `422`.
- **Exactamente tres operaciones**, todas con sesión iniciada: listar todas las tareas, crear una y actualizar una.
  - No hay lectura individual, ni borrado, ni endpoints de equipo o de personas.
  - Al crear solo se acepta el título. La tarea nace en `pending` y su responsable es quien la crea.
  - Al actualizar se acepta el estado y/o el responsable de cualquier tarea, por parte de cualquier persona con sesión. Editar el título por la API no entra en este change.
- **Título obligatorio.** Vacío o solo con espacios se rechaza. Si supera los **120 caracteres** también se rechaza con un aviso, y nunca se recorta en silencio.
- **El responsable se expone por su nombre.** La API solo devuelve `id` y `fullName` del responsable, nunca su email ni otros datos de cuenta. En pantalla se pinta el nombre o "Sin nombre", nunca el correo ni el id.
- **Pantalla de lista (`/tasks`).** Es la misma para todos y no hay vista "mis tareas". Cada fila muestra el título, el nombre del responsable y el estado, que se cambia en un solo gesto entre "Pendiente", "En curso" y "Hecho".
  - En la parte de arriba hay un formulario con un único campo, el título, y la tarea creada aparece en la lista sin recargar.
  - Si la lista está vacía se explica qué es y se invita a crear la primera tarea.
- **`/tasks` pasa a ser la portada.** Al iniciar sesión, al registrarse, al abrir `/login` o `/register` con sesión y al abrir una dirección desconocida se acaba en `/tasks`, no en `/profile`. Hay enlaces entre la lista y el perfil.
- **Fuera de alcance, a propósito:**
  - fecha de vencimiento y cualquier marca de vencida;
  - abrir una tarea, editar el título, borrar;
  - reasignar desde la pantalla, porque no hay endpoint para listar personas y la UI no ofrece ninguna forma de elegir responsable;
  - filtro por estado;
  - refresco automático de la lista (E3-2);
  - señales de presencia;
  - tests.

## Puntos abiertos

- **Orden de la lista (PA-3).** No hay regla de orden decidida, así que la API no ordena explícitamente y la lista muestra las tareas en el orden en que llegan. La tarea recién creada se añade al final de la vista. Ese orden de llegada no es un contrato: no debe darse por estable ni construirse nada encima hasta que se decida PA-3.
- **Límite del título (PA-9).** Los 120 caracteres son una decisión provisional de este change para poder cumplir el CA-3 de E2-2 ("avisar, no recortar"). El PRD sigue sin fijar el umbral.
- **Transiciones de estado (PA-7).** No hay grafo de transiciones decidido. En este change se puede pasar de cualquier estado a cualquier otro, incluida la vuelta atrás desde "Hecho".
- **Cuántas tareas "En curso" por persona (PA-4)** y **qué se ve cuando otro cambia la tarea que miras (PA-8):** siguen sin decidir y no se abordan.

## Capabilities

### New Capabilities

- `tasks`: la lista compartida de tareas del equipo. Cubre la API de listar, crear y actualizar; las reglas del título, del estado y del responsable por defecto; y la pantalla de lista con su formulario de creación y el cambio de estado desde la fila.

### Modified Capabilities

- `auth`: las pantallas de registro e inicio de sesión, y la sesión recuperada al recargar, llevan ahora a la lista en vez de al perfil; la protección de pantallas redirige a `/tasks` en lugar de a `/profile`; el perfil enlaza de vuelta a la lista.

## Impact

- **Backend:**
  - una migración nueva para la tabla de tareas, con clave ajena al usuario responsable;
  - modelo, validadores, transformer y controlador nuevos;
  - tres rutas bajo `/api/v1/tasks` protegidas con el middleware de auth;
  - se regeneran `database/schema.ts` y el código generado de `.adonisjs/`, que hay que commitear.
- **Frontend:**
  - funciones nuevas en el cliente de la API, que también traduce los errores del título;
  - tipos nuevos;
  - una página y una ruta protegida nuevas;
  - cambios de redirección en los guards y en la ruta por defecto, y un enlace en el perfil.

  Reutiliza los componentes de `components/ui/` que ya existen. No se añaden dependencias ni componentes de shadcn nuevos.
- **Documentación:** `CLAUDE.md` actualiza la tabla de rutas y menciona `/tasks` como portada.
- **Sin tests:** este change no monta base de pruebas ni añade tests.
- **Spec base:** el delta de `auth` modifica la spec viva añadida en el PR #1. Este change parte de esa rama.
