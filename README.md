# Spec-Driven Development (SDD) -- Sandbox

Este repositorio es un sandbox práctico de Spec-Driven Development
(SDD). No es teoría. Es una estructura mínima y reutilizable para
trabajar proyectos donde la especificación es el artefacto central que
guía diseño, implementación y validación.

La idea es simple:

Primero se define claramente qué se quiere construir y bajo qué reglas.
Después se escribe código.

------------------------------------------------------------------------

## Objetivo del repo

-   Tener una estructura base reutilizable para proyectos guiados por
    especificaciones.
-   Definir estándares claros de codificación desde el día cero.
-   Incluir un ejemplo de plan generado con Cloud Code.
-   Mostrar un ejemplo concreto de especificación funcional/técnica.

Este repo puede servir como: - Template para nuevos proyectos. - Base
para trabajos freelance. - Framework interno para equipos. - Punto de
partida para integrar IA generativa en el flujo de desarrollo.

------------------------------------------------------------------------

## ¿Qué es Spec-Driven Development?

Spec-Driven Development (SDD) es un enfoque donde:

1.  La especificación es el contrato.
2.  El código es una implementación verificable contra esa
    especificación.
3.  Los estándares de codificación son explícitos.
4.  La trazabilidad entre requerimiento → diseño → código es clara.

A diferencia del típico "arranco a codear y vemos", acá:

-   Se formaliza el problema.
-   Se definen criterios de aceptación.
-   Se documentan decisiones.
-   Se reduce ambigüedad antes de escribir código.

------------------------------------------------------------------------

### /specs

Contiene las especificaciones funcionales y/o técnicas.

Cada spec debería incluir:

-   Contexto del problema
-   Objetivo
-   Alcance
-   Reglas de negocio
-   Entradas y salidas esperadas
-   Criterios de aceptación
-   Casos límite

La spec es el documento principal del proyecto.

------------------------------------------------------------------------

### /plans

Incluye planes de ejecución generados manualmente o con herramientas
como Cloud Code.

Un plan típico puede incluir:

-   Breakdown en tareas
-   Dependencias
-   Supuestos
-   Estrategia de implementación
-   Validaciones previstas

El plan traduce la spec en pasos accionables.

------------------------------------------------------------------------

### /standards

Define estándares explícitos de desarrollo:

-   Convenciones de nombres
-   Estructura de proyecto
-   Manejo de errores
-   Logging
-   Formato de commits
-   Reglas de documentación

Esto evita discusiones innecesarias y hace que el proyecto escale mejor.

------------------------------------------------------------------------

## Flujo de trabajo sugerido

1.  Crear una nueva spec en /specs.
2.  Validar alcance y criterios de aceptación.
3.  Generar o definir plan en /plans.
4.  Implementar siguiendo /standards.
5.  Validar contra la spec, no contra "lo que parecía que era".

------------------------------------------------------------------------

## Integración con IA

Este repositorio está pensado para funcionar muy bien con modelos de
lenguaje.

Ejemplos de uso:

-   Generar un plan técnico a partir de una spec.
-   Validar que el código cumple criterios de aceptación.
-   Detectar ambigüedades en requerimientos.
-   Generar tests desde la especificación.

La clave es que la IA trabaja mejor cuando: - Las reglas están claras. -
El contexto está estructurado. - Las expectativas están documentadas.

------------------------------------------------------------------------

## Casos de uso ideales

-   Microservicios bien definidos.
-   ETLs y pipelines de datos.
-   APIs.
-   Automatizaciones.
-   Productos SaaS pequeños.
-   Proyectos freelance donde necesitás profesionalizar entregables.

------------------------------------------------------------------------

## Cómo usar este repo como template

1.  Clonar el repositorio.
2.  Crear una nueva spec en /specs.
3.  Adaptar estándares si hace falta.
4.  Generar plan.
5.  Comenzar implementación.

Opcional: - Convertirlo en un template repository en GitHub. -
Integrarlo con CI/CD. - Agregar validaciones automáticas de estilo.

------------------------------------------------------------------------

## Filosofía

-   Menos ambigüedad.
-   Más claridad.
-   Código con intención.
-   Documentación que sirve.
-   Especificación como contrato real.
