# Guía del Desarrollador para OpenCode

¡Bienvenido! Esta guía está diseñada para ayudarte a entender las entrañas de OpenCode y cómo puedes contribuir al proyecto. OpenCode es un agente de IA para codificación de código abierto, diseñado para ser flexible, potente y fácil de extender.

## Arquitectura del Proyecto

OpenCode es un **Monorepo** gestionado con [Turborepo](https://turbo.build/) y utiliza [Bun](https://bun.sh/) como runtime y gestor de paquetes.

### Estructura de Directorios

- `packages/opencode`: El corazón del proyecto. Contiene la lógica del agente, el servidor headless (API) y la interfaz de usuario de terminal (TUI).
- `packages/app`: Componentes compartidos de la interfaz web (SolidJS).
- `packages/desktop`: Aplicación de escritorio nativa construida con [Tauri](https://tauri.app/), que envuelve a `packages/app`.
- `packages/ui`: Componentes de UI compartidos.
- `packages/sdk`: SDK de JavaScript para interactuar con OpenCode.
- `infra`: Código de infraestructura gestionado con [SST](https://sst.dev/) (Cloudflare, AWS, etc.).

### Arquitectura Cliente-Servidor

OpenCode utiliza una arquitectura cliente-servidor:
1. **Servidor Headless**: Basado en [Hono](https://hono.dev/), se encarga de la orquestación, gestión de sesiones y comunicación con los LLMs.
2. **Clientes**:
   - **CLI/TUI**: Construido con SolidJS y `@opentui/solid`.
   - **Web App**: Una aplicación web moderna con Vite y SolidJS.
   - **Desktop App**: Tauri envolviendo la aplicación web.

## Conceptos Core

### Agentes (`packages/opencode/src/agent/agent.ts`)

Los agentes definen el comportamiento y las capacidades. Existen agentes primarios y subagentes.
- **build**: El agente por defecto con permisos completos para desarrollar.
- **plan**: Agente de solo lectura para análisis.
- **explore**: Especializado en navegar y entender bases de código rápidamente.

### Herramientas (Tools) (`packages/opencode/src/tool/`)

Las herramientas son las acciones que el agente puede ejecutar (leer archivos, ejecutar comandos bash, buscar en la web).
Para crear una herramienta, se utiliza `Tool.define`. Ejemplo simplificado:

```ts
export const MiHerramienta = Tool.define("mi_herramienta", {
  description: "Descripción de lo que hace",
  parameters: z.object({
    input: z.string()
  }),
  async execute(params, ctx) {
    // Lógica aquí
    return { output: "Resultado" }
  }
})
```

### Proveedores (Providers) (`packages/opencode/src/provider/`)

OpenCode es agnóstico al modelo. Utiliza el **Vercel AI SDK** para conectarse con Anthropic, OpenAI, Google Gemini, AWS Bedrock, y modelos locales. La configuración de estos se centraliza en `provider.ts`.

## Configuración y Desarrollo

### Requisitos
- Bun 1.3 o superior.

### Pasos Iniciales
```bash
bun install
```

### Ejecución en Desarrollo

- **TUI (Terminal UI)**:
  ```bash
  bun dev
  ```
- **Servidor Headless**:
  ```bash
  bun dev serve
  ```
- **Web App**:
  ```bash
  bun run --cwd packages/app dev
  ```
- **Desktop App**:
  ```bash
  bun run --cwd packages/desktop tauri dev
  ```

## Guía de Estilo y Convenciones

Basado en `AGENTS.md`, seguimos estas reglas:
- **Funciones**: Mantén la lógica en una sola función a menos que sea reutilizable.
- **Control de Flujo**: Evita `else`. Prefiere retornos tempranos (`early returns`).
- **Nombres**: Prefiere nombres de una sola palabra para variables y funciones (ej. `journal` en lugar de `prepareJournal`).
- **Tipado**: Evita `any`. Confía en la inferencia de tipos cuando sea posible.
- **Bun APIs**: Usa APIs nativas de Bun como `Bun.file()` siempre que puedas.

## Cómo Contribuir

1. **Issues primero**: Siempre busca o abre un issue antes de enviar un PR.
2. **Pequeños PRs**: Mantén los cambios enfocados y pequeños.
3. **Tests**: Asegúrate de que los tests pasen.
   - `bun test` en `packages/opencode`.
   - `bun run test:unit` en `packages/app`.
4. **Verificación**: Si cambias la UI, incluye capturas de pantalla o videos en tu PR.

Para más detalles, consulta [CONTRIBUTING.md](./CONTRIBUTING.md).
