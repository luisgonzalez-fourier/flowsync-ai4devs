# auth Specification

## Purpose

Permitir que una persona cree una cuenta en FlowSync con su email y una contraseña, inicie y cierre sesión, y consulte su perfil, tanto a través de la API HTTP como desde la aplicación web.

## Requirements

### Requirement: Registro de cuenta por la API

La API SHALL aceptar `POST /api/v1/auth/signup` con un cuerpo JSON que contenga `fullName` (texto o `null`), `email`, `password` y `passwordConfirmation`. Si el cuerpo es válido, SHALL crear la cuenta y responder `200` con `{ "data": { "user": <usuario>, "token": <token> } }`, de forma que la persona queda autenticada sin tener que iniciar sesión aparte.

#### Scenario: Registro correcto con nombre

- **WHEN** se envía `{ "fullName": "Ada Lovelace", "email": "ada@example.com", "password": "12345678", "passwordConfirmation": "12345678" }` y ese email no está registrado
- **THEN** la respuesta es `200` con `data.user` (con `fullName` `"Ada Lovelace"` y `email` `"ada@example.com"`) y `data.token` es una cadena no vacía que da acceso a las peticiones autenticadas

#### Scenario: Registro correcto sin nombre

- **WHEN** se envía un registro válido con `"fullName": null`
- **THEN** la respuesta es `200` y `data.user.fullName` es `null`

### Requirement: Validación del registro

La API SHALL rechazar el registro con `422` y un cuerpo `{ "errors": [ ... ] }` cuando los datos no son válidos, con una entrada por cada regla incumplida que incluye `message`, `rule`, `field` y, en las reglas de longitud, `meta` con el límite (`min` o `max`). Las reglas son:

- `fullName` MUST estar presente en el cuerpo, aunque sea con valor `null`.
- `email` MUST ser una dirección de email válida de como mucho 254 caracteres y MUST NOT coincidir con el de una cuenta existente.
- `password` y `passwordConfirmation` MUST tener entre 8 y 32 caracteres cada uno.
- `passwordConfirmation` MUST ser idéntico a `password`.

En caso de rechazo, la API MUST NOT crear la cuenta.

#### Scenario: Email ya registrado

- **WHEN** se intenta registrar un email que ya pertenece a una cuenta
- **THEN** la respuesta es `422` e incluye un error con `field` `"email"` y `rule` `"database.unique"`

#### Scenario: Contraseña demasiado corta

- **WHEN** se envía `password` con menos de 8 caracteres
- **THEN** la respuesta es `422` e incluye un error con `field` `"password"`, `rule` `"minLength"` y `meta.min` `8`

#### Scenario: Contraseña demasiado larga

- **WHEN** se envía `password` con más de 32 caracteres
- **THEN** la respuesta es `422` e incluye un error con `field` `"password"`, `rule` `"maxLength"` y `meta.max` `32`

#### Scenario: Contraseñas que no coinciden

- **WHEN** se envían `password` y `passwordConfirmation` válidos en longitud pero distintos
- **THEN** la respuesta es `422` e incluye un error con `field` `"passwordConfirmation"` y `rule` `"sameAs"`

#### Scenario: Email con formato inválido

- **WHEN** se envía `"email": "nope"`
- **THEN** la respuesta es `422` e incluye un error con `field` `"email"` y `rule` `"email"`

#### Scenario: Falta la clave del nombre

- **WHEN** se envía un registro sin la clave `fullName`
- **THEN** la respuesta es `422` e incluye un error con `field` `"fullName"` y `rule` `"required"`

#### Scenario: Varios errores a la vez

- **WHEN** se envía un registro que incumple varias reglas en campos distintos
- **THEN** la respuesta es `422` y `errors` contiene una entrada por cada regla incumplida

### Requirement: El email distingue mayúsculas y minúsculas

La API SHALL tratar el email tal como llega, sin normalizarlo: dos emails que solo difieren en mayúsculas y minúsculas SHALL considerarse cuentas distintas, tanto en el registro como en el inicio de sesión.

#### Scenario: Registro con el mismo email en otra capitalización

- **WHEN** existe una cuenta con `ada@example.com` y se registra `ADA@example.com`
- **THEN** la respuesta es `200` y se crea una segunda cuenta independiente

#### Scenario: Inicio de sesión con otra capitalización

- **WHEN** solo existe la cuenta `ada@example.com` y se inicia sesión con `ADA@example.com`
- **THEN** la respuesta es `400` de credenciales inválidas

### Requirement: Inicio de sesión por la API

La API SHALL aceptar `POST /api/v1/auth/login` con `email` y `password`. Si las credenciales corresponden a una cuenta, SHALL responder `200` con `{ "data": { "user": <usuario>, "token": <token> } }`. Cada inicio de sesión SHALL emitir un token nuevo, sin invalidar los emitidos antes.

#### Scenario: Credenciales correctas

- **WHEN** se envían el email y la contraseña de una cuenta existente
- **THEN** la respuesta es `200` con `data.user` de esa cuenta y un `data.token` nuevo

#### Scenario: Varias sesiones simultáneas

- **WHEN** la misma cuenta inicia sesión dos veces
- **THEN** se obtienen dos tokens distintos y ambos dan acceso a las peticiones autenticadas

### Requirement: Credenciales inválidas indistinguibles

Cuando el email no corresponde a ninguna cuenta o la contraseña no es la suya, la API SHALL responder `400` con `{ "errors": [ { "message": "Invalid user credentials" } ] }`, sin `field` ni `rule`, y con la misma respuesta en ambos casos, de modo que no revela si el email está registrado.

#### Scenario: Contraseña incorrecta

- **WHEN** se envía el email de una cuenta existente con una contraseña que no es la suya
- **THEN** la respuesta es `400` con el único error `"Invalid user credentials"`

#### Scenario: Email no registrado

- **WHEN** se envía un email que no pertenece a ninguna cuenta
- **THEN** la respuesta es `400` con el mismo único error `"Invalid user credentials"`

### Requirement: Validación del inicio de sesión

La API SHALL rechazar el inicio de sesión con `422` y `{ "errors": [ ... ] }` cuando falta `email` o `password` (una cadena vacía cuenta como ausente), o cuando `email` no es una dirección válida de como mucho 254 caracteres. La contraseña del inicio de sesión MUST NOT someterse a reglas de longitud.

#### Scenario: Cuerpo vacío

- **WHEN** se envía `{}`
- **THEN** la respuesta es `422` con un error `rule` `"required"` para `email` y otro para `password`

#### Scenario: Contraseña vacía

- **WHEN** se envía un email válido con `"password": ""`
- **THEN** la respuesta es `422` con un error `rule` `"required"` para `password`, no un `400` de credenciales

### Requirement: Peticiones autenticadas con token

Las rutas bajo `/api/v1/account/` SHALL exigir la cabecera `Authorization: Bearer <token>` con un token vigente emitido por el registro o el inicio de sesión. Sin token, con un token desconocido o con un token ya revocado, la API SHALL responder `401` con `{ "errors": [ { "message": "Unauthorized access" } ] }`.

#### Scenario: Sin token

- **WHEN** se pide `GET /api/v1/account/profile` sin cabecera `Authorization`
- **THEN** la respuesta es `401` con el error `"Unauthorized access"`

#### Scenario: Token desconocido

- **WHEN** se pide `GET /api/v1/account/profile` con un token que la API no emitió
- **THEN** la respuesta es `401` con el error `"Unauthorized access"`

### Requirement: Consulta del perfil

La API SHALL responder a `GET /api/v1/account/profile`, con un token vigente, `200` y `{ "data": <usuario> }` con los datos de la cuenta dueña del token.

#### Scenario: Perfil propio

- **WHEN** se pide el perfil con el token obtenido al iniciar sesión como `ada@example.com`
- **THEN** la respuesta es `200` y `data.email` es `"ada@example.com"`

### Requirement: Forma pública del usuario

Siempre que la API devuelva un usuario SHALL incluir exactamente `id` (número), `fullName` (texto o `null`), `email`, `createdAt`, `updatedAt` (fechas ISO 8601) e `initials`. La API MUST NOT devolver la contraseña ni ningún derivado de ella.

`initials` SHALL calcularse así, en mayúsculas:

- si `fullName` tiene al menos dos palabras separadas por un espacio, la primera letra de las dos primeras;
- si `fullName` es una sola palabra, sus dos primeras letras;
- si `fullName` es `null`, la primera letra de lo que hay antes de la `@` del email y la primera de lo que hay después.

#### Scenario: Iniciales con nombre y apellido

- **WHEN** se registra una cuenta con `fullName` `"Ada Lovelace"`
- **THEN** `initials` es `"AL"`

#### Scenario: Iniciales con una sola palabra

- **WHEN** se registra una cuenta con `fullName` `"Grace"`
- **THEN** `initials` es `"GR"`

#### Scenario: Iniciales sin nombre

- **WHEN** se registra una cuenta sin nombre con email `probe@x.io`
- **THEN** `initials` es `"PX"`

#### Scenario: Sin contraseña en la respuesta

- **WHEN** se registra una cuenta, se inicia sesión o se pide el perfil
- **THEN** el objeto usuario de la respuesta no contiene ninguna clave con la contraseña

### Requirement: Cierre de sesión por la API

La API SHALL aceptar `POST /api/v1/account/logout` con un token vigente, revocar solo ese token y responder `200` con `{ "message": "Logged out successfully" }` (sin envoltorio `data`). Los demás tokens de la misma cuenta SHALL seguir vigentes.

#### Scenario: Cierre de sesión correcto

- **WHEN** se hace logout con un token vigente
- **THEN** la respuesta es `200` con `"Logged out successfully"` y cualquier petición posterior con ese token recibe `401`

#### Scenario: Logout con un token ya revocado

- **WHEN** se repite el logout con el mismo token
- **THEN** la respuesta es `401` con el error `"Unauthorized access"`

#### Scenario: Otras sesiones no se cierran

- **WHEN** una cuenta tiene dos tokens vigentes y hace logout con uno
- **THEN** el otro token sigue dando acceso al perfil

### Requirement: Pantalla de registro

La aplicación web SHALL ofrecer en `/register` un formulario "Crea tu cuenta" con los campos "Nombre completo (opcional)", "Email", "Contraseña" (con la ayuda "Entre 8 y 32 caracteres.") y "Repite la contraseña", un botón "Crear cuenta" y un enlace "Inicia sesión" que lleva a `/login`. Al registrarse con éxito, la persona SHALL quedar con sesión iniciada y ver su perfil.

#### Scenario: Registro correcto desde la web

- **WHEN** una persona sin sesión rellena el formulario con datos válidos y pulsa "Crear cuenta"
- **THEN** pasa a ver su perfil con la sesión iniciada

#### Scenario: Nombre en blanco

- **WHEN** la persona deja "Nombre completo" vacío o solo con espacios y se registra
- **THEN** la cuenta se crea sin nombre y el perfil muestra "Sin nombre"

#### Scenario: Contraseñas distintas detectadas en el navegador

- **WHEN** "Contraseña" y "Repite la contraseña" no coinciden y se pulsa "Crear cuenta"
- **THEN** aparece "Las contraseñas no coinciden." bajo "Repite la contraseña" y no se envía nada al servidor

#### Scenario: Email ya registrado desde la web

- **WHEN** la persona intenta registrar un email que ya existe
- **THEN** aparece "Ese email ya está registrado. Inicia sesión en su lugar." bajo el campo "Email"

### Requirement: Pantalla de inicio de sesión

La aplicación web SHALL ofrecer en `/login` un formulario "Inicia sesión" con los campos "Email" y "Contraseña", un botón "Entrar" y un enlace "Crea una" que lleva a `/register`. Al iniciar sesión con éxito, la persona SHALL ver su perfil.

#### Scenario: Inicio de sesión correcto desde la web

- **WHEN** una persona sin sesión introduce credenciales correctas y pulsa "Entrar"
- **THEN** pasa a ver su perfil con la sesión iniciada

#### Scenario: Credenciales incorrectas desde la web

- **WHEN** la persona introduce un email o una contraseña incorrectos
- **THEN** aparece un aviso de error sobre el formulario con "El email o la contraseña no son correctos." y sigue en `/login`

### Requirement: Errores y estado de envío en los formularios de acceso

En las pantallas de registro e inicio de sesión, la aplicación web SHALL mostrar en castellano cada error de validación debajo de su campo (como mucho uno por campo) y los errores no asociados a un campo en un aviso sobre el formulario. Mientras el envío está en curso, el botón SHALL deshabilitarse y mostrar "Creando cuenta…" o "Entrando…". Los mensajes SHALL ser:

- email inválido: "Introduce una dirección de email válida.";
- campo vacío: "Falta rellenar el email." / "Falta rellenar la contraseña." (y equivalentes);
- longitud mínima o máxima: "la contraseña debe tener al menos 8 caracteres." / "la contraseña no puede superar los 32 caracteres." (y equivalentes);
- contraseñas distintas: "Las contraseñas no coinciden.";
- servidor inaccesible: "No se pudo conectar con el servidor. Comprueba que el backend está arrancado.";
- error inesperado del servidor: "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento.".

#### Scenario: Error de campo bajo su input

- **WHEN** la persona envía el registro con una contraseña de 5 caracteres
- **THEN** aparece "la contraseña debe tener al menos 8 caracteres." bajo "Contraseña", en lugar de la ayuda

#### Scenario: Servidor caído al enviar

- **WHEN** la persona envía un formulario de acceso y el servidor no responde
- **THEN** aparece sobre el formulario "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." y el botón vuelve a estar disponible

#### Scenario: Botón bloqueado durante el envío

- **WHEN** la persona pulsa "Entrar" y la respuesta aún no ha llegado
- **THEN** el botón muestra "Entrando…" y no puede pulsarse otra vez

### Requirement: Pantalla de perfil

La aplicación web SHALL mostrar en `/profile`, a una persona con sesión, sus iniciales en un círculo, su nombre completo (o "Sin nombre" si no tiene), su email, la fecha de alta como "Miembro desde" con fecha larga en castellano (por ejemplo "1 de octubre de 2026") y un botón "Cerrar sesión".

#### Scenario: Perfil con nombre

- **WHEN** Ada Lovelace (`ada@example.com`) abre su perfil
- **THEN** ve "AL", "Ada Lovelace", "ada@example.com" y "Miembro desde" con su fecha de alta

### Requirement: Cierre de sesión desde la web

Al pulsar "Cerrar sesión", la aplicación web SHALL deshabilitar el botón mostrando "Cerrando sesión…", cerrar la sesión en el navegador y llevar a la persona a `/login`, aunque el servidor no confirme la revocación del token.

#### Scenario: Cerrar sesión

- **WHEN** una persona con sesión pulsa "Cerrar sesión"
- **THEN** acaba en `/login` sin sesión y, al recargar la página, sigue sin sesión

#### Scenario: Cerrar sesión con el servidor caído

- **WHEN** una persona pulsa "Cerrar sesión" y el servidor no responde
- **THEN** igualmente acaba en `/login` sin sesión y sin ver ningún error

### Requirement: Sesión persistente en el navegador

La aplicación web SHALL conservar la sesión entre recargas y pestañas del mismo navegador. Al cargar la aplicación con una sesión guardada, SHALL mostrar un indicador de carga mientras la verifica contra el servidor, y:

- si el servidor la acepta, SHALL continuar con la sesión iniciada;
- si el servidor la rechaza, SHALL descartarla y llevar a `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión.";
- si no se puede verificar (servidor caído o error), SHALL llevar a `/login` con el aviso del error correspondiente, pero conservando la sesión guardada para que una recarga posterior la recupere cuando el servidor vuelva.

#### Scenario: Recargar con sesión válida

- **WHEN** una persona con sesión recarga `/profile`
- **THEN** ve un indicador de carga y después su perfil, sin pasar por `/login`

#### Scenario: Sesión revocada en el servidor

- **WHEN** se carga la aplicación con una sesión guardada cuyo token el servidor ya no reconoce
- **THEN** la persona ve `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión."

#### Scenario: Servidor caído al recargar

- **WHEN** se carga la aplicación con una sesión guardada y el servidor no responde
- **THEN** la persona ve `/login` con "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." y, al recargar con el servidor ya arrancado, vuelve a ver su perfil

#### Scenario: El aviso desaparece al volver a entrar

- **WHEN** la persona ve el aviso de sesión perdida en `/login` e inicia sesión con éxito
- **THEN** el aviso deja de mostrarse

### Requirement: Protección de pantallas según la sesión

La aplicación web SHALL permitir `/profile` solo con sesión iniciada y `/login` y `/register` solo sin sesión, y SHALL redirigir cualquier otra dirección a `/profile`. Mientras se verifica una sesión guardada, SHALL mostrar el indicador de carga en lugar de redirigir.

#### Scenario: Perfil sin sesión

- **WHEN** una persona sin sesión abre `/profile`
- **THEN** es llevada a `/login`

#### Scenario: Login con sesión

- **WHEN** una persona con sesión abre `/login` o `/register`
- **THEN** es llevada a `/profile`

#### Scenario: Dirección desconocida

- **WHEN** alguien abre una dirección que no existe, por ejemplo `/`
- **THEN** es llevado a `/profile`, y de ahí a `/login` si no tiene sesión
