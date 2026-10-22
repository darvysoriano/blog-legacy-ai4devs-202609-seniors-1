# Auditoría de las reglas de proceso de CLAUDE.md frente al historial de git

- **Fecha:** 2026-10-22
- **Rama auditada:** `auditoria-DS` (apunta al mismo commit que `main`)
- **Alcance:** las 39 reglas de proceso de `CLAUDE.md`, numeradas como en el resumen previo. Las seis reglas de negocio del dominio quedan fuera.

## Qué historial hay realmente

| Comando | Resultado |
|---|---|
| `git log -n 15` | Devuelve **un solo commit**: `25f9d6c Initial commit` (2026-09-25), con 91 archivos y 13 517 líneas añadidas. |
| `git branch --contains 25f9d6c` | Está en `main` y en `auditoria-DS`. |
| `git log --all -n 15` | Aparecen otros 57 commits, de las ramas `upstream/*` y `s5/start`. |
| `git merge-base HEAD upstream/s5/start` | No hay ancestro común: esas ramas son el historial de **otro proyecto (FlowSync)** y no sirven como evidencia sobre este código. |
| `gh pr list -R darvysoriano/blog-legacy-ai4devs-202609-seniors-1 --state all` | Sin PR. |
| `npm test` (en `backend/`) | Falla con `ERR_MODULE_NOT_FOUND: @poppinss/ts-exec` porque no hay `node_modules`. La suite no se puede ejecutar. |
| `git status` | Hay cambios sin commitear en `prompts.md`. No se han tocado. |

Por eso, todas las reglas que dependen del **historial de trabajo** (orden de los commits, TDD, refactor por pasos) solo se pueden contrastar contra un único commit de importación. Las reglas que dependen del **código** se contrastan contra el estado actual del árbol.

### Recuento de veredictos

| Veredicto | Reglas |
|---|---|
| se cumple | 7 (R8, R9, R16, R23, R24, R31, R33) |
| no se cumple | 20 |
| no se puede comprobar con la información disponible | 12 |

## Parte A: Tabla de Auditoría

Cada regla lleva su número (R1–R39) al principio de la primera columna para poder citarla en la Parte B.

| Qué pide la regla | Qué se revisó para comprobarla | Veredicto | Evidencia | Qué harías con ella |
|---|---|---|---|---|
| **R1.** Si la petición no detalla el proceso, se sigue CLAUDE.md sin preguntar. | `git log -n 15`; contenido de `prompts.md` | no se puede comprobar con la información disponible | El repo no guarda las sesiones del agente; solo hay un commit de importación. | dejarla |
| **R2.** El contexto se reúne antes de escribir código, sin esperar a que venga en el prompt. | `git log -n 15` | no se puede comprobar con la información disponible | Leer archivos no deja rastro en git y no hay transcripciones de sesión. | dejarla |
| **R3.** Antes de tocar un endpoint se lee `backend/start/routes.ts` entero. | `git log -n 15` | no se puede comprobar con la información disponible | Ningún commit toca un endpoint después de la importación inicial. | dejarla |
| **R4.** El esquema y los datos se toman de las migraciones y de la semilla. | `backend/database/migrations/`, `backend/database/seeders/blog_seeder.ts` | no se puede comprobar con la información disponible | Los archivos existen, pero no hay trabajo posterior en el que verificar que se consultaron. | dejarla |
| **R5.** El código nuevo imita la forma del código de autenticación (controlador, validador y transformer). | `backend/start/routes.ts` frente a `backend/app/controllers/*.ts` | no se cumple | `routes.ts:38-393` usa closures con SQL y no se parece en nada al patrón de `access_tokens_controller.ts`. | dejarla |
| **R6.** El esquema, la validación y el formato de respuesta se deducen del código, no se suponen. | `git log -n 15` | no se puede comprobar con la información disponible | Es una conducta del autor y no deja huella en el diff. | dejarla |
| **R7.** Solo se modifica el recurso o los endpoints que nombra la tarea. | `git show --stat 25f9d6c` | no se puede comprobar con la información disponible | El único commit importa 91 archivos y no está asociado a ninguna tarea. | dejarla |
| **R8.** La API usa el puerto 3401 y la web el 5401, nunca 3333 ni 5173. | `backend/.env.example`, `frontend/vite.config.js`, `grep -rn "3333\|5173"` | se cumple | `PORT=3401`, y Vite usa `port: 5401` con `strictPort`; 5173 solo aparece en comentarios (`.env.example:18`). | dejarla |
| **R9.** Si se crea la capa de repositorio, el alias `#repositories/*` se añade en el mismo cambio. | `ls backend/app`, `backend/package.json` → `imports` | se cumple | No existe `app/repositories` ni el alias: la condición no se ha activado. | dejarla |
| **R10.** Cada endpoint sigue el flujo ruta → controlador → validador → servicio → repositorio → modelo → transformer. | `ls backend/app` | no se cumple | No existen `app/services` ni `app/repositories`; los 9 endpoints del blog viven en `routes.ts`. | convertirla en una comprobación automática |
| **R11.** La ruta solo declara verbo, ruta y `[Controlador, 'metodo']`. | `backend/start/routes.ts` | no se cumple | `routes.ts:38`, `93`, `109`, `180`, `240`, `251`, `270`, `295`, `339` y `366` declaran closures con lógica. | convertirla en una comprobación automática |
| **R12.** El controlador es fino: valida, delega y devuelve. | `ls backend/app/controllers` | no se cumple | Solo existen los controladores de autenticación; no hay ningún controlador del blog. | dejarla |
| **R13.** Toda la validación de entrada se hace con validadores VineJS. | `backend/start/routes.ts`, `ls backend/app/validators` | no se cumple | `routes.ts:310-321` y `383-386` validan con `if`; solo existe `validators/user.ts`. | dejarla |
| **R14.** El servicio es el único sitio que muta estado. | `grep -n "insert\|update\|delete" backend/start/routes.ts` | no se cumple | `routes.ts:324`, `359` y `388` hacen escrituras directamente desde la ruta. | convertirla en una comprobación automática |
| **R15.** No puede haber `db.rawQuery` fuera del repositorio. | `grep -c "db.rawQuery" backend/start/routes.ts` | no se cumple | Hay 27 llamadas a `db.rawQuery` en `routes.ts`. | convertirla en una comprobación automática |
| **R16.** Los modelos Lucid nunca se devuelven al cliente. | `backend/app/models/user.ts`, `backend/app/controllers/profile_controller.ts` | se cumple | El único modelo (`User`) sale siempre a través de `UserTransformer`; el blog no tiene modelos. | dejarla |
| **R17.** Toda respuesta se construye con un transformer. | `backend/start/routes.ts`, `ls backend/app/transformers` | no se cumple | `routes.ts:82-87`, `166-174` y `289` devuelven filas de `select *`; solo existe `user_transformer.ts`. | convertirla en una comprobación automática |
| **R18.** Las respuestas públicas de un post no exponen correos ni campos internos. | `backend/start/routes.ts`, objetivo `fuga` del `Makefile` | no se cumple | `routes.ts:111` hace `select *`, que incluye `autor_email`; el propio `make fuga` lo demuestra. | convertirla en una comprobación automática |
| **R19.** Los errores se centralizan en `app/exceptions/handler.ts`. | `backend/app/exceptions/handler.ts`, `backend/start/routes.ts` | no se cumple | `handler.ts` está sin personalizar; las rutas responden 400, 422 y 200 `{ok:false}` a mano (`routes.ts:310-321`). | dejarla |
| **R20.** Los archivos se generan con `node ace make:*`, no a mano. | `git show --stat 25f9d6c`, `ls backend/app` | no se puede comprobar con la información disponible | No hay archivos del blog generados y un stub de ace no se distingue de uno escrito a mano. | borrarla |
| **R21.** TDD estricto: la prueba que falla se escribe antes que el código. | `git log -n 15 -- backend/tests` | no se puede comprobar con la información disponible | El test y el código llegaron en el mismo commit `25f9d6c`, así que no hay orden observable. | dejarla |
| **R22.** Los tests siguen AAA con comentarios `Arrange`, `Act` y `Assert`. | `grep -nE "// (Arrange\|Act\|Assert)" backend/tests/functional/comentarios.spec.ts` | no se cumple | `comentarios.spec.ts:15-43` no tiene ninguno de los tres comentarios. | convertirla en una comprobación automática |
| **R23.** El comportamiento de un endpoint se prueba con tests funcionales y la lógica pura con tests unitarios. | `ls backend/tests` | se cumple | El único test es de endpoint y está en `tests/functional/`; todavía no hay lógica pura extraída. | dejarla |
| **R24.** Cada test se aísla con `withGlobalTransaction` y sin mocks de base de datos. | `backend/tests/functional/comentarios.spec.ts` | se cumple | `comentarios.spec.ts:13` usa `group.each.setup(() => testUtils.db().withGlobalTransaction())`. | dejarla |
| **R25.** Antes de refactorizar se escriben tests de caracterización en verde. | `git log -n 15` | no se puede comprobar con la información disponible | El historial no contiene ningún refactor. | dejarla |
| **R26.** La suite se mantiene en verde en cada paso del refactor. | `git log -n 15`, `npm test` | no se puede comprobar con la información disponible | Hay un solo commit y hoy la suite no arranca (`ERR_MODULE_NOT_FOUND`). | convertirla en una comprobación automática |
| **R27.** Corregir un bug es una tarea aparte, con su propio test. | `git log -n 15 --grep="^fix"` | no se puede comprobar con la información disponible | La rama no tiene ningún commit `fix:`. | dejarla |
| **R28.** Cada test afirma el contrato completo de la respuesta. | `backend/tests/functional/comentarios.spec.ts` | no se cumple | `comentarios.spec.ts:42` solo comprueba `autor_nombre`, de los 8 campos que devuelve. | dejarla |
| **R29.** Cada test incluye aserciones negativas sobre los campos que no deben salir. | `grep -n "notProperty" backend/tests -r` | no se cumple | No hay ningún `notProperty`, aunque la respuesta incluye `autor_email`. | convertirla en una comprobación automática |
| **R30.** Se afirman los tipos donde importan y no se fijan valores volátiles. | `backend/tests/functional/comentarios.spec.ts` | no se cumple | No hay ninguna aserción de tipo (`estado`, `id`, `creado_en`). | dejarla |
| **R31.** Un test que solo comprueba el código de estado no es válido. | `backend/tests/functional/comentarios.spec.ts` | se cumple | `comentarios.spec.ts:41-42` añade un `assert.equal` al `assertStatus`; cumple, pero por la mínima. | borrarla |
| **R32.** Cada endpoint cubre feliz, entrada inválida, 404 y frontera de la regla. | `ls backend/tests/functional`, `comentarios.spec.ts:5-11` | no se cumple | Hay 1 test para 9 endpoints; el propio comentario del archivo reconoce que no cubre nada más. | dejarla |
| **R33.** La cobertura se mide con c8 y el umbral sube según crece la suite. | `.claude/hooks/guard-tests.mjs:32`, `backend/.c8rc.json` | se cumple | `COBERTURA_MINIMA = 0`, coherente con una suite de un solo test. | convertirla en una comprobación automática |
| **R34.** Nunca se hace commit en `main`: primero se crea una rama `refactor/` o `feat/`. | `git branch --contains 25f9d6c` | no se cumple | `25f9d6c` está directamente en `main`; la rama `auditoria-DS` no sigue el prefijo pedido. | convertirla en una comprobación automática |
| **R35.** Los mensajes de commit siguen el formato convencional. | `git log -n 15 --format=%s` | no se cumple | El mensaje del único commit es `Initial commit`, sin tipo convencional. | convertirla en una comprobación automática |
| **R36.** Solo se añaden al commit los archivos tocados, nunca `git add -A` ni `git add .`. | `git show --stat 25f9d6c` | no se puede comprobar con la información disponible | Git no registra qué comando de `add` se usó; el commit inicial añade los 91 archivos. | dejarla |
| **R37.** La suite tiene que estar en verde antes de cada commit, y el hook lo garantiza incluso con `--no-verify`. | `.claude/settings.json`, `.claude/hooks/guard-tests.mjs`, `npm test` | no se cumple | El hook solo intercepta la herramienta `Bash` (`settings.json:5`), su expresión no detecta `git -C . commit` (`guard-tests.mjs:42`) y hoy la suite no arranca. | convertirla en una comprobación automática |
| **R38.** El último paso es un PR abierto con `gh pr create`. | `gh pr list -R darvysoriano/blog-legacy-ai4devs-202609-seniors-1 --state all` | no se cumple | No hay ningún PR; además, el remoto `upstream` apunta a una URL rota que termina en `.gi`. | convertirla en una comprobación automática |
| **R39.** La Definition of Done exige a la vez test primero, capas, contrato, matriz, suite en verde y PR. | Comprobación agregada de las reglas 10-38 | no se cumple | Fallan las capas (10-17), el contrato (28-30), la matriz (32), la suite (26) y el PR (38). | dejarla |

## Parte B: Reflexión y Conclusiones

### 1. La regla en la que más me equivocaste respecto a la expectativa inicial

La **regla 37**: suite en verde antes de cada commit, aplicada por el hook.

**Expectativa inicial.** En el resumen previo la redacté como un hecho: *"un hook lo garantiza y bloquea incluso `--no-verify`"*. CLAUDE.md la presenta así: *"Lo aplica el hook PreToolUse… `git commit --no-verify` tampoco pasa"*. Era la única regla del documento que prometía cumplimiento automático y no solo buena conducta, así que esperaba encontrarla en verde.

**Lo que muestra la evidencia.** La garantía no se sostiene por tres motivos independientes:

1. **El hook solo intercepta una herramienta.** `.claude/settings.json:5` declara `"matcher": "Bash"`. En este entorno también existe la herramienta `PowerShell`, y un `git commit` lanzado desde ella no pasa por el hook. Un commit desde la terminal del desarrollador o desde el IDE tampoco: el hook es de Claude Code, no de git.
2. **La expresión regular se puede esquivar.** `guard-tests.mjs:42` comprueba `/\bgit\s+commit\b/`. Esa expresión no detecta `git -C . commit`, `git -c user.name=x commit` ni un alias como `git ci`; en todos esos casos el hook sale con código 0 y deja pasar el comando.
3. **Hoy la suite no puede ponerse verde.** `npm test` falla con `ERR_MODULE_NOT_FOUND: @poppinss/ts-exec` porque no hay `node_modules`. En el estado actual, el hook bloquearía *todos* los commits que sí intercepta, sin importar el código. Eso empuja justo a lo que el propio comentario del hook quiere evitar (`guard-tests.mjs:23-26`): que alguien lo apague.

Además, el único commit del historial (`25f9d6c`) no deja rastro de haber pasado por el hook, así que tampoco hay evidencia de que la regla se haya aplicado nunca.

**Conclusión.** Pasé de "se cumple por construcción" a "no se cumple". La regla es correcta, pero su mecanismo de aplicación es local, depende de una sola herramienta y se puede esquivar. Debería convertirse en una comprobación automática de verdad: un job de CI que ejecute `npm ci && npm test` en cada push y en cada PR, con protección de rama que lo exija para hacer merge.

### 2. Una regla que no se puede comprobar con la información disponible y qué tendría que cambiar para poder verificarla

La **regla 21**: TDD estricto, la prueba que falla se escribe primero.

**Por qué no se puede comprobar.** TDD es una propiedad del **orden** en que se hace el trabajo, y el historial no conserva ese orden:

- `git log -n 15` devuelve un solo commit, `25f9d6c`.
- Ese commit introduce a la vez `backend/tests/functional/comentarios.spec.ts` y el endpoint `POST /posts/:slug/comentarios` que prueba (`routes.ts:295-334`).
- `git log -n 15 -- backend/tests` devuelve ese mismo commit. No existe un estado intermedio en el que el test exista y el código todavía no.
- `npm test` no arranca, así que ni siquiera se puede reproducir el "rojo" volviendo atrás.

Con la información disponible es imposible distinguir "se escribió el test primero" de "se escribió el test después para que la suite no estuviera vacía".

**Qué tendría que cambiar para poder verificarla.**

1. **Hacer explícito el ciclo en los commits.** Pedir un commit `test:` con la prueba en rojo antes del commit `feat:`, `fix:` o `refactor:` que la pone en verde. Así el orden queda en `git log` y se puede auditar con `git log --format=%s`.
2. **Que CI deje constancia del rojo.** Un job que, para cada commit `test:`, ejecute la suite y espere que falle exactamente el test nuevo. El commit `feat:` siguiente debe pasar. Esto convierte una declaración de conducta en un dato.
3. **Dejar el entorno reproducible.** Con `npm ci` en CI y el lockfile versionado, cualquier commit se puede volver a ejecutar para comprobar su estado. Hoy ni siquiera el estado actual se puede ejecutar.
4. **Prohibir el squash de los PR que mezclen `test:` con implementación**, o conservar los commits individuales en el merge. Si no, el orden se pierde al integrar.

Ese ajuste choca a medias con la regla 38: si se usa squash-merge para tener un PR limpio, el squash destruye justo la evidencia del TDD. Hay que elegir merge commit o rebase-merge.

### 3. La regla que eliminarías y por qué

La **regla 20**: generar los archivos con `node ace make:controller`, `make:validator`, `make:service`, etc., y no escribirlos a mano.

**Por qué la eliminaría.**

1. **No se puede verificar.** Un archivo generado por `node ace make:controller` y otro escrito a mano con el mismo contenido son idénticos en el árbol y en el diff. `git show --stat 25f9d6c` no distingue cómo nació cada archivo, y ninguna herramienta del repo lo registra. En la tabla aparece como "no se puede comprobar", y no hay ningún cambio razonable que lo haga comprobable: tendría que instrumentarse el CLI.
2. **Exige el medio, no el resultado.** Lo que de verdad importa (que el controlador sea fino, que la validación esté en VineJS, que el acceso a datos esté en el repositorio, que la respuesta pase por un transformer) ya lo piden las reglas 10 a 17. Esas reglas sí se pueden verificar con grep, lint y tests. Si el archivo cumple la forma de las reglas 10 a 17, da igual si lo escribió ace o una persona. Si no la cumple, generarlo con ace no lo arregla: los stubs vienen vacíos.
3. **Su parte útil ya está en otra regla.** La intención real, "no copies patrones de AdonisJS 5 ni de memoria", ya la cubre la regla 5: imitar el código de autenticación, que es el ejemplo vivo del framework instalado. La regla 20 añade una restricción de herramienta que no aporta calidad comprobable.
4. **Una regla que nadie puede comprobar le resta credibilidad al resto.** Un documento que mezcla reglas comprobables con reglas que no lo son invita a tratarlo todo como sugerencia.

Su contenido útil puede pasar a la regla 5 como nota: *"imita la forma del código de autenticación; los generadores de `node ace` son la forma más rápida de obtener esa estructura"*. Así deja de ser una obligación.

**Nota.** En la tabla también marqué para borrar la **regla 31** ("un solo `assertStatus` no es un test"). Es redundante con la regla 28: un test que afirma el contrato completo nunca se queda en el código de estado. De las dos, la regla 20 es la que menos aporta, porque además de redundante es imposible de verificar.
