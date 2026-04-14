# Guia Paso a Paso: Crear un Agente con Mastra

Esta guia esta pensada para developers que quieren crear un agente con Mastra desde cero, de forma ordenada y replicable.

Basado en documentacion oficial de Mastra:
- https://mastra.ai/docs
- https://mastra.ai/reference/cli/create-mastra
- https://mastra.ai/docs/agents/overview
- https://mastra.ai/docs/agents/using-tools

## 1. Prerrequisitos

Antes de empezar, valida esto:

1. Node.js instalado (recomendado Node 20+).
2. npm/pnpm/yarn/bun disponible.
3. API key de un proveedor de modelos soportado (OpenAI, Anthropic, Google, etc).

Comando para validar Node.js:

```bash
node -v
```

## 2. Crear el proyecto Mastra

Comando principal:

```bash
npm create mastra@latest
```

Alternativas:

```bash
pnpm create mastra@latest
yarn create mastra@latest
bun create mastra@latest
```

Durante el wizard del CLI, completa lo siguiente:

1. Nombre del proyecto.
2. Template/base del proyecto.
3. Si quieres incluir ejemplo inicial o no.

Si quieres omitir ejemplo inicial:

```bash
npm create mastra@latest --no-example
```

## 3. Entrar al proyecto e instalar dependencias

Si el CLI no lo hizo automaticamente, entra al directorio e instala paquetes:

```bash
cd <tu-proyecto>
npm install
```

## 4. Configurar variables de entorno (API key)

Crea o completa el archivo `.env` del proyecto con la key del proveedor que usaras.

Ejemplos:

```env
OPENAI_API_KEY=tu_api_key
ANTHROPIC_API_KEY=tu_api_key
GOOGLE_GENERATIVE_AI_API_KEY=tu_api_key
```

Importante:
- Usa solo la variable del proveedor/modelo que realmente vas a usar.
- No subas `.env` al repositorio.

## 5. Entender estructura base del proyecto

En Mastra normalmente trabajas en `src/mastra/`:

- `agents/`: definicion de agentes.
- `tools/`: herramientas que el agente puede ejecutar.
- `workflows/`: orquestacion por pasos.
- `scorers/`: evaluaciones del comportamiento.
- `index.ts`: registro central de agentes, workflows, storage, observability.

### ¿Para que sirve cada cosa?

#### Tools (`tools/`)

Son las capacidades concretas que le das al agente para interactuar con el mundo exterior. Sin tools, el agente solo puede razonar en base a su conocimiento del modelo; con tools puede:

- Consultar APIs externas (clima, precios, datos en tiempo real).
- Leer o escribir bases de datos.
- Ejecutar logica de negocio propia.
- Integrarse con servicios de terceros.

El agente decide autonomamente *cuando* y *con que argumentos* llamar cada tool segun la conversacion. Tu defines que tools estan disponibles y el modelo elige.

#### Workflows (`workflows/`)

Son flujos de trabajo orquestados paso a paso. A diferencia de los agentes (que razonan libremente), un workflow ejecuta una secuencia de pasos definida y predecible. Los workflows son ideales cuando:

- Necesitas un proceso con pasos ordenados y condiciones claras.
- Quieres control total del flujo sin depender del razonamiento del modelo.
- El proceso involucra aprobaciones, esperas, o ramas (branches).
- Necesitas reproducibilidad: el mismo input siempre da el mismo flujo.

Ejemplo: un pipeline de onboarding, un flujo de aprobacion de gastos, un ETL con pasos definidos.

#### Scorers (`scorers/`)

Son evaluadores automaticos que miden la calidad de las respuestas del agente en tiempo real. Sirven para:

- Detectar si el agente usa las tools correctas cuando corresponde.
- Medir si las respuestas son completas o les falta informacion clave.
- Validar logica especifica del dominio (ej: que se traduzcan correctamente nombres de ciudades).
- Alimentar dashboards de calidad en Mastra Studio y Mastra Cloud.

Cada scorer produce un score entre 0 y 1 con una explicacion, y puedes configurar con que frecuencia se ejecutan (por ejemplo, en el 100% de las llamadas o en una muestra).

## 6. Crear o ajustar una herramienta (Tool)

Las tools se crean con `createTool` y se validan con `zod`.

Ejemplo minimo:

```ts
import { createTool } from '@mastra/core/tools';
import { z } from 'zod';

export const demoTool = createTool({
  id: 'demo-tool',
  description: 'Ejemplo de tool',
  inputSchema: z.object({
    query: z.string(),
  }),
  outputSchema: z.object({
    result: z.string(),
  }),
  execute: async ({ query }) => {
    return { result: `Procesado: ${query}` };
  },
});
```

Buenas practicas:
- Define bien `inputSchema` y `outputSchema`.
- Maneja errores de red/API con mensajes claros.
- Mantener IDs descriptivos y estables.

## 7. Crear o ajustar el agente

Un agente se define con la clase `Agent`.

Campos clave que debes configurar:

1. `id`: identificador unico tecnico.
2. `name`: nombre legible para devs/Studio.
3. `instructions`: reglas de comportamiento.
4. `model`: formato `proveedor/modelo`.
5. `tools`: herramientas disponibles para el agente.
6. `memory` (opcional): persistencia de contexto conversacional.
7. `scorers` (opcional): evaluacion de calidad.

Ejemplo base:

```ts
import { Agent } from '@mastra/core/agent';
import { demoTool } from '../tools/demo-tool';

export const demoAgent = new Agent({
  id: 'demo-agent',
  name: 'Demo Agent',
  instructions: 'Eres un asistente util y preciso.',
  model: 'google/gemini-2.5-pro',
  tools: { demoTool },
});
```

## 8. Registrar el agente en Mastra

En `src/mastra/index.ts`, registra el agente para que Mastra Studio y el runtime lo detecten:

```ts
import { Mastra } from '@mastra/core/mastra';
import { demoAgent } from './agents/demo-agent';

export const mastra = new Mastra({
  agents: { demoAgent },
});
```

Si tienes workflows y scorers, registralos ahi tambien.

## 9. Levantar Mastra Studio

Inicia entorno de desarrollo:

```bash
npm run dev
```

Abre:

- http://localhost:4111

Desde Studio puedes:

1. Seleccionar el agente.
2. Probar prompts.
3. Ver llamadas a tools.
4. Revisar trazas y comportamiento.

## 10. Probar flujo minimo de calidad

Checklist rapido:

1. El agente responde sin errores.
2. Llama tools cuando corresponde.
3. Respeta instrucciones del system prompt.
4. Maneja input invalido con mensajes claros.
5. No rompe si la API externa falla.

## 11. Build para produccion

```bash
npm run build
```

Si el build falla:

1. Revisa variables de entorno.
2. Revisa tipos TS (`tsconfig`).
3. Revisa imports y registro en `src/mastra/index.ts`.

## 12. Troubleshooting comun

1. Error de API key:
   - Variable en `.env` no coincide con el proveedor/modelo.
2. El agente no aparece en Studio:
   - No esta registrado en `src/mastra/index.ts`.
3. El agente no llama la tool:
   - La tool no esta en `tools` del agente o instrucciones ambiguas.
4. Error de modulo/TS:
   - Asegura configuracion ES2022 y dependencias al dia.

## Recomendaciones finales para el equipo

1. Mantener prompts versionados y cortos.
2. Crear una carpeta de tools por dominio (weather, billing, support, etc).
3. Escribir scorers desde el inicio para medir calidad.
4. Probar cada cambio en Studio antes de merge.
5. Documentar ejemplos de prompts validos por agente.

---

Con este flujo ya puedes crear y escalar agentes en Mastra de forma consistente para todo el equipo.
