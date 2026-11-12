# Agentes para el flujo HU → PR

Cinco agentes: cuatro especialistas y un orquestador. Se pueden usar de dos formas:

| Modo | Cómo | Cuándo |
|---|---|---|
| **Manual (Fase 1)** | Tú encadenas con los botones de handoff | Primeras HU piloto, para medir el flujo |
| **Orquestado (Fase 2)** | Le das la HU al `orchestrator` y él delega | Cuando el flujo manual ya es mecánico |

## Archivos

| Archivo | Rol |
|---|---|
| `orchestrator.agent.md` | Decide el orden, aplica gates, **único escritor de `progress.json`** |
| `qa-analyst.agent.md` | Estrategia y casos de prueba desde la spec |
| `backend-dev.agent.md` | Backend Fastify + unit |
| `frontend-dev.agent.md` | Frontend Angular + unit |
| `qa-automation.agent.md` | api-tests y ui-tests (Playwright) |

## Instalación

1. Copia los `.agent.md` a `.github/agents/` del repo donde abras VS Code, o a una carpeta común registrada en `chat.agentFilesLocations` (útil con workspace multi-root y repo de specs).
2. Las skills (`test-strategy`, `test-cases`, `unit`, `api-tests`, `ui-tests`, `frontend-dev`, `backend-dev`) se referencian por nombre. Deben estar donde VS Code las descubre (por ejemplo `.github/skills/<nombre>/SKILL.md`).
3. Abre el selector de agentes en el chat y verifica que aparecen los cinco.

## Supuestos que debes revisar

- **Nombres de herramientas** en `tools:` (`agent`, `read`, `search`, `edit`, `execute`): confirma que coinciden con tu versión de VS Code. Las que no existan se ignoran.
- **El orquestador tiene `edit` y `execute`**: la restricción "solo `progress.json`" y "solo re-ejecutar verificaciones" es una instrucción, no un permiso. Revisa sus diffs durante el piloto.
- **Subagentes:** el campo `agents` del orquestador requiere la herramienta `agent` habilitada. Verifica que la invocación de subagentes esté activa en tu entorno.
- **Ruta de specs:** se asume `specs/<HU-ID>/` en el repo de specs. Cámbiala si usas la estructura de OpenSpec u otra.
- **Comandos de verificación:** el orquestador usa los que cada repo declare (por defecto typecheck, lint y unit). Decláralos en cada repo (por ejemplo en `copilot-instructions.md`).
- **Protección real por carpeta:** CODEOWNERS, branch policy y el check anti-`.skip` del pipeline. Las reglas "no editar" de los agentes no sustituyen eso.
- Los hooks, handoffs y subagentes en VS Code evolucionan: valida los campos contra la documentación de tu versión.

## Estructura por HU

```
specs/<HU-ID>/
  spec.md            # aprobada por una persona (gate)
  test-strategy.md   # qa-analyst
  test-cases.md      # qa-analyst, con Mapa de trazabilidad (gate humano)
  contract.*         # contrato API (OpenAPI/Pact)
  progress.json      # SOLO el orquestador (o la persona en modo manual)
  failures.md        # qa-automation, si hay fallos
```

## Esquema de `progress.json`

```json
{
  "hu": "HU-000",
  "criteria": [
    {
      "id": "AC-1",
      "cases": ["TC-1", "TC-2"],
      "backend": "pending",
      "frontend": "pending",
      "api_test": "pending",
      "ui_test": "pending",
      "corrections": 0,
      "evidence": []
    }
  ]
}
```

- Estados válidos: `pending`, `failed`, `passes`, `blocked`.
- Un campo pasa a `passes` solo después de que el orquestador (o la persona) re-ejecute la verificación. Se agrega a `evidence` una línea: `fecha agente: comando → resultado (verificado)`.
- `corrections` cuenta los ciclos de corrección del criterio. Máximo 2; al tercero pasa a `blocked`.
- Nadie elimina criterios ni edita IDs.

## Formato del bloque Reporte (todos los especialistas)

Los especialistas **no escriben `progress.json`**. Terminan su respuesta con una línea por criterio:

```
AC-n | campo | passes|failed|blocked | comando | resultado resumido
```

Ejemplo:

```
AC-1 | backend | passes | npm test -- --grep "AC-1" | 4 passed, 0 failed
AC-2 | backend | failed | npm test -- --grep "AC-2" | 1 failed: expected 401, got 200
```

## Uso manual con una HU piloto

1. Aprueba `spec.md` (gate humano).
2. `qa-analyst`: "Diseña la estrategia y casos para `specs/<HU-ID>/`". Revisa `test-cases.md` (gate humano) y crea `progress.json` desde el Mapa de trazabilidad.
3. Handoff a `backend-dev` y `frontend-dev`. Registra tú sus reportes en `progress.json`.
4. Handoff a `qa-automation`. Si hay fallos, usa los handoffs de corrección y repite.
5. Abre el PR: pipeline y revisión humana cierran el ciclo.

Anota en cada HU: dónde hubo retrabajo, qué pregunta abierta apareció tarde y qué paso fue pura mecánica.

## Uso orquestado

1. Selecciona `orchestrator` y escribe: "Ejecuta la HU `<HU-ID>`".
2. El orquestador valida la spec, delega a `qa-analyst` y **se detiene** esperando tu aprobación de `test-cases.md`.
3. Respondes "aprobado" (o pides cambios). Continúa con implementación, verificación, automatización y correcciones.
4. Al final entrega el resumen por criterio. Abres el PR.

Primera ejecución: observa de cerca. Revisa que el orquestador no haya editado nada fuera de `progress.json`, que respete el gate y que re-ejecute las verificaciones.

## No cubre (va en CI)

Pipeline de PR con tests obligatorios, branch policy, revisor obligatorio sobre carpetas de tests y check anti-`.skip` o borrado de tests.
