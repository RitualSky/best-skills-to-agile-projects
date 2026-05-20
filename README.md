# SDD - Best Skills to Agile Projects

Flujo de desarrollo guiado por especificación (SDD → BDD → TDD) orquestado por agent skills.

---

## Tabla de Contenidos

- [Flujo del Pipeline](#flujo-del-pipeline)
- [Componentes del Sistema](#componentes-del-sistema)
  - [Fuentes de Contexto e Inputs](#1-fuentes-de-contexto-e-inputs)
  - [Catálogo de Agent Skills](#2-catálogo-de-agent-skills-comandos-)
  - [Artefactos del Pipeline](#3-artefactos-del-pipeline-entregables-md)
- [Flujo de Ejecución Paso a Paso](#flujo-de-ejecución-paso-a-paso)
  - [Fase 1 – Inicialización y Contexto](#fase-1--inicialización-y-contexto)
  - [Fase 2 – Spec-Driven Development (SDD)](#fase-2--spec-driven-development-sdd)
  - [Fase 3 – BDD y Planificación](#fase-3--bdd-y-planificación)
  - [Fase 4 – Ejecución Inteligente (TDD)](#fase-4--ejecución-inteligente-tdd)
  - [Fase 5 – Manejo de Errores](#fase-5--manejo-de-errores)
  - [Fase 6 – Cierre y Persistencia](#fase-6--cierre-y-persistencia)

---

## Flujo total del SDD

```text
               ┌──────────────────────────────────────┐
               │  VERIFICACIÓN INICIAL DE CARPETAS    │
               │            [ @codebase ]             │
               │                  │                   │
               │                  ▼                   │
               │      /───────────────────────\       │
               │     < ¿Existe directorio docs/ >     │
               │      \───────────────────────/       │
               │         │                 │          │
               │      NO │                 │ SI       │
               │         ▼                 │          │
               │ ┌───────────────┐         │          │
               │ │ /docs-creator │         │          │
               │ └───────┬───────┘         │          │
               │         │ (Crea)          │          │
               │         ▼                 │          │
               │ ┌───────────────┐         │          │
               │ │docs/architect │         │          │
               │ │docs/features  │         │          │
               │ └───────┬───────┘         │          │
               └─────────┼─────────────────┼──────────┘
                         │                 │
                         └────────┬────────┘
                                  │
                                  ▼
   [ Contexto Inicial: History_Jira ] + [ Directivas Globales: AGENTS.md ]
                  │
                  ▼
      ┌───────────────────────┐
      │    /spec-creator      │ ◄────────────────────────────────────────┐
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
      ┌───────────────────────┐                                          │
      │       spec.md         │                                          │
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
      /─── ¿Revisión Humana? ───\  NO                                    │ (Bucle de Feedback:
      \─────── OK o Falla ──────/ ───────► [ Re-evaluar Spec ]           │  Inconsistencia Lógica
                  │                                                      │  o de Arquitectura)
               SI │                                                      │
                  ▼                                                      │
      ┌───────────────────────┐                                          │
      │     /bdd-creator      │                                          │
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
      ┌───────────────────────┐                                          │
      │       bdd.md          │                                          │
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
      ┌───────────────────────┐                                          │
      │    /plan-creator      │                                          │
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
      ┌───────────────────────┐                                          │
      │       plan.md         │                                          │
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
      ┌───────────────────────┐                                          │
      │     /apply-spec       │                                          │
      │  (Contenedor TDD)     │                                          │
      │  ┌─────────────────┐  │                                          │
      │  │  tdd (test)     │  │                                          │
      │  │       │         │  │                                          │
      │  │       ▼         │  │                                          │
      │  │  code (prod)    │  │                                          │
      │  └─────────────────┘  │                                          │
      └───────────────────────┘                                          │
                  │                                                      │
                  ▼                                                      │
        /─── ¿Error Técnico ────\  SI                                    │
        \─── de Arquitectura? ──/ ───────────────────────────────────────┘
                  │
               NO │
                  ▼
      ┌───────────────────────┐
      │     /docs-update      │ ───► Actualiza: [ docs/architecture ] & [ docs/features ]
      └───────────────────────┘
                  │
                  ▼
      ┌───────────────────────┐
      │       /commit         │ ───► Envío a: [ repository ]
      └───────────────────────┘
```

---

## Componentes del Sistema

### 1. Fuentes de Contexto e Inputs

| Fuente | Descripción |
|---|---|
| **History_Jira** | El disparador del pipeline. Contiene el requerimiento de negocio original en lenguaje natural escrito por Product Owners o Stakeholders. |
| **@codebase** | El estado actual del repositorio de código. Proporciona el mapa de dependencias y la estructura técnica real del proyecto. |
| **AGENTS.md** | Archivo maestro de gobernanza que contiene las reglas globales de codificación, convenciones de diseño y directrices de comportamiento para toda la suite de agentes. |

---

### 2. Catálogo de Agent Skills (Comandos `/`)

| Comando | Agente Skill | Responsabilidad Principal |
|---|---|---|
| `/story-enricher` | Enriquecedor de Historias de Usuario| Especializado en enriquecer las historias de usuario con detalles técnicos y contextuales  |
| `/docs-creator` | Pre-condicional | Analiza la raíz de `@codebase`. Si no encuentra el directorio `docs/`, inicializa la estructura de carpetas estándar (`docs/architecture` y `docs/features`). |
| `/spec-creator` | Diseñador de Contratos (SDD) | Consolida la historia de Jira, las directrices de AGENTS.md y la documentación existente para generar un documento de especificación técnica determinista y unificado. |
| `/bdd-creator` | Analista de Comportamiento (BDD) | Traduce el documento formal de especificación en escenarios de comportamiento legibles por humanos y automatizables por máquinas (Casos de prueba en formato funcional). |
| `/plan-creator` | Planificador Táctico | Descompone los casos de prueba globales en una lista ordenada de micro-tareas atómicas e independientes basadas en el principio INVEST. |
| `/apply-spec` | Ingeniero de Ejecución (TDD) | El motor de desarrollo. Extrae secuencialmente las tareas del plan, escribe primero las aserciones de prueba unitarias e implementa la mínima cantidad de código para cumplirlas. |
| `/docs-update` | Documentador de Sistema | Sincroniza las especificaciones técnicas finales y el código resultante con las carpetas de documentación del repositorio, evitando la obsolescencia. |
| `/commit` | Orquestador de Despliegue | Realiza el control de calidad final de la rama, empaqueta los cambios validados y efectúa el envío seguro al repositorio central. |

---

### 3. Artefactos del Pipeline (Entregables `.md`)

#### `spec.md` — Contrato de Especificación

Documento rector que rige la funcionalidad. Estructura estricta:

- **Objetivo y Alcance** — Qué soluciona y los límites de la intervención.
- **Qué no cubre** — Exclusiones explícitas para mitigar el scope creep.
- **Reglas de Negocio (MUST)** — Criterios de aceptación inflexibles.
- **UI/UX** — Especificación visual o de flujos de pantalla si aplica.
- **Componentes Afectados** — Lista de archivos del `@codebase` que serán modificados o creados.

#### `bdd.md` — Casos de Comportamiento

Listado secuencializado de Casos de Prueba (Caso #1 al Caso #n) estructurados preferentemente bajo sintaxis **Given-When-Then**.

#### `plan.md` — Gestión de Estado Activo

Lista de verificación dinámica utilizada por los agentes para mitigar la pérdida de contexto:

| Estado | Significado |
|---|---|
| `[X]` | Tarea Finalizada — validada e integrada. |
| `[*]` | Tarea en Progreso — tarea activa en el contenedor de ejecución. |
| `[ ]` | Tarea Pendiente — en cola de espera. |

---

## Flujo de Ejecución Paso a Paso

### Fase 1 – Inicialización y Contexto

El pipeline arranca con una **History_Jira**.

El skill `/docs-creator` comprueba de forma segura la presencia de documentación en `@codebase`. Si está ausente, genera la infraestructura base en `docs/`.

---

### Fase 2 – Spec-Driven Development (SDD)

El skill `/spec-creator` procesa las entradas y genera el archivo formal `spec.md`.

> **Revisión Humana (Gate):** El pipeline detiene su automatización temporalmente. Un ingeniero humano evalúa si la especificación cumple las expectativas de negocio.
>
> - **Ruta NO:** Se devuelve el control a `/spec-creator` mediante el flujo "afina spec" sin gastar recursos en las siguientes etapas.
> - **Ruta SI:** El flujo avanza con luz verde hacia la fase técnica.

---

### Fase 3 – BDD y Planificación

1. El skill `/bdd-creator` consume la especificación aprobada y genera los escenarios funcionales detallados en `bdd.md`.
2. El skill `/plan-creator` toma esos escenarios y fragmenta el alcance del proyecto en una secuencia lógica de micro-tareas dentro de `plan.md`.

---

### Fase 4 – Ejecución Inteligente (TDD)

El skill `/apply-spec` inicializa el contenedor aislado y realiza una lectura del `plan.md`.

**Bucle iterativo por tarea:**

1. Identifica la primera tarea con estado `[ ]`, cambia su estado a `[*]` y asume el contexto exclusivo de dicho requerimiento.
2. **Ciclo TDD:**
   - Genera el archivo `tdd` (test) de la función. Al ejecutarse, falla _(Fase Roja)_.
   - Escribe el archivo de producción `code` _(Fase Verde)_ hasta que el entorno aprueba la aserción.
3. Modifica el `plan.md` cambiando el estado a `[X]` e inicia la siguiente subtarea.

---

### Fase 5 – Manejo de Errores

Al concluir el código, se evalúa la estabilidad estructural mediante la compuerta de errores:

| Tipo de Error | Respuesta |
|---|---|
| **Error Menor de Código** — fallos de sintaxis o aserciones ordinarias | Itera internamente en el contenedor de ejecución mediante el Ciclo TDD hasta subsanarse. |
| **Error de Arquitectura / Lógica de Negocio** — conflicto técnico insalvable con `@codebase` | El flujo se desvía de inmediato hacia `/spec-creator`, forzando una re-evaluación de la Spec y activando de nuevo la aprobación humana. |

---

### Fase 6 – Cierre y Persistencia

1. El skill `/docs-update` recopila los artefactos temporales (`spec.md`, `bdd.md`), los concatena al contexto del `@codebase` y actualiza la documentación viva en `docs/architecture` y `docs/features`.
2. El skill `/commit` empaqueta el código productivo validado junto con la documentación fresca y empuja los cambios de forma determinista hacia el `repository` central.
