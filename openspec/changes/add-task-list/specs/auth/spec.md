# Spec Delta

## MODIFIED Requirements

### Requirement: Pantalla de registro

La aplicación web SHALL ofrecer en `/register` un formulario "Crea tu cuenta" con:

- los campos "Nombre completo (opcional)", "Email", "Contraseña" (con la ayuda "Entre 8 y 32 caracteres.") y "Repite la contraseña";
- un botón "Crear cuenta";
- un enlace "Inicia sesión" que lleva a `/login`.

Al registrarse con éxito, la persona SHALL quedar con sesión iniciada y ver la lista de tareas del equipo en `/tasks`.

#### Scenario: Registro correcto desde la web

- **WHEN** una persona sin sesión rellena el formulario con datos válidos y pulsa "Crear cuenta"
- **THEN** pasa a ver la lista de tareas en `/tasks` con la sesión iniciada

#### Scenario: Nombre en blanco

- **WHEN** la persona deja "Nombre completo" vacío o solo con espacios y se registra
- **THEN** la cuenta se crea sin nombre y el perfil muestra "Sin nombre"

#### Scenario: Nombre con espacios alrededor

- **WHEN** la persona escribe " Ada Lovelace " en "Nombre completo" y se registra
- **THEN** el perfil muestra "Ada Lovelace", sin los espacios de los extremos

#### Scenario: Contraseñas distintas detectadas en el navegador

- **WHEN** "Contraseña" y "Repite la contraseña" no coinciden y se pulsa "Crear cuenta"
- **THEN** aparece "Las contraseñas no coinciden." bajo "Repite la contraseña" y no se envía nada al servidor

#### Scenario: Email ya registrado desde la web

- **WHEN** la persona intenta registrar un email que ya existe
- **THEN** aparece "Ese email ya está registrado. Inicia sesión en su lugar." bajo el campo "Email"

### Requirement: Pantalla de inicio de sesión

La aplicación web SHALL ofrecer en `/login` un formulario "Inicia sesión" con:

- los campos "Email" y "Contraseña";
- un botón "Entrar";
- un enlace "Crea una" que lleva a `/register`.

Al iniciar sesión con éxito, la persona SHALL ver la lista de tareas del equipo en `/tasks`.

#### Scenario: Inicio de sesión correcto desde la web

- **WHEN** una persona sin sesión introduce credenciales correctas y pulsa "Entrar"
- **THEN** pasa a ver la lista de tareas en `/tasks` con la sesión iniciada

#### Scenario: Credenciales incorrectas desde la web

- **WHEN** la persona introduce un email o una contraseña incorrectos
- **THEN** aparece un aviso de error sobre el formulario con "El email o la contraseña no son correctos." y sigue en `/login`

### Requirement: Pantalla de perfil

La aplicación web SHALL mostrar en `/profile`, a una persona con sesión:

- sus iniciales en un círculo;
- su nombre completo, o "Sin nombre" si no tiene;
- su email;
- la fecha de alta como "Miembro desde", con fecha larga en castellano (por ejemplo "1 de octubre de 2026");
- un botón "Cerrar sesión";
- un enlace "Ver tareas" que lleva a la lista de tareas en `/tasks`.

#### Scenario: Perfil con nombre

- **WHEN** Ada Lovelace (`ada@example.com`) abre su perfil
- **THEN** ve "AL", "Ada Lovelace", "ada@example.com" y "Miembro desde" con su fecha de alta

#### Scenario: Del perfil a la lista

- **WHEN** la persona pulsa "Ver tareas" en su perfil
- **THEN** ve la lista de tareas en `/tasks`

### Requirement: Protección de pantallas según la sesión

La aplicación web SHALL permitir:

- `/profile` y `/tasks` solo con sesión iniciada;
- `/login` y `/register` solo sin sesión.

Cualquier otra dirección SHALL redirigirse a `/tasks`. Mientras se verifica una sesión guardada, SHALL mostrarse el indicador de carga en lugar de redirigir.

#### Scenario: Perfil sin sesión

- **WHEN** una persona sin sesión abre `/profile`
- **THEN** es llevada a `/login`

#### Scenario: Login con sesión

- **WHEN** una persona con sesión abre `/login` o `/register`
- **THEN** es llevada a `/tasks`

#### Scenario: Dirección desconocida

- **WHEN** alguien abre una dirección que no existe, por ejemplo `/`
- **THEN** es llevado a `/tasks`, y de ahí a `/login` si no tiene sesión

### Requirement: Sesión persistente en el navegador

La aplicación web SHALL conservar la sesión entre recargas y en las pestañas que se abran después en el mismo navegador. Una pestaña ya abierta MUST NOT enterarse de que se ha iniciado o cerrado sesión en otra hasta que se recargue. Al cargar la aplicación con una sesión guardada, SHALL mostrar un indicador de carga mientras la verifica contra el servidor, y:

- si el servidor la acepta, SHALL continuar con la sesión iniciada;
- si el servidor la rechaza, SHALL descartarla y llevar a `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión.";
- si no se puede verificar (servidor caído o error), SHALL llevar a `/login` con el aviso del error correspondiente, pero conservando la sesión guardada para que una recarga posterior la recupere cuando el servidor vuelva.

El aviso de sesión perdida SHALL mostrarse solo en `/login`, no en `/register`, y SHALL sustituirse por el error del intento actual si la persona intenta entrar y falla.

#### Scenario: Recargar con sesión válida

- **WHEN** una persona con sesión recarga `/profile`
- **THEN** ve un indicador de carga y después su perfil, sin pasar por `/login`

#### Scenario: Sesión revocada en el servidor

- **WHEN** se carga la aplicación con una sesión guardada cuyo token el servidor ya no reconoce
- **THEN** la persona ve `/login` con el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión."

#### Scenario: Servidor caído al recargar

- **WHEN** se carga la aplicación con una sesión guardada y el servidor no responde
- **THEN** la persona ve `/login` con "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." y, al recargar con el servidor ya arrancado, vuelve a ver la lista de tareas en `/tasks`

#### Scenario: El aviso desaparece al volver a entrar

- **WHEN** la persona ve el aviso de sesión perdida en `/login` e inicia sesión con éxito
- **THEN** el aviso deja de mostrarse
