# Cómo Crear Agentes y Subagentes Óptimamente en OpenCode

OpenCode está diseñado para ser extendido. Gracias a su arquitectura basada en sesiones jerárquicas y delegación recursiva, puedes crear estructuras de agentes muy potentes sin romper el funcionamiento interno.

## Estrategias para Crear Agentes

Existen tres formas principales de definir agentes en OpenCode:

### 1. Configuración Global (`~/.opencode/opencode.json`)
Ideal para agentes que quieres tener disponibles en todos tus proyectos (ej: un "Traductor" o un "Revisor de Seguridad").

### 2. Configuración de Proyecto (`opencode.json`)
Ideal para agentes específicos de una tecnología usada en el repositorio (ej: un "Experto en SQL").

### 3. Directorio de Agentes (`.opencode/agents/`) **(ESTRATEGIA ÓPTIMA)**
Esta es la mejor estrategia. Te permite crear archivos `.md` donde el nombre del archivo es el nombre del agente. El contenido del archivo es el prompt del sistema, y el frontmatter (YAML al inicio) define la configuración.

---

## La Estrategia Más Óptima: "Expertos Atómicos"

La mejor forma de trabajar con la jerarquía de OpenCode no es crear un único agente gigante, sino un **Agente Primario Orquestador** y varios **Subagentes Especialistas**.

### Ventajas:
- **Ahorro de Tokens**: Los subagentes solo reciben el contexto necesario para su tarea.
- **Precisión**: Un prompt especializado en una sola tarea (ej: CSS) es más efectivo que uno general.
- **Modularidad**: Puedes actualizar la lógica de un especialista sin afectar al resto.

---

## Ejemplo Práctico: Desarrollo con SAPUI5

Imagina que estás trabajando en una aplicación SAPUI5 compleja. Vamos a configurar una jerarquía de expertos.

### Paso 1: Definir el Orquestador
Crea `.opencode/agents/ui5-lead.md`:

```markdown
---
description: Orquestador experto en SAPUI5. Coordina tareas de UI y lógica.
mode: primary
model: anthropic/claude-3-5-sonnet
---
Eres el arquitecto líder de SAPUI5. Tu trabajo es recibir solicitudes del usuario y delegar el trabajo detallado a tus especialistas.

- Para cambios en vistas (XML), usa al especialista `ui5-xml`.
- Para lógica de negocio o controladores, usa al especialista `ui5-logic`.
- Siempre revisa que se sigan las guías de SAP Fiori.
```

### Paso 2: Definir el Especialista en Vistas
Crea `.opencode/agents/ui5-xml.md`:

```markdown
---
description: Experto en vistas XML y Fragmentos de SAPUI5.
mode: subagent
---
Eres un experto en el lenguaje declarativo XML de SAPUI5.
- Conoces todos los controles de `sap.m` y `sap.ui.table`.
- Tu objetivo es generar código XML limpio, accesible y siguiendo los estándares de SAP.
- No escribas JavaScript, solo XML.
```

### Paso 3: Definir el Especialista en Lógica
Crea `.opencode/agents/ui5-logic.md`:

```markdown
---
description: Experto en Controllers y Modelos (OData/JSON) de SAPUI5.
mode: subagent
---
Eres un experto en JavaScript para SAPUI5.
- Te centras en el ciclo de vida del controlador (onInit, onExit).
- Eres experto en la gestión de `ODataModel` v2/v4.
- Sabes cómo realizar binding de datos de forma eficiente.
```

### Cómo funciona en la práctica:
1. El usuario dice: `opencode run "Añade una tabla de productos a la vista principal y carga los datos desde el servicio OData" @ui5-lead`.
2. `ui5-lead` analiza la tarea.
3. `ui5-lead` llama a la herramienta `task` para el subagente `ui5-xml` pidiéndole la tabla.
4. `ui5-lead` llama a la herramienta `task` para el subagente `ui5-logic` pidiéndole el binding y la gestión del modelo.
5. `ui5-lead` integra ambos resultados y entrega la solución final.

## Compatibilidad con Sesiones
Este enfoque es **totalmente compatible** y aprovecha el funcionamiento nativo de OpenCode:
- Cada llamada a `task` crea una **sesión hija**.
- El historial de la sesión hija es atómico y no ensucia la sesión principal.
- Los permisos pueden restringirse (ej: el especialista XML solo tiene permiso para editar archivos `.view.xml`).
