# Contexto: orquestación de agentes para el flujo HU → PR

Resumen de la conversación que dio origen a estos artefactos. Sirve para retomar el trabajo en otra sesión o compartirlo con el equipo.

**Fecha:** 12 de noviembre de 2026

---

## 1. Punto de partida

Un equipo de desarrollo tiene skills refinadas que usa **una a una, de forma manual**. La pregunta: ¿cuál es el siguiente paso?

**Entorno declarado**
- Repos en **Azure Repos** (Azure DevOps). Asistente de IA: **GitHub Copilot** (VS Code).
- Skills de QA: `test-strategy`, `test-cases`, `unit`, `api-tests`, `ui-tests`.
- Skills de desarrollo: `frontend-dev`, `backend-dev`.
- Frontend Angular y backend Fastify (TypeScript), en repos separados; UI con Playwright + TypeScript.
- Repo de specs dedicado y workspace multi-root como capa de coordinación.

## 2. Conclusiones principales

1. **Siguiente paso:** pasar de skills sueltas a un **flujo orquestado**, con contratos explícitos entre skills (artefactos en archivo, no en el chat) y puntos de control.
2. **Niveles de madurez:** (1) skills manuales → (2) workflow encadenado → (3) orquestación con agentes → (4) evals continuos.
3. **Orden fijo de skills:** es buena práctica cuando el orden es conocido y estable (workflow). Si el orden depende del input, usar routing; si no se puede predecir, orquestador dinámico. Lo habitual es un híbrido: esqueleto fijo para gates y fases críticas, autonomía dentro de cada paso.
4. **Preocupación central:** que un agente no haga pruebas. La solución no es un orden rígido en el prompt, sino **hacer imposible dar la tarea por terminada sin pruebas**, con verificación externa al agente.
5. **Un agente por rol, no por skill.** Una skill es una capacidad; un agente aporta aislamiento de contexto y herramientas restringidas.

## 3. Referencias y evidencia revisada

| Tema | Hallazgo | Fuente |
|---|---|---|
| Patrones de agentes | Prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer. Recomendación: empezar simple y escalar solo si hace falta | anthropic.com/engineering/building-effective-agents |
| Sistema multi-agente de investigación | Lead Opus + subagentes Sonnet superó a Opus solo en 90.2% en evaluación interna de investigación; el uso de tokens explica ~80% de la varianza; costo ≈15× tokens de un chat | anthropic.com/engineering/multi-agent-research-system |
| Harness para agentes largos | Agente inicializador + agente de código; lista de features con campo `passes`; una feature a la vez; progreso en archivos | anthropic.com/engineering/effective-harnesses-for-long-running-agents |
| Hooks de Copilot | `.github/hooks/*.json`; evento `agentStop` puede bloquear la continuación; en VS Code los hooks están en public preview | docs.github.com → Copilot → hooks |
| Custom agents en VS Code | `.agent.md` en `.github/agents`; campos `agents` (subagentes) y `handoffs`; ubicaciones extra con `chat.agentFilesLocations` | code.visualstudio.com/docs/copilot/customization/custom-agents |

**Límites de la evidencia:** los resultados de Anthropic vienen de evaluaciones internas en investigación y construcción de apps. No se encontraron casos públicos específicos de orquestación para QA.

## 4. Arquitectura acordada

### Cinco agentes

| Agente | Skills | Edita | No toca |
|---|---|---|---|
| `orchestrator` | ninguna | solo `progress.json` | código y tests |
| `qa-analyst` | test-strategy, test-cases | plan y casos en el repo de specs | código |
| `backend-dev` | backend-dev, unit | repo Fastify (src + unit) | tests de API |
| `frontend-dev` | frontend-dev, unit | repo Angular (src + unit) | tests UI/E2E |
| `qa-automation` | api-tests, ui-tests | tests de API y Playwright | src |

### Flujo con gates

1. Spec/HU aprobada → **gate humano**.
2. `qa-analyst` → `test-strategy.md` y `test-cases.md` → **gate humano**.
3. Contrato API (OpenAPI/Pact) definido.
4. `backend-dev` ‖ `frontend-dev` en paralelo (secuencial si la HU cambia el contrato).
5. `qa-automation`: api-tests y Playwright, desde los casos aprobados.
6. Pipeline de PR (gate automático) → revisión humana.

### Decisiones tomadas

| Decisión | Resultado |
|---|---|
| Independencia | Quien escribe los tests de aceptación no escribe la implementación; los tests salen de la spec, no del código |
| `unit` | Lo escriben los devs (caja blanca); el CI vigila cobertura, borrado de tests y `.skip` |
| Escritor de `progress.json` | **Solo el orquestador** (en modo manual, la persona). Los especialistas reportan evidencia en un bloque `Reporte` |
| Verificación | El orquestador **re-ejecuta** los comandos antes de marcar `passes` |
| Paralelismo | Back ‖ front si el contrato es estable; secuencial si cambia |
| Reintentos | Máximo 2 ciclos de corrección por criterio; al tercero, `blocked` y escalar |
| Límite de invocaciones | 12 subagentes por HU |
| Nombre del entregable de estrategia | `test-strategy.md` (en vez de `test-plan.md`), para coincidir con la skill |
| Trazabilidad | Los tests llevan el ID del criterio (`AC-n`) en su nombre |

### Estrategia de arranque

- **Fase 1:** cuatro especialistas encadenados con `handoffs` (manual, orden fijo, gates humanos con clic). Correr una **HU piloto** y anotar dónde hubo retrabajo.
- **Fase 2:** agregar el `orchestrator` con subagentes para los pasos que resultaron mecánicos.

## 5. Capas de garantía (que el agente no cierre sin pruebas)

| Capa | Fuerza |
|---|---|
| Instrucción en el prompt / `copilot-instructions.md` | Débil |
| Checklist/artefacto de "done" (`progress.json`) | Media |
| Hooks locales (`agentStop` en Copilot) | Fuerte, pero en preview y variable por superficie |
| **CI con checks obligatorios en Azure Pipelines** | **Fuerte, fuera del agente (la capa que realmente protege)** |
| Agente verificador / revisión humana | Fuerte |

### Capa 1 (pendiente de implementar): Azure Pipelines

Pipeline de PR sobre `develop`:

```yaml
trigger: none
pr:
  branches: { include: [develop] }
steps:
  - script: npm ci
  - script: npx playwright install --with-deps
  - script: npx tsc --noEmit
  - script: npm test
  - script: npx playwright test --project=smoke
```

Branch policy en `develop`: build validation requerido, mínimo 1 revisor humano, y revisor obligatorio sobre `tests/`, `playwright.config.ts` y `azure-pipelines.yml`.

Check anti-trampa (paso extra del pipeline):

```bash
git diff --diff-filter=D --name-only origin/develop... | grep -E '\.(spec|test)\.ts$' && exit 1
git diff origin/develop... | grep -E '^\+.*(test|it|describe)\.(skip|only)' && exit 1
exit 0
```

Orden sugerido: pipeline de PR + branch policy → check anti-`.skip` → protección de carpetas de tests → hook local → (más adelante) mutation testing (Stryker) para detectar tests triviales.

## 6. Supuestos y riesgos abiertos

- Los nombres de herramientas en `tools:` (`agent`, `read`, `search`, `edit`, `execute`) deben validarse contra la versión de VS Code.
- La invocación de subagentes y los hooks de VS Code están en evolución; validar cada campo contra la documentación vigente.
- Hasta donde se sabe, el Copilot cloud agent opera sobre repos de GitHub; con Azure Repos se usa Copilot en el IDE (agent mode) o la CLI.
- Las restricciones en los `.agent.md` son por herramienta, no por carpeta. La protección por carpeta depende de CODEOWNERS, branch policy y el check del pipeline.
- El orquestador tiene `edit` y `execute`: "solo `progress.json`" es una instrucción, no un permiso. Revisar sus diffs en las primeras corridas.
- Se asume la ruta `specs/<HU-ID>/` para los artefactos; ajustar si el repo de specs usa otra estructura (por ejemplo, la de OpenSpec).
- Las skills se referencian por nombre y deben estar donde VS Code las descubre (por ejemplo `.github/skills/<nombre>/SKILL.md`).

## 7. Próximos pasos

1. Instalar los `.agent.md` y verificar que aparecen en el selector de agentes.
2. Correr una **HU piloto en modo manual** (Fase 1) y registrar retrabajo, preguntas abiertas tardías y pasos puramente mecánicos.
3. Implementar la **Capa 1** en Azure Pipelines (pipeline de PR, branch policy, check anti-`.skip`).
4. Pasar la misma HU, u otra, por el `orchestrator` y comparar con el modo manual.
5. Más adelante: evals de las skills y del flujo (dataset pequeño de casos con resultado esperado) y mutation testing.

## 8. Contenido del paquete

```
context.md                  # este archivo
agents/
  README.md                 # instalación, convenciones, esquema de progress.json, formato de Reporte
  orchestrator.agent.md
  qa-analyst.agent.md
  backend-dev.agent.md
  frontend-dev.agent.md
  qa-automation.agent.md
```
