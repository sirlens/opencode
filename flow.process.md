# Proceso de Ejecución: De la Solicitud al Resultado

Este documento explica paso a paso cómo OpenCode procesa una tarea, aclarando el rol de los modelos (como el mencionado "BigPickle"), los agentes primarios y la creación dinámica de subagentes.

## 1. Inicio de la Tarea (User Prompt)
Cuando el usuario envía una instrucción (ej: `opencode run "refactoriza este componente"`), el sistema:
1.  **Carga el Agente Primario**: Por defecto es `build`. Este agente tiene un prompt de sistema y un conjunto de herramientas (tools) permitidas.
2.  **Selecciona el Modelo**: El agente utiliza un modelo de lenguaje (LLM). "BigPickle" es un identificador interno que el sistema usa para priorizar modelos de alta capacidad (como Claude 3.5 Sonnet o similares) para tareas complejas.
3.  **Crea la Sesión**: Se genera un ID de sesión único y se guarda el mensaje del usuario en la base de datos local (SQLite).

## 2. El Bucle de Razonamiento (Primary Agent)
El agente primario entra en un bucle gestionado por `SessionPrompt.loop`:
1.  **Análisis**: El modelo recibe el historial y decide qué pasos tomar.
2.  **Toma de Decisiones**: Si la tarea es grande, el modelo puede decidir resolverla él mismo usando herramientas básicas (leer, escribir, bash) o delegar.

## 3. Delegación Dinámica (Subagentes)
Aquí es donde ocurre la "magia" de los subagentes. **No se crean de forma rígida ni automática por el código, sino que el Agente Primario decide crearlos usando la herramienta `task`.**

### ¿Cómo funciona la herramienta `task`?
Si el Agente Primario decide que necesita un experto (ej: un experto en exploración de código), ejecuta la herramienta `task` con parámetros específicos:
-   `subagent_type`: El tipo de experto (ej: `explore`, `general`).
-   `prompt`: La instrucción específica para ese experto.

### El proceso paso a paso de una `task`:
1.  **Nueva Sub-sesión**: La herramienta `task` crea una **nueva sesión hija** en la base de datos, vinculada a la sesión principal.
2.  **Invocación del Subagente**: Se inicia un proceso de `SessionPrompt` independiente para este subagente.
3.  **Ejecución Atómica**: El subagente trabaja en su propia sesión, con sus propias herramientas y contexto limitado, para resolver la tarea específica que se le dio.
4.  **Retorno de Resultados**: Una vez que el subagente termina, su conclusión se devuelve como el "output" de la herramienta `task` al Agente Primario.

## 4. Integración y Finalización
1.  **Recepción**: El Agente Primario recibe el resultado del subagente.
2.  **Continuación**: El Agente Primario evalúa si la respuesta del subagente es suficiente o si necesita realizar más pasos.
3.  **Respuesta Final**: Finalmente, el Agente Primario presenta la solución completa al usuario.

## Resumen del Flujo de Control

```text
Usuario -> [Agente Primario (Brain: BigPickle/Sonnet)]
                |
                |-- Herramienta: Read/Write (Acción Directa)
                |
                |-- Herramienta: Task (Delegación Dinámica)
                |       |
                |       v
                |   [Subagente (Especialista)] --|
                |       |                        |
                |       v                        |
                |   (Ejecuta herramientas)       |-- Sesión Hija (Atómica)
                |       |                        |
                |       v                        |
                |   [Resultado del Subagente] ---|
                |
                v
Agente Primario integra resultados -> Respuesta al Usuario
```

## Conclusión sobre tu pregunta:
-   **¿Se crean subagentes dinámicamente?** SÍ. El agente primario los invoca según sea necesario usando la herramienta `task`.
-   **¿Es atómico?** SÍ, en el sentido de que cada subagente corre en una sesión de base de datos separada con un contexto de mensajes limpio y específico para su subtarea.
-   **¿Rompe el funcionamiento nativo?** NO. Este sistema de delegación recursiva es la base del diseño de OpenCode. Definir nuevos agentes en la configuración simplemente le da al Agente Primario más "expertos" a los que puede llamar.
