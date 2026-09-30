# Contexto del proyecto — the-seedskill

> Este archivo existe para que cualquier IA o persona que retome este proyecto (incluso con
> otra cuenta de Claude u otra IA, sin acceso a conversaciones anteriores) entienda qué es,
> de dónde salen las lecciones, hasta dónde están asimiladas y cómo agregar más sin repetir
> trabajo. Se actualiza en cada sesión que toque la skill — no es la documentación de la
> skill (eso es `SKILL.md` + `references/`), es el **registro de procedencia**.
>
> Última actualización: 2026-09-30 (misma fecha: se puso al día también el `CONTEXTO.md` del webapp).

**Este archivo se carga solo.** El `CLAUDE.md` de una línea en la raíz lo importa (mismo
patrón que `prp-gestion-webapp`). Ojo: cuando la skill está *instalada* en
`~/.claude/skills/the-seedskill`, Claude Code carga `SKILL.md`, no este archivo — este es
para quien *mantiene* el repo.

## Qué es

Una skill de Claude Code (`the-seedskill`, antes `multitenant-saas-playbook`) de L
(co-founder de PRP Data Analytics, Obligado, Itapúa, Paraguay). Es un playbook de
ingeniería para construir y evolver un SaaS multi-tenant real con clientes que pagan.
Stack de referencia: Next.js + Supabase/Postgres + Tailwind + Vercel. Repo:
`lean19r-cell/the-seedskill`. Idioma del contenido de la skill: **inglés** (los ejemplos
llevan identificadores en español del dominio, ej. `anulada`, `turno_caja_id`); las
explicaciones al usuario y estas notas: **español**.

**La fuente de las lecciones es el proyecto `lean19r-cell/prp-gestion-webapp`** (sistema
de gestión multi-vertical: Restaurante, Alquileres/Ventas por zona, Comercio). Cada vez
que en el webapp se arregla un bug, se hace una auditoría o pasa algo que costó tiempo, la
idea es "asimilarlo" acá: extraer el **mecanismo** (no el parche puntual), decir dónde más
mirar, y dejar el ejemplo roto/arreglado.

## Estructura

| Archivo | Para qué |
|---|---|
| `SKILL.md` | Entrada: descripción (dispara la skill), tabla "cuándo leer qué", las 6 reglas que no se doblan, autorización de push/merge, racionalizaciones comunes |
| `references/debugging-and-verification.md` | Depuración por causa raíz + gate de verificación (adaptado de obra/superpowers, MIT) |
| `references/testing-discipline.md` | Red→green→refactor; tests que nombran el bug; sección "Tests that looked fine and weren't" (de rondas de revisión reales) |
| `references/design-plan-execute.md`, `references/review-and-agents.md` | Diseño→plan→ejecución; revisión y agentes en paralelo (adaptado de superpowers) |
| `references/nextjs-supabase-gotchas.md` | Bugs concretos del stack, numerados (#1–#16), cada uno con mecanismo + roto/arreglado + "dónde mirar" |
| `references/money-and-ledger-integrity.md` | Plata/stock/caja en RPCs de Postgres (8 secciones + checklist #8) |
| `references/multitenant-architecture.md` | Un codebase para negocios distintos sin forkear |
| `references/responsive-design-method.md` | Mobile: mockup-first, patrones repetidos |

## Cómo agregar una lección (convención ya usada)

1. **Leer el código real, no solo el mensaje de commit.** Los snippets SQL/TS que se citan
   deben coincidir con la migración o el archivo real (en la sesión de 2026-09-30 un
   snippet escrito de memoria resultó no ser fiel — `listar_productos_venta()` lanza
   excepción, no devuelve vacío — y se corrigió contra la migración antes de commitear).
2. Estructura de cada entrada: **mecanismo** → **cómo se ve desde afuera** → **roto/arreglado**
   → **dónde más mirar** (un grep concreto). Después de arreglar una instancia, siempre
   buscar la misma forma en el resto del código (regla 6 de `SKILL.md`).
3. Ser honesto con la procedencia: decir si algo lo reportó un usuario en producción, o lo
   encontró una auditoría/revisión y se reprodujo en base local. No afirmar más de lo que
   dicen los commits.
4. Colocar la lección donde la va a buscar quien esté en esa situación (tabla de `SKILL.md`).
   Si es un patrón de plata/stock → `money-and-ledger-integrity.md`; si es del stack →
   gotchas; si es sobre tests → `testing-discipline.md`.
5. Actualizar el registro de abajo (fecha, hash fuente, qué se agregó).

## Registro de asimilaciones

| Fecha | Fuente en `prp-gestion-webapp` | Qué se agregó |
|---|---|---|
| 2026-09-14 | Estado inicial del webapp | Commit inicial: skill `multitenant-saas-playbook` (gotchas #1–#7, arquitectura multi-tenant, responsive) |
| 2026-09-22 | Sprints del webapp hasta ~2026-09-21 (commit `0b000b1` de esta skill) | 5 lecciones: gotchas #8–#12 (sticky, `for update` en batch global, tolerancia de timestamps, timezone de offset fijo, `overflow-x`) + regresión de rol en RLS de `clientes` (en `multitenant-architecture.md`) |
| 2026-09-25 | — (no es del webapp) | Renombre a `the-seedskill` + 4 referencias de disciplina general adaptadas de obra/superpowers |
| **2026-09-30** | **`master` hasta `751894c`** (PR #73 auditoría Tanda 1; PR #72 vertical Comercio, fixes de la Tarea 25; PR #71 responsive de Alquileres) | Ver detalle abajo |

### Detalle de la asimilación del 2026-09-30

Mapa lección → commits fuente del webapp (para poder releer el diff original):

| Dónde quedó | Lección | Commits |
|---|---|---|
| `money-and-ledger-integrity.md` #1 | Flag de anulación debe respetarlo cada lector **y** la entrada compensatoria; reintegro sigue la forma de pago | `6bcfd92`, `fa8abd6`, `70a3af0`, `93c2da4`, `4ed77d1`, `5a70e03` |
| idem #2 | Lookup sin `for update` (RES-07) + inversión de orden de locks (deadlock `40P01`) | `008765e`, `8d9fb78` |
| idem #3 | "Eliminar" + `on delete cascade` borra historia; `activo` solo en pickers de operación nueva | `8dfada1`, `5bc6206`, `d6169b2` |
| idem #4 | `NaN` pasa `>= 0`; redondear antes de validar; no coaccionar parámetros; `array_agg(distinct)` y NULL | `70489bd`, `bf77d60`, `5519f63` |
| idem #5 | Regla de negocio en **toda** vía de escritura + default seguro; cobros en efectivo dentro del arqueo | `6f2b2d4`, `02821cf`, `06d20a1`, `656166f`, `16a46df` |
| idem #6 | Campo con unidad equivocada (ALQ-01): default que reproduce el bug, lectores que cambian de significado | `c511793`, `73a5f33`, `afa8e3d` |
| idem #7 | `create or replace` desde base obsoleta (la auditoría citaba `cerrar_caja` previo a Comercio) | plan `docs/superpowers/plans/2026-09-28-auditoria-tanda1.md` (Tarea 6), `8d9fb78` |
| idem #8 | Checklist "antes de dar por cerrado un fix de plata" | síntesis de todas las rondas 2/3 |
| `nextjs-supabase-gotchas.md` #13 | Gate convertido a `{ok:false}` que se ignora (SEG-01) | `be0870a`, `61672f9`, `53c35a8` |
| idem #14 | RLS vacía joins/embeds del rol menos privilegiado | `6ba490a`, `ab0ed69`, `54d410c`, `e634d11` |
| idem #15 | RLS filtra filas, no columnas → grants por columna | `dd893d6`, `0dbb1de`, `6f2b2d4` |
| idem #16 | Prop nueva en componente compartido: obligatoria, no opcional | `70a3af0`, `93c2da4` |
| idem #7 (infra) | DB local compartida entre worktrees, choque de numeración, Docker Desktop apagado | `CONTEXTO.md` del webapp |
| `testing-discipline.md` | Test que re-implementa la fórmula; extracción que no movió lo testeado; test de concurrencia que no aislaba el lock; probar rojo contra la definición vieja; testear cada objeto de la migración | `73a5f33`, `61672f9`, `53c35a8`, `519d925`, `5a70e03` |
| `review-and-agents.md` | Revisión de rama completa por dimensión; ronda 2 de la misma clase = clase sin arreglar; verificar también el fix de la revisión | `8d9fb78`, `dd893d6`…`16a46df`, `53c35a8` |
| `responsive-design-method.md` | Pase responsive de Alquileres: formas chicas y repetidas | `52a5197` (PR #71) |
| `multitenant-architecture.md` | Extender una función compartida entre verticales, probar no-regresión "por construcción" | `656166f` |
| `SKILL.md` | Fila nueva en la tabla, descripción, migraciones ≠ merge, 5 racionalizaciones nuevas, puntero al checklist | — |

**Limitación de esta sesión:** las lecciones se extrajeron **leyendo** commits, migraciones,
tests y el plan de la auditoría. No se corrió la suite del webapp ni se reprodujo nada de
nuevo; la evidencia de "falla sin el fix" es la que dejaron los propios commits.

### Qué NO se asimiló a propósito

- Detalles de negocio del vertical Comercio (límites de crédito, garantías, transferencias
  de stock) — solo el patrón general que enseñan.
- ALQ-04 (llevar `precio_total`/`cantidad_cuotas` al modelo de `contratos`): sigue
  pendiente en el webapp; cuando se haga, revisar si cambia la lección #6 de
  `money-and-ledger-integrity.md`.
- El vertical Agro (diseñado, 0 líneas de código): nada que asimilar todavía.

## Próxima vez que haya que asimilar

1. En el webapp: `git fetch origin && git log --format='%h %ad %s' --date=short 751894c..origin/master`
   (o el hash que figure como último en la tabla de arriba). Priorizar commits `fix(...)` y
   los mensajes con "ronda 2"/"revisión final" — ahí están las lecciones.
2. La auditoría `informe-auditoria-2026-09-27.md` vive **fuera** de los repos
   (`D:\` en la máquina de L); el plan de la Tanda 1 habla de "9 hallazgos", así que
   probablemente haya Tandas 2+. Si aterrizan, asimilarlas.
3. **`CONTEXTO.md` del webapp**: estaba desactualizado (2026-09-21) y se puso al día el
   2026-09-30 (commit `f28c744` en la rama `claude/great-fermat-93hop7` del webapp, **todavía
   no mergeado a `master`**): agrega el vertical Comercio, la Tanda 1 de la auditoría y una
   advertencia de que las migraciones `126`–`163` no tienen confirmación de estar aplicadas
   a producción. Ahí también quedó anotado el último hash asimilado por esta skill
   (`751894c`). Mantener ambos registros en sincronía.

## Reglas de trabajo en este repo

- Rama de desarrollo de la sesión: `claude/great-fermat-93hop7`. No pushear a `main` ni
  abrir PR sin autorización explícita puntual de L (mismo criterio que el webapp).
- La skill es autocontenida: no depender de que `superpowers` esté instalado.
- Preferir borrar líneas obsoletas antes que dejarlas "por las dudas".
