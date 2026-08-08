

# Revisión Externa de Claude Code

Skills para Claude Code que envían artefactos (planes, código) a una revisión independiente mediante un modelo externo. El revisor explora la base de código, devuelve observaciones, Claude Code las verifica y las corrige: el ciclo se repite hasta obtener aprobación.

## Skills

| Skill | Propósito |
|-------|------------|
| `codex-plan-review` | Revisión de planes de implementación mediante Codex |
| `codex-code-review-fix` | Revisión de código mediante Codex con corrección automática de observaciones |

## Instalación

Copie las carpetas necesarias desde `skills/` al directorio de skills de su proyecto o de forma global en `~/.claude/skills/`.

---

## codex-plan-review

Revisión independiente de planes de implementación mediante Codex.

### Uso

```bash
/codex-plan-review путь/к/плану.md [максимум N итераций]
```

¿Qué sucede:
1. El plan y las reglas del proyecto se envían a Codex para su revisión
2. Codex explora la base de código y devuelve observaciones (JSON)
3. Claude Code verifica cada observación contra el código: la confirma o la rechaza
4. Las observaciones confirmadas se corrigen en el plan
5. El ciclo se repite hasta obtener aprobación o alcanzar el límite de iteraciones (predeterminado: 5)

Resultado: informe final + registro en `.codex-plan-review/`.

### Características clave

**Verificación, no confianza ciega.** Claude Code no acepta las observaciones de Codex a ciegas: cada una se verifica contra el código real. La observación recibe el estado CONFIRMED (se corrige) o FALSE POSITIVE (se rechaza con justificación).

**Ciclo iterativo.** La revisión no es una consulta puntual. Después de las correcciones, el plan se envía para una revisión posterior, y así sucesivamente hasta obtener aprobación o alcanzar el límite de iteraciones.

**Diálogo entre modelos.** Las observaciones rechazadas se devuelven al revisor con argumentos detallados: por qué se rechazaron, con referencias a código específico. El revisor puede aceptar el argumento o insistir con nuevas pruebas.

**Acumulación de contexto entre iteraciones.** Cada iteración le proporciona al revisor listas de observaciones ya aceptadas y rechazadas. El revisor no repite lo anterior y se centra en nuevos problemas.

**Detección de over-engineering.** El revisor evalúa por separado la complejidad excesiva: abstracciones innecesarias, generalización prematura, indirección innecesaria, scope creep. Si el plan se puede implementar de manera más sencilla, se genera una observación.

**Umbral de confianza.** El revisor solo informa sobre problemas con una confianza >= 75. Las críticas estilísticas, suposiciones y «sería bueno» se filtran a nivel del prompt del sistema: solo quedan observaciones significativas.

**Detección de punto muerto.** Si los modelos discrepan y las observaciones comienzan a repetirse, el ciclo se interrumpe y se muestra la discrepancia al usuario para una resolución manual.

**Recopilación automática de contexto.** Codex no ve `.claude/rules/` ni `CLAUDE.md`: el skill recopila todas las reglas (globales y del proyecto) y las pasa explícitamente en el prompt para que la revisión tenga en cuenta los estándares del proyecto.

**Auditoría completa.** Cada iteración se registra: observaciones, verificación, decisiones. El archivo de registro es un artefacto al que se puede volver a consultar.

### Requisitos

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Codex CLI](https://github.com/openai/codex) (el comando `codex` en PATH)

---

## codex-code-review-fix

Revisión independiente de código mediante Codex con corrección automática de los problemas detectados.

### Uso

```bash
/codex-code-review-fix <scope> [N итераций] [план <путь> | <описание доработки>]
```

`scope` — qué revisar: cambios sin confirmar, último commit, rango de commits, archivos específicos, etc.

¿Qué sucede:
1. Los cambios y las reglas del proyecto se envían a Codex para su revisión
2. Codex explora la base de código, ejecuta `git diff` y devuelve observaciones (JSON)
3. Claude Code verifica cada observación contra el código: la confirma o la rechaza
4. Las observaciones CONFIRMED son corregidas por el agente `1c-code-writer` (o su reemplazo); Claude Code verifica el resultado
5. El ciclo se repite: Codex revisa incluyendo las correcciones sin confirmar del agente
6. Finalización: por aprobación, límite de iteraciones o stalemate (impasse) (predeterminado: 5 iteraciones)

Resultado: informe final + registro en `.codex-code-review/`.

### Características clave

**Verificación, no confianza ciega.** Claude Code no acepta las observaciones de Codex a ciegas: cada una se verifica contra el código real. La observación recibe el estado CONFIRMED (se corrige) o FALSE POSITIVE (se rechaza con justificación).

**Corrección automática.** Las observaciones CONFIRMED se pasan al agente `1c-code-writer` con una localización precisa mediante un ancla (`"ИмяМетода > фрагмент кода"`). Después de la corrección, Claude Code verifica el resultado según el ancla y repite la llamada si es necesario.

**Ciclo iterativo con diff actualizado.** Después de las correcciones del agente, Codex recibe un `git diff` sin límite superior: ve tanto los commits originales como las correcciones sin confirmar. El ciclo continúa hasta obtener aprobación completa.

**Diálogo entre modelos.** Las observaciones rechazadas se devuelven al revisor con argumentos detallados: por qué se rechazaron, con referencias a código específico. El revisor puede aceptar el argumento o insistir con nuevas pruebas.

**Acumulación de contexto entre iteraciones.** Cada iteración le proporciona al revisor listas de observaciones ya aceptadas y rechazadas. El revisor no repite lo anterior y se centra en nuevos problemas.

**Detección de over-engineering.** El revisor evalúa por separado la complejidad excesiva: abstracciones innecesarias, generalización prematura, indirección innecesaria. Si el código se puede implementar de manera más sencilla, se genera una observación.

**Umbral de confianza.** El revisor solo informa sobre problemas con una confianza >= 75. Las críticas estilísticas y «sería bueno» se filtran: solo quedan observaciones significativas.

**Detección de punto muerto.** Si los modelos discrepan y las observaciones comienzan a repetirse, el ciclo se interrumpe y se muestra la discrepancia al usuario para una resolución manual.

**Recopilación automática de contexto.** Codex no ve `.claude/rules/` ni `CLAUDE.md`: el skill recopila todas las reglas (globales y del proyecto) y las pasa explícitamente en el prompt.

**Contexto de la modificación.** Opcionalmente se pasa la ruta al plan o una descripción textual de la tarea: Codex lo lee antes de la revisión para comprender la intención de los cambios.

**Auditoría completa.** Cada iteración se registra: observaciones, verificación, decisiones, resultado de las correcciones. El archivo de registro es un artefacto al que se puede volver a consultar.

### Adaptación a otro stack

El skill está optimizado para 1C, pero es adaptable:

- **Agente de corrección** — utiliza `1c-code-writer` del repositorio [1c-ai-feature-dev-workflow](https://github.com/AndreevED/1c-ai-feature-dev-workflow). Reemplácelo por su propio agente o por un subagente general de Claude Code (subagent_type=`general`) en la sección «Llamada al agente» del skill.
- **Prompt del sistema** (`references/review-system-prompt.md`) contiene ejemplos de problemas orientados a 1C: «модули globales», «directivas de compilación», «Попытка/Исключение» (Intento/Excepción), etc. Son solo ejemplos sugerentes: el revisor no está limitado por ellos y usa su criterio experto. Las reglas de su proyecto se recopilan automáticamente a través de `RULES_CONTEXT` (contenido de `.claude/rules/` y `CLAUDE.md`). Si los ejemplos interfieren, reemplácelos por otros relevantes para su stack.

### Requisitos

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [Codex CLI](https://github.com/openai/codex) (el comando `codex` en PATH)
- Agente para corrección de código (predeterminado: `1c-code-writer`, reemplácelo por el suyo)

---

## Autor

Andreev Evgeniy

## Licencia

MIT — consulte [LICENSE](LICENSE)
