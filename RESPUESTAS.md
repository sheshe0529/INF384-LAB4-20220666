# Respuestas · Laboratorio 4
Alumno: [20220666]
Presupuesto: [ 8 ] min  Métrica DORA: [ Tasa de fallos en cambios / change failure rate ]

## Parte 1 · Diagnóstico
### Defecto 1
- Job y etapa: smoke - deploy (etapa)
- Problema que causa en el pipeline: el problema encontrado está relacionado con el job smoke, pues arranca en paralelo con deploy. El paso "URL" encuentra la función URL que ya existía y smoke.sh llama a /salud mientras deploy todavía está ejecutando update-function-code, lo que ocasiona que las pruebas smoke validen una versión anterior a la desplegada. Además. como e2e y load dependen de smoke, pueden empezar las pruebas sobre una versión desactualizada, lo que ocasionaría falsos positivos en el despliegue.
- Link al commit: https://github.com/sheshe0529/INF384-LAB4-20220666/commit/e2a45e2b7bbe3f1174aac0996f4706e09e3586a1
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): mover de etapa
- Evidencia (dato observado, con link al run o fragmento de la salida local): https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40960255463
### Defecto 2
- Job y etapa: smoke - deploy (etapa)
- Problema que causa en el pipeline: el problema encontrado está relacionado con el job de smoke también, pues el script de smoke.sh tiene un bloque que compara /version contra EXPECTED_COMMIT, pero ci.yml nunca le pasa esa variable. Se trata de código muerto. 
- Link al commit: https://github.com/sheshe0529/INF384-LAB4-20220666/commit/d386ef58a9f3b735d7b13b7202d3fb22128ed330. Debido a que hice un comentario con mala notación, los demás actions fallaron, pero hice la corrección en este commit (quitar los "//", me confundí). Commit de corrección: https://github.com/sheshe0529/INF384-LAB4-20220666/commit/d76701c8d08760ad3d7d5c1502c9e21788a1b60f
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): Ajustar contiguración
- Evidencia (dato observado, con link al run o fragmento de la salida local): https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40961371388. Debido al problema de antes, falló. Se presenta el run funcionando: https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40964240012
### Defecto 3
- Job y etapa: e2e - post-deploy (etapa)
- Problema que causa en el pipeline: el defecto encontrado se trata de que se utiliza Chromium de forman innecesaria y no controlada para pruebas que no lo requieren, en este caso, pruebas de API. Esto desperdicia tiempo en cada run, además que repite validaciones que cubren unit e integración, lo que ocasiona que la piramide de pruebas quede invertida.
- Link al commit:
  Se tienen dos commits:
  https://github.com/sheshe0529/INF384-LAB4-20220666/commit/ccf0fc83de3483326cb2d9476ee173f94838471e
  https://github.com/sheshe0529/INF384-LAB4-20220666/commit/d771923feda4517b7513a988aca11d95328797a3
  (runs no están en verde por la falla del defect 2 que puse mal un comentario, pero el cambio está bien :(, considerarlo por favor).

- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests):modificar tests y eliminar
- Evidencia (dato observado, con link al run o fragmento de la salida local): https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40963022386 (run no está en verde por la falla del defect 2 que puse mal un comentario, pero el cambio está bien :(, considerarlo por favor).
### Defecto 4
- Job y etapa: sca, etapa pre-build
- Problema que causa en el pipeline: el problema está relacionado con el bloqueo por dependencias de desarrollo que no se despliegan. --include-der-deps hace que Trivy escanee y bloquee por parquetes que nunca llegan a producción. Una CVE HIGH en lob o brace-expansion frenaría un despliegue que no la contiene, lo que sería un falso despliegue. 
- Link al commit: https://github.com/sheshe0529/INF384-LAB4-20220666/commit/98c597d3a41e53e17057841b4de49f6fca62186f
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): ajustar contiguración
- Evidencia (dato observado, con link al run o fragmento de la salida local):

jq '[.packages | to_entries[] | select(.key != "" and .value.dev)] | length' package-lock.json   # 84
jq '[.packages | to_entries[] | select(.key != "")] | length' package-lock.json                   # 122

84 de 122 paquetes (69 %) que escanea sca son de desarrollo y no están en la imagen.

### Defecto 5
- Job y etapa: integration - etapa pre-build
- Problema que causa en el pipeline: El problema encontrado está relacionado con las pruebas de integración, pues no son herméticas. Las 12 pruebas dependen de que /tmp/wallet.db no exista y de correr en orden. Por ejemplo, "el saldo de ana refleja el débido" depende de la prueba anterior. Ene l runner pasan solo porque la máquina es nueva. 
- Link al commit: https://github.com/sheshe0529/INF384-LAB4-20220666/commit/96acba618f5806ba7bb4cf8bf5919b3daa606ef5
- Corrección (una o más: mover de etapa / dejar como informativo / eliminar / ajustar configuración / modificar tests): modificar tests.
- Evidencia (dato observado, con link al run o fragmento de la salida local): https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40964837073 

  DB_PATH=/tmp/wallet.db npm run test:integration   # 1ª vez: pass 12 fail 0
  DB_PATH=/tmp/wallet.db npm run test:integration   # 2ª vez: pass 6  fail 6

Con la corrección, tres corridas seguidas dan pass 12 fail 0.

## Parte 2 · Implementación
- Link al run en verde en main: https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40964837073
- Link al pull request: https://github.com/sheshe0529/INF384-LAB4-20220666/pull/1
- Defecto de la aplicación introducido (archivo y descripción): un usuario ya no puede transferir su saldo completo. Si diego tiene S/ 300 y transfiere 300, recibe 400 saldo insuficiente. Es un error de borde (off-by-one) en la regla de saldo.
- Job que debería detenerlo: Falla la prueba “transferir todo el saldo deja el origen en cero”. Integration también falla 3 pruebas, pero unit es la capa más barata y la primera responsable.
- Link al run fallido: https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40966483559

## Parte 3 · Decisión bajo restricción
Link al run medido: 

| Job | Duración (min:seg) |
|---|---|
| unit | 00:14 |
| integration | 00:13 |
| sast | 00:17 |
| sca | 00:17 |
| build-push | 00:47 |
| deploy | 00:17 |
| smoke | 00:08 |
| e2e | 00:15 |
| load | 12:11 |
| **Total** | 14:39 |

Resultado de load en ese run:
| Métrica | Valor |
|---|---|
| p90 | 73.11ms |
| p95 | 75.39ms |
| Máximo | 335.95ms |
| Tasa de error | 0,00 % (0 de 34 914 requests) |
| Requests por segundo | 48,43 req/s |

Conclusión sobre el comportamiento bajo carga:
Bajo 30 usuarios virtuales durante 10 minutos de carga constante, Monedero Andino respondió
con latencias bajas y muy estables: mín. 57,7 ms, mediana 67,33 ms, p90 73,11 ms y
p95 75,39 ms. Entre la mediana y el p95 hay solo 8 ms. Como el mínimo ya es 57,7 ms, la mayor
parte de la latencia es el piso de red (runner de GitHub → Function URL en us-east-1) y no
procesamiento de la aplicación.
El máximo de 335,95 ms (unas 4,5 veces el p95) corresponde a los cold starts. Cada instancia
nueva de Lambda carga la imagen, arranca Node y crea su SQLite en /tmp. Son pocos requests y
no mueven el p95.
El throughput de 48,43 req/s coincide con el máximo que permite el script. Cada iteración
dura 1,13 s (1 s de sleep más 2 requests). Con 27,5 VUs en promedio considerando las rampas,
el máximo es 27,5 × 2 / 1,13 ≈ 48,7 req/s. Por lo tanto, el límite lo puso el script y no la
API: con esta carga el sistema está holgado, y la prueba no encuentra su punto de saturación.
La tasa de error fue 0 % y los checks pasaron al 100 %, pero esto no prueba que los saldos sean
correctos. El script acepta 400 como respuesta válida, y cada instancia de Lambda tiene su
propia base, así que dos requests pueden ver saldos distintos.

SLO propuesto (percentil, valor, ventana) y justificación:
El 95 % de los requests a /saldo y /transferir responde en ≤ 200 ms, medido en una ventana
móvil de 28 días en producción. En el pipeline, la ventana es el stage de carga constante de
4 minutos del job load. Configurado en ci.yml como SLO_P95: 200.
Justificación:
- Uso el p95 (75,39 ms) y no el máximo (335,95 ms), porque el máximo lo dominan los cold
  starts y haría fallar load sin una regresión real.
- 200 ms deja unas 2,6 veces de margen sobre lo medido. Eso absorbe la variación normal entre
  runs: el runner puede caer en otra región con más latencia de red, la ventana de 4 minutos
  con rampas de 15 s concentra los cold starts en menos requests, y hay ruido de AWS. Con eso,
  load no falla por variación normal.
- 500 ms (el valor original) era 6,6 veces lo medido: dejaba pasar regresiones grandes sin
  detectarlas.
- Con 200 ms, cualquier cambio que agregue más de ~125 ms al p95 rompe el SLO y detiene el
  pipeline.

SLA propuesto y diferencia con el SLO:
SLA para los clientes: disponibilidad mensual ≥ 99,5 % y p95 mensual
≤ 500 ms. Si se incumple, el cliente recibe un crédito sobre la comisión del mes.
Diferencias con el SLO:
- El SLA es un compromiso contractual externo, con consecuencias económicas si se incumple.
  El SLO es un objetivo interno del equipo, cuyo incumplimiento activa acciones internas
  (congelar despliegues, priorizar el arreglo) pero no penalidades.
- El SLA es deliberadamente más laxo (500 ms frente a 200 ms). La brecha es el colchón que
  permite detectar una degradación cuando se rompe el SLO y corregirla antes de incumplirle
  al cliente.
- El SLA se mide desde la red del cliente, que agrega latencia que el equipo no controla.
- Si ambos tuvieran el mismo valor, cada incumplimiento interno ya sería un incumplimiento
  contractual, sin margen de reacción.

Cambios para cumplir el presupuesto:
| Cambio | Job | Minutos recuperados |
|---|---|---|
| Stages de test/load/transferir.js: 1m / 10m / 1m → 15s / 4m / 15s. El stage constante queda en 4 min, el mínimo permitido. | load | ⟨12:11 − nueva duración de load⟩ (estimado ≈ 7:30) |
| SLO_P95 en ci.yml: 500 → 200. No recupera tiempo; load sigue evaluando el SLO en cada run en main. | load | 0 |

Total resultante: ⟨suma de los jobs del nuevo run⟩ min (estimado: 2:28 del resto de jobs + ≈ 4:41 de load ≈ 7:09, bajo el presupuesto de 8 min)
Se han realizado dos commits para los cambios:
- https://github.com/sheshe0529/INF384-LAB4-20220666/commit/d2111ca38b5ecdcf215ae16e1addab1989db3344
- https://github.com/sheshe0529/INF384-LAB4-20220666/commit/4bf57b8def835455af5964ba0ec642796e879347
Link al run en main con los cambios: https://github.com/sheshe0529/INF384-LAB4-20220666/actions/runs/40968869887

Riesgo aceptado (defecto de la aplicación concreto y efecto en la métrica DORA asignada):
Al pasar la carga constante de 10 a 4 minutos (≈ 4,5 min de prueba en total), el pipeline deja
de detectar defectos que aparecen después de cierto tiempo de vida de la instancia de Lambda.
Defecto concreto: para reducir lecturas a SQLite, alguien agrega en src/wallet.js una caché en
memoria de usuarios que expira a los 5 minutos. La recarga tiene un bug: al expirar llama a un
método que el store no tiene (store.usuarios()), lanza un TypeError y el handler responde 500.
- Unit e integration no lo detectan, porque cada prueba dura milisegundos y la caché nunca
  expira.
- Con la ventana anterior (12 min), las instancias creadas en la rampa superaban los 5 minutos
  de vida. http_req_failed habría pasado el 1 % y load habría fallado.
- Con la ventana nueva, la prueba termina antes de los 5 minutos, la caché nunca expira y load
  queda en verde.
El defecto llega a producción y los clientes empiezan a recibir errores a los 5 minutos de cada
instancia.
Efecto en change failure rate: SUBE.
- Un cambio defectuoso pasa el pipeline, se despliega y obliga a un hotfix o a un rollback, así
  que cuenta como despliegue fallido.
- Este pipeline despliega antes de las pruebas post-deploy y no tiene rollback automático, así
  que load era una de las últimas barreras. Con la ventana corta, la falla ni siquiera se marca
  en el pipeline.

Decisión sobre sast (se queda / se mueve a otra etapa o evento / informativo) y razón:
sast se queda tal como está: en pre-build, bloqueante, en cada run.
- Costo: 0:17 por run, es decir 0,28 min, o 3,5 % del presupuesto de 8 min. El presupuesto se
  cumple solo con el recorte de load (≈ 7:09), así que quitar sast no es necesario para
  cumplirlo.
- Además, corre en paralelo con unit (0:14), integration (0:13) y sca (0:17), y empata con sca
  como el más lento. Moverlo no adelantaría el inicio de build-push, que igual espera a sca.
Efecto en change failure rate:
- Mantenerlo bloqueante evita que lleguen a producción patrones inseguros de severidad ERROR,
  por ejemplo una consulta SQL armada con template string en src/db.js. Ese cambio terminaría
  en un incidente de seguridad, es decir, en un despliegue fallido.
- Como el pipeline despliega sin rollback, sast es una de las barreras que actúan antes del
  despliegue. Quitarlo de main o pasarlo a informativo podría hacer subir change failure rate
  a cambio de ahorrar solo 17 s.
- Pasarlo a informativo sería la peor opción: seguiría costando los mismos 0:17 y dejaría
  pasar los hallazgos.
  

## Declaración de uso de IA
Herramientas usadas y para qué: Se ha utilizado inteligencia artificial para la búsqueda adicional de defectos en el repositorio, para puntos de mejora de los repositorios, para analizar error durante la ejecución de los cambios y para la redacción general, de forma que se entienda mejor mi explicación.
