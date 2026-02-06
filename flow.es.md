# Flujo de Ejecución de OpenCode

Este documento detalla el flujo de ejecución del sistema, desde el punto de entrada hasta el bucle principal del agente, para ayudarte a navegar por el código y saber dónde realizar modificaciones.

## Punto de Entrada (CLI)

El sistema comienza en `packages/opencode/src/index.ts`. Este archivo:
1. Configura el sistema de logs.
2. Define los comandos disponibles utilizando `yargs`.
3. Delega la ejecución a los manejadores en `packages/opencode/src/cli/cmd/`.

### Comandos Principales

- **`run` (`run.ts`)**: Es el comando principal para iniciar una tarea desde la terminal. Inicia una sesión, procesa el mensaje del usuario y entra en el bucle del agente.
- **`serve` (`serve.ts`)**: Inicia el servidor headless (API) en el puerto 4096 (por defecto).
- **`web` (`web.ts`)**: Inicia el servidor y abre la interfaz web.

## El Servidor (API)

El servidor está definido en `packages/opencode/src/server/server.ts` y utiliza **Hono**.
- Las rutas se dividen por responsabilidad en `packages/opencode/src/server/routes/` (ej. `session`, `project`, `file`).
- Utiliza `Instance.provide` para establecer el contexto del directorio de trabajo (CWD) para cada petición.

## El Bucle del Agente (Core Logic)

El flujo lógico cuando envías un mensaje (vía `run` o vía API) es el siguiente:

1. **`SessionPrompt.prompt` (`packages/opencode/src/session/prompt.ts`)**:
   - Crea el mensaje del usuario en la base de datos (SQLite/Storage).
   - Inicia el bucle principal llamando a `loop()`.

2. **`SessionPrompt.loop`**:
   - Recupera el historial de la sesión.
   - Determina si hay tareas pendientes (como una `subtask` o `compaction`).
   - Crea un nuevo mensaje de tipo `assistant`.
   - Resuelve las herramientas (tools) disponibles para el agente actual.
   - Invoca a `SessionProcessor.process()`.

3. **`SessionProcessor.process` (`packages/opencode/src/session/processor.ts`)**:
   - Llama a `LLM.stream()` para obtener la respuesta del modelo.
   - Itera sobre los eventos del stream (texto, inicio de herramienta, razonamiento).
   - **Ejecución de Herramientas**: Si el modelo pide una herramienta, se ejecuta en tiempo real y el resultado se envía de vuelta al modelo para continuar el flujo.
   - Gestiona el "doom loop" (bucles infinitos) y las peticiones de permisos.

## Dónde centrarse según lo que quieras modificar

| Objetivo | Ubicación clave |
| :--- | :--- |
| **Añadir una nueva herramienta** | `packages/opencode/src/tool/`. Usa `Tool.define`. |
| **Cambiar el comportamiento del agente** | `packages/opencode/src/agent/agent.ts` y el prompt en `packages/opencode/src/session/prompt.ts`. |
| **Modificar la interfaz de terminal (TUI)** | `packages/opencode/src/cli/cmd/tui/`. |
| **Añadir un nuevo endpoint a la API** | `packages/opencode/src/server/routes/`. No olvides correr `./script/generate.ts` después. |
| **Modificar la lógica de sesiones/historial** | `packages/opencode/src/session/`. |
| **Cambiar la UI Web** | `packages/app/`. |

## Resumen del Flujo de Datos

```
CLI/Web -> API (Hono) -> SessionPrompt -> SessionProcessor -> LLM
                                 ^             |
                                 |             v
                                 +------- Herramientas (Bash, Read, Edit...)
```

Este flujo recursivo y basado en eventos permite que el agente sea altamente interactivo y capaz de corregir su propio rumbo durante la ejecución.
