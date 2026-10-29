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
- Link al run fallido: https://github.com/Cereal-jpg/INF384-lab4-dev/actions/runs/40959211010

## Parte 3 · Decisión bajo restricción
Link al run medido:

| Job | Duración (min:seg) |
|---|---|
| unit | |
| integration | |
| sast | |
| sca | |
| build-push | |
| deploy | |
| smoke | |
| e2e | |
| load | |
| **Total** | |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | |
| p95 | |
| Máximo | |
| Tasa de error | |
| Requests por segundo | |

Conclusión sobre el comportamiento bajo carga:

SLO propuesto (percentil, valor, ventana) y justificación:

SLA propuesto y diferencia con el SLO:

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|

Total resultante: [ ] min
Link al run en main con los cambios:

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:

## Declaración de uso de IA
Herramientas usadas y para qué:
