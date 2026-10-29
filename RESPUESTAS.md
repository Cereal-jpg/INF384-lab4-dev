# Respuestas · Laboratorio 4
Alumno: 20220749
Presupuesto: 11 min  Métrica DORA: time to restore

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: `smoke`, etapa deploy (tenía `needs: build-push`, al mismo nivel que `deploy`). Arrastra a `e2e` y `load`, que dependen de `smoke`.
- Problema que causa en el pipeline: `smoke` arranca en paralelo con `deploy`, así que `smoke`, `e2e` y `load` pueden probar la versión anterior de la Lambda (o una función todavía sin Function URL). Deja pasar una versión rota: el verde de las pruebas post-deploy no corresponde al commit que se desplegó.
- Link al commit: https://github.com/Cereal-jpg/INF384-lab4-dev/commit/fa86a194bf5571329ef8f8a98f06a5de2230afec
- Corrección: mover de etapa (`smoke` pasa a `needs: deploy`, etapa post-deploy).
- Evidencia: run https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40957126021 (antes de la corrección). Horas de inicio de los jobs, de la API de GitHub:
  ```
  build-push  2026-10-29T21:22:11Z  2026-10-29T21:22:55Z
  smoke       2026-10-29T21:22:57Z  2026-10-29T21:23:30Z
  deploy      2026-10-29T21:22:58Z  2026-10-29T21:23:24Z
  ```
  `smoke` empezó 1 s antes que `deploy`. Su step `URL` estuvo de 21:23:01 a 21:23:26 reintentando hasta que `deploy` creó la Function URL (el log de `deploy` la imprime a las 21:23:22). Si `smoke` hubiera encontrado una URL ya creada, habría probado la versión anterior.

  Después de la corrección, run https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959109982:
  ```
  deploy      2026-10-29T21:36:58Z  2026-10-29T21:37:13Z
  smoke       2026-10-29T21:37:15Z  2026-10-29T21:37:25Z
  ```
  `smoke` arranca 2 s después de que `deploy` termina.
### Defecto 2
- Job y etapa: `smoke`, etapa deploy en el pipeline original (post-deploy tras corregir el defecto 1).
- Problema que causa en el pipeline: `test/smoke/smoke.sh` solo compara `/version` con el commit esperado si recibe `EXPECTED_COMMIT`, y `ci.yml` no se lo pasaba. `smoke` daba verde con cualquier versión que respondiera `/salud`, incluida la anterior o la de otro run (por ejemplo, un PR que despliega en la misma función).
- Link al commit: https://github.com/Cereal-jpg/INF384-lab4-dev/commit/8ed36a26d97160400ef067955f2ecf4fbf12c909
- Corrección: ajustar configuración (`EXPECTED_COMMIT: ${{ github.sha }}` en el env del step `Smoke`).
- Evidencia: log de `smoke` del run https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40957126021. Solo aparecen las líneas `salud:` y `saldo:`, no la línea `commit:` que imprime el bloque de `EXPECTED_COMMIT`:
  ```
  2026-10-29T21:23:27.7273473Z salud: {"ok":true,"version":"1.0.0","commit":"49ba49ebee84f4c9f8b6943031595970311a3bb6"}
  2026-10-29T21:23:27.9294522Z saldo: {"usuario":"ana","saldo":1000}
  ```
  Después de la corrección, log de `smoke` del run https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959109982. Aparece la verificación del commit esperado:
  ```
  2026-10-29T21:37:22.9065927Z salud: {"ok":true,"version":"1.0.0","commit":"03d2e7ebc5c95ef5721c1441c0454cf71bb27f4f"}
  2026-10-29T21:37:23.0287303Z commit: 03d2e7ebc5c95ef5721c1441c0454cf71bb27f4f
  2026-10-29T21:37:23.1805504Z saldo: {"usuario":"ana","saldo":1000}
  ```
### Defecto 3
- Job y etapa: nivel de workflow. Afecta a `deploy` (etapa deploy) y a `smoke`, `e2e` y `load` (etapa post-deploy).
- Problema que causa en el pipeline: los PR y `main` despliegan en la misma función, y la única `concurrency` estaba en el job `deploy`. Esa concurrency serializa los despliegues, pero no protege las pruebas: mientras el run A corre `smoke`/`e2e`/`load`, el run B puede desplegar su imagen. A termina en verde o en rojo según el código de B. Es un falso verde o un falso rojo.
- Link al commit: https://github.com/Cereal-jpg/INF384-lab4-dev/commit/8e1d5dad88c44ee2de6f9513cffc465cd02804c1
- Corrección: ajustar configuración (`concurrency: { group: ci-${{ github.repository }}, cancel-in-progress: false }` a nivel de workflow).
- Evidencia: `grep -n concurrency .github/workflows/ci.yml` antes de la corrección:
  ```
  175:    concurrency:
  ```
  La única concurrency es la del job `deploy`. Ni el workflow ni los jobs post-deploy la tienen.

  Efecto de la corrección: el run del PR https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959211010 (creado a las 21:36:26Z) quedó en estado `pending`, con 0 jobs iniciados, mientras el run de `main` https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959109982 seguía `in_progress` (en `load`). Salida de `gh run list` a las 21:39:03Z:
  ```
  pending      Defecto de la aplicación: validarMonto acepta monto cero  ci  defecto/monto-cero  pull_request  40959211010  2m38s
  in_progress  docs: RESPUESTAS Parte 1                                  ci  main                push          40959109982  3m20s
  ```
### Defecto 4
- Job y etapa: `sca`, etapa pre-build.
- Problema que causa en el pipeline: `--include-dev-deps` hace que Trivy escanee el árbol completo, con `@playwright/test`, `c8`, `esbuild` y sus dependencias. El `Dockerfile` hace `npm prune --omit=dev`, así que esas dependencias no llegan a la imagen. Con `--exit-code 1`, una vulnerabilidad HIGH o CRITICAL en una herramienta de desarrollo detiene el pipeline aunque no se despliegue. Además escanea unas 3 veces más paquetes de los que corresponden.
- Link al commit: https://github.com/Cereal-jpg/INF384-lab4-dev/commit/01864e9ec16edb1b47d2e3381874e4e712ac4398
- Corrección: ajustar configuración (se quita `--include-dev-deps`).
- Evidencia (salida local):
  ```
  $ npm ls --all | wc -l
  146
  $ npm ls --omit=dev --all | wc -l
  50
  ```
  Las 50 líneas son solo `better-sqlite3` y sus transitivas, que es lo que queda en la imagen después de `npm prune --omit=dev`. Con la versión actual del lock, Trivy 0.76.1 da 0 HIGH/CRITICAL con y sin el flag (exit=0 en ambos casos), así que hoy el defecto está latente: el escaneo cubre 96 líneas de dependencias que no se despliegan.
### Defecto 5
- Job y etapa: `integration`, etapa pre-build (`test/integration/transferir.test.js` y `DB_PATH: /tmp/wallet.db` en `ci.yml`).
- Problema que causa en el pipeline: los tests asumen los saldos semilla (por ejemplo, ana con 1000 y 900 después de transferir), pero escriben en una base de ruta fija que persiste entre corridas. Solo pasan sobre una base recién creada: si se repiten sobre la misma base (corrida local, re-run o runner reutilizado), fallan aunque `src/` no tenga defectos. Es un falso rojo.
- Link al commit: https://github.com/Cereal-jpg/INF384-lab4-dev/commit/f988a8f3bd2680204c6f74d97ca88a2ae6a387e4
- Corrección: modificar tests (cada corrida usa una base nueva en `fs.mkdtempSync(path.join(os.tmpdir(), 'wallet-'))`).
- Evidencia (salida local con Node 20, `DB_PATH=/tmp/wallet.db npm run test:integration` dos veces seguidas).
  Antes:
  ```
  == corrida 1
  # tests 12
  # pass 12
  # fail 0
  == corrida 2
  not ok 1 - ana transfiere 100 a beto
  not ok 2 - el saldo de ana refleja el debito
  not ok 3 - el saldo de beto refleja el credito
  not ok 4 - el historial de ana registra la transferencia
  not ok 6 - una transferencia rechazada no cambia saldos
  not ok 7 - diego transfiere todo su saldo a elena
  # tests 12
  # pass 6
  # fail 6
  ```
  Después:
  ```
  == corrida 1
  # tests 12
  # pass 12
  # fail 0
  == corrida 2
  # tests 12
  # pass 12
  # fail 0
  ```

## Parte 2 · Implementación
- Link al run en verde en main: https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959109982 (commit `03d2e7e`, que tiene en su historial los cinco commits de la Parte 1)
- Link al pull request: https://github.com/Cereal-jpg/INF384-lab4-dev/pull/1
- Defecto de la aplicación introducido (archivo y descripción): `src/util.js`, función `validarMonto`. La condición `monto <= 0` pasa a `monto < 0`, así que un monto de `0` se acepta como válido. La regla de `/transferir` exige 400 si el monto es menor o igual a cero. Con el defecto, `POST /transferir` con `monto: 0` pasa la validación de negocio y llega a la base, donde el `CHECK (monto > 0)` de `movimientos` lo rechaza. El cliente recibe un 500 (`error interno`) en lugar de un 400 con el mensaje `monto debe ser mayor que cero`.
- Job que debería detenerlo: `unit` (etapa pre-build). Lo detectan los tests `validarMonto rechaza cero` (test/unit/util.test.js) y `transferir con monto invalido es 400` (test/unit/wallet.test.js). Salida local con Node 20:
  ```
  not ok 4 - validarMonto rechaza cero
  not ok 27 - transferir con monto invalido es 400
  # pass 26
  # fail 2
  ```
- Link al run fallido: https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959211010 (`unit` en rojo, por lo que `build-push`, `deploy` y las pruebas post-deploy no se ejecutan). Log de `unit`:
  ```
  2026-10-29T21:49:50.1270268Z not ok 4 - validarMonto rechaza cero
  2026-10-29T21:49:50.1452152Z not ok 27 - transferir con monto invalido es 400
  2026-10-29T21:49:50.1460463Z # pass 26
  2026-10-29T21:49:50.1460639Z # fail 2
  2026-10-29T21:49:50.1765351Z ##[error]Process completed with exit code 1.
  ```

## Parte 3 · Decisión bajo restricción
Link al run medido: https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959109982

| Job | Duración (min:seg) |
|---|---|
| unit | 0:15 |
| integration | 0:13 |
| sast | 0:17 |
| sca | 0:17 |
| build-push | 0:50 |
| deploy | 0:15 |
| smoke | 0:10 |
| e2e | 0:37 |
| load | 12:12 |
| **Total** | **15:06** |

(Duraciones calculadas con `started_at` y `completed_at` de cada job, según la API de GitHub.)

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 53.7 ms |
| p95 | 56.35 ms |
| Máximo | 1.86 s |
| Tasa de error | 0.00 % (0 de 36240) |
| Requests por segundo | 50.26 req/s (36240 requests) |

Conclusión sobre el comportamiento bajo carga: con 30 VUs sostenidos durante 10 min (50 req/s), Monedero Andino no muestra saturación. La latencia es estable y concentrada (mediana 46.72 ms, p90 53.7 ms, p95 56.35 ms; entre p90 y p95 hay menos de 3 ms) y no hubo ningún request fallido (0 de 36240). El máximo de 1.86 s es un valor aislado, unas 33 veces el p95. Es coherente con cold starts de Lambda: cada run despliega una imagen nueva y durante la rampa se crean instancias nuevas, cada una con su base SQLite en `/tmp`. Ese valor afecta la cola extrema, pero no los percentiles. Conclusión: a esta carga la función tiene amplio margen; el riesgo de latencia está en los arranques en frío, no en la carga sostenida.

SLO propuesto (percentil, valor, ventana) y justificación: **p95 de la latencia de las requests a la API < 200 ms**, medido en producción en una **ventana móvil de 28 días**. En el pipeline la ventana es la duración del test de carga (`SLO_P95: 200` en `ci.yml`). Uso el p95 y no el máximo porque el máximo está dominado por cold starts aislados (1.86 s) que no representan la experiencia típica; el p95 sí la representa y fue estable (56.35 ms). Elegí 200 ms y no un valor pegado a lo medido (por ejemplo 60 o 70 ms) porque entre runs varían los cold starts, el estado de la región y la cantidad de instancias. Un umbral de 60 ms haría fallar `load` sin que haya un defecto (falso rojo). 200 ms deja unas 3,5 veces de margen sobre el p95 medido y sigue detectando una regresión real, por ejemplo una consulta que pase de 50 a 250 ms. Además, una billetera con p95 bajo 200 ms se percibe como instantánea.

SLA propuesto y diferencia con el SLO: **el 95 % de las requests responde en menos de 1 s, y disponibilidad mensual del 99,5 %** (respuestas no 5xx), medido por mes calendario. Si no se cumple, el cliente recibe una compensación (crédito o bonificación). El SLO es una meta interna de ingeniería: guía decisiones (por ejemplo, detener un despliegue en `load`) y su incumplimiento no tiene consecuencias contractuales. El SLA es un compromiso contractual con los clientes y su incumplimiento tiene penalidad. Por eso el SLA es más laxo que el SLO (1 s contra 200 ms): la diferencia es el margen que permite al equipo detectar y corregir una degradación cuando se incumple el SLO, antes de llegar a incumplir el contrato. Si ambos tuvieran el mismo valor, cada desvío del SLO ya sería un incumplimiento con penalidad.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Stages de `test/load/transferir.js` de `1m / 10m / 1m` a `30s / 4m / 30s` (carga constante de 4 min, el mínimo permitido) | load | 7:02 (de 12:12 a 5:10) |
| `SLO_P95` de 500 a 200 ms (punto 2; no cambia la duración) | load | 0:00 |

Duración por job en el run con los cambios: unit 0:11, integration 0:12, sast 0:19, sca 0:22, build-push 0:42, deploy 0:17, smoke 0:07, e2e 0:34, load 5:10. Calculadas con `started_at` y `completed_at` de la API de GitHub. Los demás jobs varían unos segundos entre runs; en total, el pipeline baja 7:12, de 15:06 a 7:54.

Total resultante: **7:54 min** (presupuesto: 11 min, margen de 3:06)
Link al run en main con los cambios: https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40961511153

En ese run, `load` evaluó el nuevo SLO y pasó: `✓ 'p(95)<200' p(95)=74.83ms`, con p90 72.22 ms, máximo 425.63 ms, 0.00 % de error (0 de 14330) y 47.70 req/s. El p95 subió de 56 a 75 ms respecto del run anterior y sigue con margen de unas 2,7 veces bajo el SLO. Esto confirma que un SLO pegado a lo medido (por ejemplo 60 ms) habría hecho fallar este run sin que hubiera un defecto.

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada): al bajar la carga constante de 10 a 4 minutos, el pipeline detecta más tarde, o deja de detectar, **una degradación de latencia que crece con el volumen de datos**. Ejemplo concreto: en `src/db.js` se quita el `LIMIT 50` de la consulta de `historial`, o la consulta `WHERE origen = ? OR destino = ?` sobre `movimientos` hace un recorrido completo de la tabla. Cada transferencia de `load` inserta una fila en `movimientos`. En el run medido hubo 18120 iteraciones en 10 min; con 4 min hay unas 2,5 veces menos filas. La consulta se vuelve lenta solo cuando la tabla es grande, así que en 4 min el p95 puede quedar bajo los 200 ms y `load` dar verde. El defecto llega a producción, donde la tabla crece sin límite, y la latencia se degrada de forma gradual hasta incumplir el SLO.

Efecto en **time to restore: sube**. Una degradación gradual no dispara una alerta en el momento del despliegue como sí lo hace un 500. Se detecta tarde, cuando el p95 de producción cruza el SLO, días después y con varios despliegues de por medio. Además cuesta asociarla al cambio que la causó: hay que revisar varios commits, no solo el último. El tiempo desde que empieza el incidente hasta que se restaura el servicio aumenta. El pipeline más corto compensa solo una parte, porque el despliegue del rollback o del fix tarda unos 7 minutos menos (8 min en lugar de 15). Ese ahorro es pequeño frente a horas o días de detección tardía.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón: **se queda**, como bloqueante en pre-build y en cada evento. Costo: 0:17 de los 15:06 del run medido (un 2 %). El presupuesto de 11 min ya se cumple recortando `load`, así que mover o quitar `sast` no es necesario para cumplirlo. Además, `sast` corre en paralelo con `unit` (0:15), `integration` (0:13) y `sca` (0:17), así que en tiempo real solo agrega unos 2 s antes de `build-push`. Efecto en **time to restore: no cambia** en la práctica. Cuando hay que restaurar con un hotfix o un revert, el pipeline tarda lo mismo con o sin `sast`, porque no está en el camino crítico. Si pasara a informativo o a post-deploy, un patrón inseguro (por ejemplo, SQL armado concatenando el usuario de la ruta) podría llegar a producción. Eso genera un incidente de seguridad cuya restauración (contención, parche, revisión de datos) es mucho más larga que 17 s de pipeline.

## Declaración de uso de IA
Herramientas usadas y para qué:
- **Claude Code (modelo Claude Opus 5.5, de Anthropic)**, como asistente en el editor (VS Code). Lo usé para:
  - leer el repositorio (`ci.yml`, `package.json`, `test/`, `src/`, `Dockerfile`) y verificar los cinco defectos del pipeline que había identificado previamente;
  - ejecutar comandos de verificación (tests con Node 20 en contenedor Docker, `npm ls`, Trivy, `gh run list` y la API de GitHub) y obtener la evidencia pegada en este documento;
  - aplicar las correcciones (un commit por defecto), crear la rama con el defecto de la aplicación y abrir el pull request;
  - redactar el texto de este documento (Partes 1 a 3).

  Revisé y aprobé cada cambio, el push y cada decisión (defectos, SLO, SLA, recorte de `load`, decisión sobre `sast`) antes de entregar.
