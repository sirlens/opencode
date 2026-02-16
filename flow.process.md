# Deep Dive: El Proceso de Ejecución de OpenCode

Este documento detalla técnicamente cómo OpenCode gestiona una solicitud, desde que entra por el CLI hasta la ejecución recursiva de subagentes, respaldado por referencias directas al código fuente.

## 1. Entrada y Orquestación Inicial
Todo comienza cuando se invoca una solicitud al agente. El punto de entrada lógico es la función `prompt` en `packages/opencode/src/session/prompt.ts`.

### Flujo de Inicialización:
1.  **`prompt(input)`**: Valida la entrada y crea el mensaje del usuario (`createUserMessage`).
2.  **`loop(sessionID)`**: Inicia el bucle principal de la sesión.

```typescript
// packages/opencode/src/session/prompt.ts

export const prompt = fn(PromptInput, async (input) => {
  const session = await Session.get(input.sessionID);
  const message = await createUserMessage(input); // Crea el mensaje en DB
  // ...
  return loop(input.sessionID); // Entra al bucle recursivo
});
```

## 2. El Bucle de Razonamiento (`loop`)
La función `loop` es el motor de OpenCode. Se encarga de:
1.  **Cargar el historial**: `MessageV2.filterCompacted(MessageV2.stream(sessionID))`.
2.  **Identificar Tareas**: Revisa si el último mensaje tiene partes de tipo `subtask` o `compaction`.
3.  **Procesar**: Llama al `SessionProcessor` para interactuar con el LLM.

## 3. Delegación Dinámica via `TaskTool`
Cuando el Agente Primario (ej. `build` usando un modelo de alta capacidad como "BigPickle") decide que la tarea requiere un subagente, invoca la herramienta `task`.

### Implementación de la Delegación:
Ubicación: `packages/opencode/src/tool/task.ts`

El código realiza los siguientes pasos críticos:
1.  **Crea una sesión hija**: Utiliza el `parentID` para vincularla a la actual.
2.  **Llama recursivamente a `SessionPrompt.prompt`**: Pero esta vez en la nueva sesión y con un agente especializado.

```typescript
// packages/opencode/src/tool/task.ts

export const TaskTool = Tool.define("task", async (ctx) => {
  // ...
  async execute(params, ctx) {
    const session = await Session.create({
      parentID: ctx.sessionID, // Vínculo jerárquico
      title: params.description + ` (@${agent.name} subagent)`,
      // ...
    });

    const result = await SessionPrompt.prompt({
      messageID,
      sessionID: session.id, // Nueva sesión atómica
      agent: agent.name,     // Agente especializado (ej. explore)
      // ...
    });
    // El resultado del subagente se devuelve como output de la herramienta
    return { output: result.parts.findLast(x => x.type === "text")?.text };
  }
});
```

## 4. Procesamiento de la Respuesta y Herramientas (`SessionProcessor`)
Ubicación: `packages/opencode/src/session/processor.ts`

Mientras el LLM genera tokens, el `SessionProcessor` escucha el stream y actúa según el tipo de evento:
-   **`tool-call`**: Activa la ejecución de la herramienta localmente.
-   **`tool-result`**: Captura el output y lo añade al contexto para el siguiente paso del modelo.
-   **`step-finish`**: Registra el uso de tokens y el coste.

```typescript
// packages/opencode/src/session/processor.ts

case "tool-call": {
  const match = toolcalls[value.toolCallId];
  if (match) {
    const part = await Session.updatePart({
      // ... actualiza estado a 'running'
    });
    // ... lógica para detectar bucles infinitos (doom loop)
  }
  break;
}
```

## 5. Gestión del Contexto y Almacenamiento
La jerarquía de sesiones se gestiona en `packages/opencode/src/session/index.ts`. Cada sesión es un objeto en el `Storage` (SQLite) que contiene:
-   `id`: Identificador único.
-   `parentID`: ID de la sesión que la originó (si existe).
-   `permission`: Reglas específicas que heredan o restringen las capacidades del subagente.

## Resumen del Flujo Técnico Completo

1.  **`prompt.ts`**: Recibe "Refactoriza X". Crea mensaje. Llama a `loop`.
2.  **`processor.ts`**: Envía historial al LLM. El LLM responde: "Necesito explorar el código primero". Llama a herramienta `task` con subagente `explore`.
3.  **`task.ts`**: Crea sesión `B` (hija de `A`). Llama a `prompt.ts` para la sesión `B`.
4.  **`prompt.ts` (Sesión B)**: El subagente `explore` realiza búsquedas (grep, ls) y devuelve un resumen.
5.  **`task.ts`**: Recibe el resumen y lo entrega a la sesión `A`.
6.  **`processor.ts` (Sesión A)**: El LLM original recibe el resumen y ahora tiene contexto para realizar el cambio real usando la herramienta `edit`.

Esta arquitectura permite que OpenCode sea **recursivo y modular**, manteniendo cada subtarea aislada y permitiendo que agentes especializados realicen el trabajo pesado de forma atómica.

## Inmersión en A2A (Agent-to-Agent) y Orquestación Nativa

Para entender cómo OpenCode logra que un agente hable con otro sin usar librerías externas como Google ADK, debemos observar tres capas: la **Inyección**, la **Instanciación** y la **Ejecución Recursiva**.

### 1. La "Inyección" (Registro de Herramientas)
Ubicación: `packages/opencode/src/tool/registry.ts`

El sistema no utiliza un "inyector de dependencias" clásico de Java/C#, sino un **Registro de Herramientas**. En la función `all()`, se "inyectan" las herramientas básicas que todos los agentes pueden ver, incluida `TaskTool`.

```typescript
// packages/opencode/src/tool/registry.ts
async function all(): Promise<Tool.Info[]> {
  return [
    // ... otras herramientas
    TaskTool, // Aquí se "inyecta" la capacidad de A2A
    // ...
  ]
}
```

### 2. La Instanciación del Contexto
Ubicación: `packages/opencode/src/session/prompt.ts` -> `resolveTools`

Antes de que el LLM reciba el mensaje, el sistema prepara su "caja de herramientas". Aquí es donde se le da al agente el **Contexto de Ejecución**, que incluye el `sessionID` actual y el `abortSignal`.

```typescript
// packages/opencode/src/session/prompt.ts
async function resolveTools(input: { ... }) {
  const context = (args, options): Tool.Context => ({
    sessionID: input.session.id,
    // ...
  });
  // ... transforma las herramientas del registro en funciones usables por el LLM
}
```

### 3. El Flujo de Uso A2A (Paso a Paso)

El flujo exacto de una comunicación entre agentes es:

1.  **Activación**: El LLM (Agente A) decide que no puede resolver algo solo y emite un `tool-call` de la herramienta `task`.
2.  **Captura**: El `SessionProcessor` (`processor.ts`) detecta la llamada y ejecuta `TaskTool.execute`.
3.  **Bifurcación**: `TaskTool` crea una **sesión hija** atómica en la base de datos.
4.  **Recursión (A2A)**: `TaskTool` llama de nuevo a `SessionPrompt.prompt` pero pasando el ID de la **sesión hija** y el tipo del **Agente B** (ej. `explore`).
5.  **Aislamiento**: El Agente B resuelve su tarea en su propia burbuja de contexto (sesión hija).
6.  **Retorno**: El Agente B termina, y `TaskTool` captura su última respuesta de texto, devolviéndola como el resultado de la herramienta al Agente A.

### ¿Por qué es "Nativo"?
Es nativo porque **OpenCode reutiliza su propio punto de entrada (`prompt`) para ejecutar subagentes**. No hay una capa intermedia de orquestación ni grafos externos; es simplemente una función llamándose a sí misma (recursividad) pero con diferentes parámetros de sesión y agente.

### Resumen de Protocolos
- **Comunicación Interna (A2A)**: Basada en el sistema de sesiones de OpenCode.
- **Comunicación Externa (Editor/IDE)**: Usa **ACP (Agent Client Protocol)**, definido en `packages/opencode/src/acp/`.
- **Abstracción de Modelos**: Usa **Vercel AI SDK**, permitiendo que cualquier modelo pueda realizar estas llamadas a herramientas.
