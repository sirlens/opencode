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
    // 1. Creación de la sesión jerárquica
    const session = await Session.create({
      parentID: ctx.sessionID,
      title: params.description + ` (@${agent.name} subagent)`,
      // ...
    });

    // 2. Llamada recursiva al punto de entrada de OpenCode (A2A)
    const result = await SessionPrompt.prompt({
      messageID,
      sessionID: session.id,
      agent: agent.name,
      parts: promptParts,
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

```typescript
// packages/opencode/src/session/processor.ts

case "tool-call": {
  const match = toolcalls[value.toolCallId];
  if (match) {
    const part = await Session.updatePart({
      // ... actualiza estado a 'running'
    });
  }
  break;
}
```

## 5. Gestión del Contexto y Almacenamiento
La jerarquía de sesiones se gestiona en `packages/opencode/src/session/index.ts`. Cada sesión es un objeto en el `Storage` (SQLite) que contiene:
-   `id`: Identificador único.
-   `parentID`: ID de la sesión que la originó (si existe).

```typescript
// packages/opencode/src/session/index.ts

export async function createNext(input: { ... }) {
  const result: Info = {
    id: Identifier.descending("session", input.id),
    parentID: input.parentID, // <--- Aquí se guarda el vínculo A2A
    // ...
  }
  await Storage.write(["session", Instance.project.id, result.id], result)
  return result
}
```

## Inmersión en A2A (Agent-to-Agent) y Orquestación Nativa

Para entender cómo OpenCode logra que un agente hable con otro sin usar librerías externas como Google ADK, debemos observar tres capas: la **Inyección**, la **Instanciación** y la **Ejecución Recursiva**.

### 1. La "Inyección" (Registro de Herramientas)
Ubicación: `packages/opencode/src/tool/registry.ts`

OpenCode "inyecta" la capacidad A2A al incluir `TaskTool` en el registro global de herramientas que el LLM puede invocar.

```typescript
// packages/opencode/src/tool/registry.ts

async function all(): Promise<Tool.Info[]> {
  return [
    // ...
    TaskTool, // Inyecta la capacidad de delegar a otros agentes
    // ...
  ]
}
```

### 2. La Instanciación del Contexto
Ubicación: `packages/opencode/src/session/prompt.ts`

Antes de la ejecución, se resuelve qué herramientas tiene el agente. El contexto incluye el `sessionID` que permite a `TaskTool` saber quién es el "padre".

```typescript
// packages/opencode/src/session/prompt.ts

async function resolveTools(input: { ... }) {
  const context = (args, options): Tool.Context => ({
    sessionID: input.session.id, // Contexto de la sesión actual
    // ...
  });
  // ...
}
```

### 3. El Flujo de Uso A2A (Resumen de Código involucrado)

1.  **`processor.ts`**: Detecta `tool-call: task`.
2.  **`task.ts`**: Crea sesión con `parentID` (`Session.create`) y llama a `SessionPrompt.prompt`.
3.  **`prompt.ts`**: Inicia un nuevo `loop` para la sesión hija.
4.  **`task.ts`**: Espera la resolución y devuelve el texto al agente padre.

Esta arquitectura es **nativa** porque reutiliza el motor de procesamiento (`prompt.ts`) de forma recursiva, delegando el estado a la base de datos (sesiones) en lugar de un orquestador externo.
