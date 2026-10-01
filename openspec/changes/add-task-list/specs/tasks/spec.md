# Spec Delta

## Purpose

Mantener una única lista de tareas compartida por todo el equipo, en la que cualquiera apunta una tarea con solo su título y ve, sin abrir nada, quién la lleva y en qué estado está, y puede cambiar ese estado desde la propia lista.

## ADDED Requirements

### Requirement: Forma pública de una tarea

Siempre que la API devuelva una tarea, SHALL incluir exactamente `id` (número), `title` (texto), `status` (uno de `pending`, `in_progress` o `done`) y `assignee`. `assignee` SHALL ser un objeto con exactamente `id` (número) y `fullName` (texto o `null`) de la persona responsable. La API MUST NOT devolver en una tarea el email del responsable ni ningún otro dato de su cuenta, y MUST NOT devolver ninguna fecha de vencimiento ni marca de vencida.

#### Scenario: Tarea con responsable con nombre

- **WHEN** Ada Lovelace (`ada@example.com`) es la responsable de una tarea y se lista
- **THEN** la tarea trae `assignee` con su `id` y `fullName` `"Ada Lovelace"`, y en ninguna parte aparece `"ada@example.com"`

#### Scenario: Responsable sin nombre

- **WHEN** la responsable de una tarea se registró sin nombre
- **THEN** la tarea trae `assignee.fullName` `null` y no se sustituye por su email

#### Scenario: Sin fechas en la tarea

- **WHEN** se lista, se crea o se actualiza una tarea
- **THEN** el objeto tarea no contiene ninguna clave de fecha de vencimiento ni de vencida

### Requirement: Las tareas exigen sesión

Las operaciones sobre tareas SHALL exigir la cabecera `Authorization: Bearer <token>` con un token vigente. Sin token, o con un token desconocido o revocado, la API SHALL responder `401` con `{ "errors": [ { "message": "Unauthorized access" } ] }`, sin devolver ni modificar ninguna tarea.

#### Scenario: Listar sin sesión

- **WHEN** se pide `GET /api/v1/tasks` sin cabecera `Authorization`
- **THEN** la respuesta es `401` y no incluye ninguna tarea

#### Scenario: Crear sin sesión

- **WHEN** se envía `POST /api/v1/tasks` con un token revocado
- **THEN** la respuesta es `401` y no se crea ninguna tarea

### Requirement: Solo tres operaciones sobre tareas

La API SHALL ofrecer exactamente tres operaciones sobre tareas: listar todas (`GET /api/v1/tasks`), crear una (`POST /api/v1/tasks`) y actualizar una (`PATCH /api/v1/tasks/:id`). La API MUST NOT ofrecer lectura individual de una tarea, borrado de tareas ni ningún endpoint de equipo o de personas.

#### Scenario: No hay lectura individual

- **WHEN** se pide `GET /api/v1/tasks/1` con sesión y la tarea 1 existe
- **THEN** la respuesta es `404` y no devuelve la tarea

#### Scenario: No hay borrado

- **WHEN** se envía `DELETE /api/v1/tasks/1` con sesión y la tarea 1 existe
- **THEN** la respuesta es `404` y la tarea sigue existiendo

### Requirement: Una sola lista compartida

`GET /api/v1/tasks` SHALL responder `200` con `{ "data": [ <tarea>, ... ] }`, con todas las tareas que existen, sean de quien sean. El resultado MUST NOT depender de quién lo pide: no hay tareas privadas, filtros por persona ni contenido reservado a ningún rol. La API MUST NOT aplicar ningún criterio de ordenación explícito, y quien la consuma MUST NOT dar por estable el orden en que llegan. Listar MUST NOT cambiar ninguna tarea.

#### Scenario: Dos personas ven lo mismo

- **WHEN** dos personas distintas, cada una con su token, piden la lista sin que nadie cambie nada entre medias
- **THEN** las dos reciben exactamente el mismo conjunto de tareas

#### Scenario: La tarea que otro se asignó también aparece

- **WHEN** Grace crea una tarea, que queda a su nombre, y después Ada pide la lista
- **THEN** la lista de Ada incluye la tarea de Grace

#### Scenario: Espacio sin tareas

- **WHEN** se pide la lista y no se ha creado ninguna tarea
- **THEN** la respuesta es `200` con `{ "data": [] }`

#### Scenario: Listar no modifica nada

- **WHEN** se pide la lista varias veces seguidas
- **THEN** ninguna tarea cambia de título, de estado ni de responsable

### Requirement: Crear una tarea con solo el título

`POST /api/v1/tasks` SHALL aceptar un cuerpo JSON con `title` y, si es válido, crear la tarea y responder `201` con `{ "data": <tarea> }`. La tarea creada SHALL tener `status` `pending` y como responsable a la persona dueña del token, sin que lo indique. La API SHALL guardar el título sin los espacios de los extremos y SHALL ignorar cualquier otro campo del cuerpo, como `status` o `assigneeId`.

#### Scenario: Crear con un título

- **WHEN** Ada envía `{ "title": "Preparar la demo" }`
- **THEN** la respuesta es `201` con una tarea de `title` `"Preparar la demo"`, `status` `"pending"` y `assignee.fullName` `"Ada Lovelace"`, y esa tarea aparece después en la lista

#### Scenario: Los campos extra no cuentan

- **WHEN** Ada envía `{ "title": "Revisar el PR", "status": "done", "assigneeId": <id de Grace> }`
- **THEN** la tarea creada tiene `status` `"pending"` y Ada como responsable

#### Scenario: Espacios en los extremos

- **WHEN** se crea una tarea con `"title": "  Preparar la demo  "`
- **THEN** la tarea queda con `title` `"Preparar la demo"`

### Requirement: Título obligatorio y acotado

La API SHALL rechazar la creación con `422` y `{ "errors": [ ... ] }`, y MUST NOT crear ninguna tarea, cuando `title` falta, está vacío, contiene solo espacios o tiene más de 120 caracteres después de quitar los espacios de los extremos. Un título de 120 caracteres o menos SHALL aceptarse completo, y la API MUST NOT recortar nunca un título para que quepa.

#### Scenario: Sin título

- **WHEN** se envía `{}`
- **THEN** la respuesta es `422` con un error de `field` `"title"` y no se crea ninguna tarea

#### Scenario: Título solo con espacios

- **WHEN** se envía `{ "title": "    " }`
- **THEN** la respuesta es `422` con un error de `field` `"title"`, igual que si faltara, y la lista no gana ninguna fila

#### Scenario: Título demasiado largo

- **WHEN** se envía un `title` de 121 caracteres
- **THEN** la respuesta es `422` con un error de `field` `"title"`, `rule` `"maxLength"` y `meta.max` `120`, y no se guarda ninguna versión recortada

#### Scenario: Título justo en el límite

- **WHEN** se envía un `title` de exactamente 120 caracteres
- **THEN** la respuesta es `201` y la tarea guarda los 120 caracteres

### Requirement: Tres estados fijos

El estado de una tarea SHALL ser siempre exactamente uno de `pending`, `in_progress` o `done`. Estos identificadores son el contrato de la API. La API MUST NOT aceptar otros valores, incluidos los nombres en castellano, y MUST NOT ofrecer forma alguna de añadir, renombrar ni eliminar estados.

#### Scenario: Valor desconocido

- **WHEN** se actualiza una tarea con `{ "status": "blocked" }`
- **THEN** la respuesta es `422` con un error de `field` `"status"` y la tarea conserva su estado

#### Scenario: Estado nulo o vacío

- **WHEN** se actualiza una tarea con `{ "status": null }` o `{ "status": "" }`
- **THEN** la respuesta es `422` con un error de `field` `"status"` y la tarea conserva su estado

#### Scenario: Nombre en castellano

- **WHEN** se actualiza una tarea con `{ "status": "Hecho" }`
- **THEN** la respuesta es `422` con un error de `field` `"status"` y la tarea conserva su estado

### Requirement: Actualizar estado y responsable de cualquier tarea

`PATCH /api/v1/tasks/:id` SHALL aceptar un cuerpo JSON con `status` y/o `assigneeId`, aplicar los que vengan y responder `200` con `{ "data": <tarea> }` ya actualizada. Cualquier persona con sesión SHALL poder actualizar cualquier tarea, sea quien sea su responsable, sin permisos especiales. Se SHALL poder pasar de cualquier estado a cualquier otro, incluida la vuelta atrás desde `done`. La API SHALL:

- ignorar cualquier otro campo del cuerpo, incluido `title`;
- responder `200` sin cambios si no viene ninguno de los dos campos;
- responder `422` con un error de `field` `"assigneeId"` si `assigneeId` es `null` o no corresponde a ninguna persona registrada;
- responder `404` si la tarea no existe.

#### Scenario: Cambiar el estado de una tarea ajena

- **WHEN** Ada actualiza con `{ "status": "in_progress" }` una tarea cuyo responsable es Grace
- **THEN** la respuesta es `200` con `status` `"in_progress"`, la tarea sigue siendo de Grace y la lista lo refleja

#### Scenario: Volver atrás desde hecho

- **WHEN** una tarea en `done` se actualiza con `{ "status": "pending" }`
- **THEN** la respuesta es `200` y la tarea vuelve a `pending`

#### Scenario: Reasignar por la API

- **WHEN** se actualiza una tarea con `{ "assigneeId": <id de Grace> }`
- **THEN** la respuesta es `200` con `assignee.fullName` `"Grace"` y el estado sin cambios

#### Scenario: Responsable inexistente

- **WHEN** se actualiza una tarea con un `assigneeId` que no es de nadie
- **THEN** la respuesta es `422` con un error de `field` `"assigneeId"` y la tarea conserva su responsable

#### Scenario: El título no se edita por aquí

- **WHEN** se actualiza una tarea con `{ "title": "Otro título" }`
- **THEN** la respuesta es `200` y el título no cambia

#### Scenario: Tarea inexistente

- **WHEN** se actualiza la tarea `999999`, que no existe
- **THEN** la respuesta es `404`

### Requirement: Pantalla de la lista del equipo

La aplicación web SHALL mostrar en `/tasks`, a una persona con sesión, la lista de todas las tareas en el orden en que las devuelve la API, la misma para todo el mundo. Cada fila SHALL mostrar, sin abrir nada, el título, el nombre del responsable y el estado como "Pendiente", "En curso" o "Hecho". El responsable SHALL identificarse por su nombre completo, o por "Sin nombre" si no tiene o si su nombre está en blanco, y MUST NOT mostrarse nunca su email ni su id. La pantalla MUST NOT mostrar fechas, marcas de vencida ni señales de presencia o de actividad por persona. La aplicación MUST NOT ofrecer ninguna otra vista de tareas, como "mis tareas", ni ningún filtro.

#### Scenario: Cada fila responde quién y en qué estado

- **WHEN** hay una tarea "Preparar la demo" de Ada Lovelace en curso y una tarea "Revisar el PR" de una persona sin nombre pendiente
- **THEN** la lista muestra "Preparar la demo" con "Ada Lovelace" y "En curso", y "Revisar el PR" con "Sin nombre" y "Pendiente"

#### Scenario: Nada de correos ni ids

- **WHEN** se recorre la lista entera
- **THEN** en ninguna fila aparece el email ni el id de su responsable

#### Scenario: Mientras carga

- **WHEN** la persona abre `/tasks` y la lista todavía no ha llegado
- **THEN** ve un indicador de carga en lugar de la lista

#### Scenario: La lista no carga

- **WHEN** la persona abre `/tasks` y el servidor no responde
- **THEN** ve un aviso de error con "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." en lugar de una lista vacía

### Requirement: Lista vacía

Cuando no existe ninguna tarea, la aplicación web SHALL sustituir la lista por un mensaje que explique que aquí se ve todo el trabajo del equipo y que invite a crear la primera tarea con el formulario de creación. MUST NOT mostrar una lista vacía sin explicación.

#### Scenario: Primera visita a un espacio sin tareas

- **WHEN** una persona abre `/tasks` y no hay ninguna tarea
- **THEN** ve el mensaje explicativo con la invitación a crear la primera tarea y el formulario de creación disponible

### Requirement: Crear una tarea desde la lista

La pantalla de lista SHALL ofrecer un formulario con un único campo, "Título", y un botón "Añadir tarea". El formulario MUST NOT pedir, ofrecer ni sugerir responsable, estado, fecha ni ningún otro dato. Al crearse la tarea, SHALL aparecer en la lista sin recargar ni navegar, como "Pendiente" y con el nombre de quien la creó, y el campo SHALL quedar vacío. Mientras el envío está en curso, el botón SHALL deshabilitarse y mostrar "Añadiendo…".

#### Scenario: Crear desde la lista

- **WHEN** Ada escribe "Preparar la demo" y pulsa "Añadir tarea"
- **THEN** sin recargar, aparece en la lista una fila "Preparar la demo" con "Ada Lovelace" y "Pendiente", y el campo "Título" queda vacío

#### Scenario: Crear la primera tarea

- **WHEN** no hay tareas y la persona crea una
- **THEN** el mensaje de lista vacía desaparece y en su lugar se ve la lista con esa única tarea

#### Scenario: Sin selector de nada más

- **WHEN** la persona recorre el formulario de creación
- **THEN** solo encuentra el campo "Título" y el botón "Añadir tarea", sin selector de responsable, de estado ni de fecha

### Requirement: Errores del título en pantalla

La aplicación web SHALL explicar en castellano, debajo del campo "Título", por qué no se ha creado la tarea, y MUST NOT añadir ninguna fila a la lista en ese caso. Un título vacío o solo con espacios SHALL detectarse en el navegador sin llamar al servidor, con "Escribe un título para la tarea.". El navegador SHALL medir el título igual que el servidor, sin los espacios de los extremos. Un título de más de 120 caracteres SHALL mostrar "El título no puede superar los 120 caracteres." y conservar en el campo el texto escrito, sin recortarlo.

#### Scenario: Título vacío

- **WHEN** la persona pulsa "Añadir tarea" con el campo vacío
- **THEN** aparece "Escribe un título para la tarea." bajo el campo, no se envía nada al servidor y la lista no cambia

#### Scenario: Título solo con espacios en pantalla

- **WHEN** la persona escribe solo espacios y pulsa "Añadir tarea"
- **THEN** aparece el mismo aviso que con el campo vacío y la lista no gana ninguna fila

#### Scenario: Título demasiado largo en pantalla

- **WHEN** la persona escribe un título de 130 caracteres y pulsa "Añadir tarea"
- **THEN** aparece "El título no puede superar los 120 caracteres." bajo el campo, el texto escrito sigue entero en el campo y no se crea ninguna tarea

### Requirement: Cambiar el estado desde la fila

Cada fila de la lista SHALL ofrecer, a la vista, los tres estados "Pendiente", "En curso" y "Hecho", con el actual marcado, como únicos destinos posibles. Un solo clic sobre otro estado SHALL cambiarlo, sin abrir la tarea, sin diálogos de confirmación y sin rellenar ningún campo. Esto SHALL funcionar igual en cualquier tarea, sea quien sea su responsable, y sin advertencias. El nuevo estado SHALL reflejarse en la fila de inmediato, sin esperar a la respuesta del servidor. Mientras el cambio de una fila está pendiente de respuesta, sus botones de estado SHALL quedar deshabilitados. Si el servidor rechaza el cambio o no responde, la fila SHALL volver al estado anterior y SHALL mostrarse un aviso de error.

#### Scenario: Marcar en curso con un clic

- **WHEN** la persona pulsa "En curso" en una tarea pendiente
- **THEN** la fila pasa a marcar "En curso" al momento, sin ningún diálogo, y al recargar la lista sigue en curso

#### Scenario: Tarea de otra persona

- **WHEN** Ada pulsa "Hecho" en una tarea cuyo responsable es Grace
- **THEN** el cambio se aplica igual que en una tarea propia, sin advertencias, y la tarea sigue a nombre de Grace

#### Scenario: Solo tres destinos

- **WHEN** la persona mira cómo cambiar el estado de una fila
- **THEN** solo se ofrecen "Pendiente", "En curso" y "Hecho", y la tarea queda siempre en exactamente uno de ellos

#### Scenario: El cambio falla

- **WHEN** la persona pulsa "Hecho" y el servidor no responde
- **THEN** la fila vuelve a marcar el estado que tenía y aparece un aviso con "No se pudo conectar con el servidor. Comprueba que el backend está arrancado."

### Requirement: Sesión perdida mientras se usa la lista

Si el servidor rechaza la sesión (`401`) al cargar la lista, al crear una tarea o al cambiar un estado, la aplicación web SHALL cerrar la sesión en el navegador y llevar a la persona a `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión.".

#### Scenario: Token revocado con la lista abierta

- **WHEN** el token de la persona se revoca en otra pestaña y después pulsa un estado en la lista
- **THEN** acaba en `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión."

### Requirement: Navegación entre lista y perfil

La pantalla de lista SHALL ofrecer un enlace "Mi perfil" que lleva a `/profile`.

#### Scenario: De la lista al perfil

- **WHEN** la persona pulsa "Mi perfil" en `/tasks`
- **THEN** ve su perfil en `/profile`

### Requirement: La lista exige sesión

La aplicación web SHALL permitir `/tasks` solo con sesión iniciada. Sin sesión, SHALL llevar a `/login` sin mostrar ninguna tarea. Mientras se verifica una sesión guardada, SHALL mostrar el indicador de carga.

#### Scenario: Lista sin sesión

- **WHEN** una persona sin sesión abre `/tasks`
- **THEN** es llevada a `/login` y no ha visto ninguna tarea
