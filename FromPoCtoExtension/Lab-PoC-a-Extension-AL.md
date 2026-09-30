# Lab: de PoC (no-code) a extensión AL — Revisor de Pedidos de Venta Abiertos

Este laboratorio convierte el agente diseñado sin código en el Lab 2 (Revisor de Pedidos de Venta Abiertos) en una extensión AL lista para producción, usando la plantilla oficial **Agent** de VS Code.

Partimos de un agente ya probado en sandbox con instrucciones estables. El objetivo no es reescribir nada desde cero: es tomar lo que exportamos y convertirlo en código versionado, publicable y evaluable.

---

## 1. Descarga el XML del agente diseñado

1. Abre el agente **Revisor de Pedidos de Venta Abiertos** en el diseñador (Agent Card).
2. Usa la acción de exportación del agente para descargar su definición.
3. Guarda el fichero (`REVISOR DE PEDIDOS DE VENTA ABIERTOS-agent.xml`) en un sitio que luego puedas referenciar desde el proyecto AL — por ejemplo una carpeta `xml/` dentro del propio proyecto.

Este XML contiene: `Name`, `DisplayName`, `Initials`, `Profile`, `AccessControls`, `UserSettings` y las `Instructions` completas (Responsibilities/Guidelines/Instructions). Es la única fuente de verdad de lo que el agente hace hoy — todo lo que sigue es transportar este contenido a AL, no reinventarlo.

---

## 2. Crea el proyecto desde la plantilla Agent

1. En VS Code, abre la paleta de comandos: **Ctrl+Shift+P**.
2. Ejecuta **AL: New Project** (no `AL: Go!` — ese comando solo crea un `HelloWorld.al` básico, sin selector de plantilla).
3. Elige la plantilla **Agent**.
4. Define el rango de IDs de tu app (idRange) — tiene que ser el rango que tengas reservado para este proyecto en tu licencia/sandbox.
5. Elige una carpeta vacía como destino y el servidor sandbox donde vas a publicar.

El asistente crea el `app.json`, el `launch.json` y la estructura de carpetas (`Setup/`, `Integration/`, `Example/`) con los objetos base ya escritos, pero todavía sin nada de tu agente concreto.

---

## 3. Descarga los símbolos

Con el proyecto abierto: **Ctrl+Shift+P → AL: Download symbols**. Esto trae del sandbox las definiciones de todos los objetos de plataforma y de las apps base que tu proyecto referencia (`System.Agents`, `System.AI`, `Microsoft.Sales.Document`, etc.). Sin este paso, el proyecto no compila — el editor no reconoce tipos como `Codeunit "Agent Task Builder"` o `Record "Agent Task"`.

---

## 4. Los ficheros que hay que tocar

Esto es lo que realmente cambia entre el "Hello World" de la plantilla y un agente que hace lo que dice el XML.

### 4.1 El núcleo del agente (identidad y ejecución)

**`Setup/Metadata/MyAgentFactory.Codeunit.al`** — codeunit que implementa `IAgentFactory`.
Qué es: crea la instancia del agente — initials, página de primer setup, perfil por defecto, permisos por defecto.
Qué tocar: nada directamente aquí; delega en `MyAgentSetup.Codeunit.al` (ver más abajo), que es donde van los datos concretos del Revisor de Pedidos.

**`Setup/Metadata/MyAgentMetadata.Codeunit.al`** — codeunit que implementa `IAgentMetadata`.
Qué es: identidad en tiempo de ejecución — initials, página de setup, página de resumen (KPI), página de mensajes de tarea.
Qué tocar: nada directamente; también delega en `MyAgentSetup.Codeunit.al`.

**`Setup/TaskExecution/MyAgentTaskExecution.Codeunit.al`** — codeunit que implementa `IAgentTaskExecution`.
Qué es: los "hooks" de ejecución — validar mensajes de entrada, sugerir intervención, dar contexto de página, posprocesar el mensaje de salida.
Qué tocar: para este caso simple, **nada obligatorio**. La lógica real del Revisor de Pedidos vive en las Instructions (texto), no en AL. Deja los métodos con su comportamiento por defecto (`exit(true)`, sin anotaciones) salvo que quieras añadir validaciones propias más adelante.

**`Setup/Metadata/MyAgentMetadataProvider.EnumExt.al`** — enumextension sobre `Agent Metadata Provider`.
Qué es: el "enchufe" que conecta las tres codeunits anteriores con la plataforma:
```al
value(50901; "My Agent")
{
    Caption = 'My Agent'; // TODO
    Implementation = IAgentFactory = MyAgentFactory, IAgentMetadata = MyAgentMetadata, IAgentTaskExecution = MyAgentTaskExecution;
}
```
Qué tocar: renombra el valor del enum y el `Caption` a algo del dominio, por ejemplo `"Sales Order Review Agent"` con caption "Revisor de Pedidos de Venta Abiertos".

**`Setup/MyAgentSetup.Codeunit.al`** — codeunit interna con los datos concretos del agente.
Qué es: aquí están los labels que identifican al agente (nombre, display name, initials, resumen) y la carga de las Instructions vía `NavApp.GetResourceAsText`.
Qué tocar — **este es el fichero con más cambios reales**:
```al
AgentInitialsLbl: Label 'RDPD';
AgentNameLbl: Label 'REVISOR DE PEDIDOS DE VENTA ABIERTOS';
DefaultDisplayNameLbl: Label 'Revisor de Pedidos de Venta Abiertos';
AgentSummaryLbl: Label 'Revisa los pedidos de venta abiertos...';
DefaultProfileTok: Label 'SALES ORDER AGENT';
DefaultPermissionSetTok: Label 'MY AGENT'; // ver punto 4.3
```
Copia estos valores directamente del XML exportado — no hay que inventar nada. Ya vienen bien en el proyecto generado, revisa solo que coincidan.

### 4.2 Las instrucciones

**`.resources/Instructions/InstructionsV1.txt`**
Qué es: el texto completo de Responsibilities/Guidelines/Instructions.
Qué tocar: pega aquí el bloque `<Instructions>` del XML tal cual, sin resumir ni reformular. Es el mismo patrón que en el ejercicio AL-1 del taller: instrucciones versionadas en Git, no solo en el diseñador.

### 4.3 Permisos y capacidad

**`Setup/Permissions/MyAgent.permissionset.al`**
Qué es: el permission set que se asigna al agente. La plantilla arranca con `IncludedPermissionSets = "D365 BASIC"` — **no** hereda el `RoleID="SUPER"` que traía el XML del diseñador no-code, pero tampoco es el mínimo estricto.
Qué tocar: acota a lectura de Sales Header (no posteado) y Customer. Nada de posting, nada de administración. Es la deuda que dejamos anotada al hablar del export no-code — aquí es donde se paga.

**`Integration/MyAgentCopilotCapability.EnumExt.al`**
Qué es: registra la capacidad Copilot que verás en "Copilot & agent capabilities".
Qué tocar: renombra `"My Agent Capability"` y su `Caption` a algo reconocible, por ejemplo `"Sales Order Review Capability"`.

**`Integration/MyAgentInstall.Codeunit.al`**
Qué es: al instalar la app, registra la capacidad y aplica instrucciones + control de acceso a cada instancia de setup ya creada.
Qué tocar: normalmente nada — solo revisa el `LearnMoreUrlTxt` (hoy es un placeholder `'link-to-my-documentation'`).

**`Integration/MyAgentUpgrade.Codeunit.al`**
Qué es: para cuando saques una v2 y quieras propagar cambios de instrucciones a instancias existentes.
Qué tocar: nada para este v1 — está todo comentado a propósito.

### 4.4 Perfil y pantallas

**`Setup/Profile/Prof.SalesOrderAgent.al`**
Qué es: el perfil (`"Sales Order Agent"`) que ve el agente al arrancar — Role Center + customizaciones.
Qué tocar: el `Caption` y `Description` ya vienen bien (copiados del XML). Revisa que `RoleCenter` y `Customizations` apunten a los ficheros correctos (ver abajo).

**`Setup/Profile/MyAgentRoleCenter.Page.al`**
Qué es: el Role Center vacío del agente.
Qué tocar: para este caso simple puede quedarse vacío — el agente no navega interactivamente, ejecuta la tarea y termina. Si más adelante quieres que el agente tenga accesos directos propios, aquí es donde se añaden.

**`Setup/Profile/MyAgentCustomerList.PageCustomization.al`**
Qué es: personalización de la Customer List para el perfil del agente (hoy vacía, todo comentado).
Qué tocar: opcional. Si quieres limitar qué ve el agente al consultar el cliente bloqueado, aquí se recorta la vista. Para el caso simple, dejarla vacía es válido — el agente ve la lista estándar.

### 4.5 Configuración y resumen (KPI)

**`Setup/MyAgentSetup.Table.al`** y **`Setup/MyAgentSetup.Page.al`**
Qué es: tabla y página de configuración propia del agente, más allá de lo que ya trae la plataforma (Agent Setup Part).
Qué tocar: el campo placeholder `Custom Property` no aplica a este caso — puedes dejarlo vacío/sin usar o quitarlo si prefieres un proyecto limpio. No es obligatorio rellenarlo.

**`Setup/KPI/MyAgentKPI.Table.al`** y **`Setup/KPI/MyAgentKPI.Page.al`**
Qué es: la página de resumen del agente (lo que se ve como tarjeta de actividad).
Qué tocar: renombra el campo `Custom KPI` a algo real, por ejemplo `"Exceptions Found"`, y complétalo desde `MyAgentTaskExecution` cuando el agente reporte el total de excepciones encontradas. No es obligatorio para que el agente funcione, pero sí para que el KPI tenga sentido en producción.

### 4.6 El disparador — ya viene hecho

**`Example/Pag-Ext50902.MyAgentSalesOrderListExt.al`**
Qué es: pageextension sobre **Sales Order List** con la acción "Review Open Orders with Agent", que lanza la tarea con `Agent Task Builder` sobre el listado completo de pedidos abiertos.
Qué tocar: nada — coincide exactamente con el diseño del taller (acción sobre la lista, no sobre un pedido concreto, porque las Instructions dicen "Open the list of open sales orders"). Revisa solo el wording del `Caption`/`ToolTip` si quieres afinarlo.

> Nota de carpeta: este fichero vive en `Example/` porque la plantilla lo trata como "un ejemplo de disparador", pero en nuestro caso **es el disparador real** — no lo borres junto con el resto de `Example/` (ver punto 5).

---

## 5. Los ficheros que no valen para este proyecto

**`Example/MyAgentCustomerCardExt.PageExt.al`**
Qué es: pageextension sobre **Customer Card** con una acción "Assign Agent Task" que lanza una tarea con un mensaje genérico ("Please process the customer with the name %1 and no. %2").
Por qué no vale: nuestro agente no se lanza desde una ficha de cliente — se lanza desde la lista de pedidos abiertos (punto 4.6). Este fichero es un ejemplo genérico de la plantilla, no nuestro caso de uso.
Para qué podría servir: si en el futuro quieres poder lanzar el agente manualmente sobre un cliente concreto (por ejemplo, para revisar solo sus pedidos), este es el patrón de partida.
**Acción: bórralo.**

**`Example/MyAgentPublicAPI.Codeunit.al`**
Qué es: codeunit pública con procedures `AssignTask`, `IsActive`, `Deactivate` — una API para que otras apps o triggers lancen tareas al agente sin conocer los detalles internos.
Por qué no vale: solo lo usa la acción de la Customer Card que acabamos de descartar. Nuestro disparador real (Sales Order List) ya llama a `Agent Task Builder` directamente.
Para qué podría servir: si más adelante quieres lanzar el agente desde un **business event** o una **Job Queue** (en vez de una acción de página), esta API es exactamente el punto de entrada que necesitarías — evita duplicar la lógica de `AgentTaskBuilder.Initialize(...).AddTaskMessage(...).Create()` en varios sitios.
**Acción: bórralo (de momento).**

**`Example/MyAgentPublicAPIImpl.Codeunit.al`**
Qué es: la implementación interna de la API pública anterior.
Por qué no vale: mismo motivo — sin la API pública ni la acción de Customer Card, esta implementación queda sin ningún llamador.
**Acción: bórralo junto con el anterior.**

---

## 6. Compilar, publicar y configurar

### 6.1 Compilar

1. Revisa que el `idRange` del `app.json` coincide con el rango reservado de tu proyecto — todos los objetos generados (50900–50907 en este caso) tienen que caer dentro.
2. **Ctrl+Shift+B** (o F5 para compilar + publicar directamente) para verificar que el proyecto compila sin errores tras los cambios de los puntos 4 y 5.
3. Errores típicos en esta fase: referencias rotas a los ficheros de `Example/` que acabas de borrar (revisa que ningún otro fichero los siga usando — en este proyecto no deberían quedar referencias), o nombres de objeto que superan el límite de longitud de AL (usa las initials, `RDPD`, como abreviación si hace falta).

### 6.2 Publicar en el sandbox

**F5** publica y despliega la app en el sandbox configurado en `launch.json`. La primera vez, BC instala la app: se ejecuta `MyAgentInstall.Codeunit.al`, que registra la capacidad Copilot.

### 6.3 Qué configurar en el Setup

1. Ve a **Copilot & agent capabilities** y comprueba que tu capacidad (renombrada en el punto 4.3) aparece y está disponible.
2. Activa la capacidad si no lo está ya.
3. Desde el lanzador de agentes (el **+** del role center), crea una instancia del nuevo agente AL. Se abre automáticamente `MyAgentSetup.Page.al` (el Agent Setup Part estándar): revisa nombre, display name y control de acceso.
4. Confirma que el permission set asignado es el que acotaste en el punto 4.3 (no `SUPER`, no `D365 BASIC` sin revisar).
5. Marca la instancia como **Active**.
6. Prueba desde **Sales Order List → Review Open Orders with Agent** (punto 4.6) y sigue la tarea en el task pane / Agent Task Log.

---

## 7. ¿Se puede cargar como extensión en el mismo sitio que ya tenemos la PoC?

**Sí.** No hay ningún impedimento técnico para publicar esta app AL en el mismo entorno sandbox donde ya vive el agente diseñado sin código — de hecho, es el flujo recomendado por el libro: no se migra de un entorno a otro, se añade la versión AL al lado de la versión no-code para poder compararlas.

Dos cosas a tener en cuenta:

- **Convive, no sustituye automáticamente.** El agente diseñado y el agente AL son dos capacidades Copilot distintas (`Agent Design Experience` vs. tu nueva capacidad AL) y dos instancias de agente distintas. Verás ambos en el lanzador de agentes hasta que decidas qué hacer con el no-code.
- **Desactiva el diseñado para no confundirte.** Una vez que el agente AL está activo y probado, desactiva la instancia no-code (o al menos no la dejes activa a la vez que pruebas la AL) — así evitas lanzar sin querer el agente equivocado, y si tienes una suite de evaluación, apúntala a la capacidad AL en vez de a `Agent Design Experience` antes de volver a correrla.

No hace falta un sandbox nuevo ni limpiar nada del anterior — es el mismo tenant, la misma sandbox, dos capacidades conviviendo mientras decides cuál pasa a producción.
