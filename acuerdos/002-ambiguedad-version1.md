# Divergencias entre el reglamento y el PRFC-CC8-2026

- **Estado:** Bloqueante — requiere decisión de la clase
- **Fecha:** 2026-07-23

Comparación entre el *Reglamento básico* entregado por el catedrático y el
[`PRFC-CC8-2026`](protocolo/PRFC-CC8-2026.md) que se está usando como
especificación común.

**No son el mismo juego.** El PRFC está bien construido y es implementable, pero
describe un juego de rejilla con movimiento tipo serpiente, mientras que el
reglamento describe un mapa continuo con un círculo central y movimiento libre
por teclado. Esto tiene que resolverse antes de que alguien escriba más código,
porque afecta la geometría, los mensajes y la calificación.

## Contradicciones directas

Casos donde el PRFC dice lo contrario del reglamento.

| # | Tema | Reglamento | PRFC | Gravedad |
|---|---|---|---|---|
| 1 | Geometría | Círculo central en un mapa continuo | Matriz rectangular de 20×20, coordenadas `[fila, columna]` desde cero| Alta |
| 2 | Movimiento | Libre, con los controles de teclado que cada quien configure | Continuo tipo serpiente: dirección activa, sin acción de detenerse  Alta |
| 3 | Tomar la bandera | Acercarse y **presionar la tecla de interacción** | Automático al llegar a la casilla| Alta |
| 4 | Robar | Acercarse y **presionar la tecla de interacción** | Automático al intentar avanzar sobre el portador| Alta |
| 5 | Inmunidad | "No existe tiempo de espera. No existe inmunidad. El robo es instantáneo." | `protectionTimeMs: 1000` de protección tras el robo | Alta |
| 6 | Victoria | Cruzar **completamente** el límite del círculo; no basta tocar el borde | Un movimiento hacia afuera desde una casilla del borde | Media |
| 7 | Modo servidor | Solo muestra el juego; únicamente en modo cliente se puede jugar | "CREAR PARTIDA" / "UNIRSE A PARTIDA" sin aclarar si el anfitrión juega | Media |
| 8 | Jugadores | Hasta 100 conexiones | `maximumPlayers: 30` (§22) | Baja |

El punto 5 es el más claro de todos: el reglamento lo dice con tres frases
seguidas, sin margen de interpretación. Toda la sección §15 del PRFC, más el
`protectionTimeMs` de `GAME_STARTED` y `FLAG_STOLEN`, contradice el reglamento.

Los puntos 3 y 4 son los más caros de arreglar: el PRFC no tiene ningún mensaje
de cliente a servidor para "interactuar". Los únicos son `JOIN`,
`CHANGE_DIRECTION` y `LEAVE`. Si la interacción requiere tecla, hace falta
un mensaje nuevo.

## Faltantes

Cosas que el reglamento pide y el PRFC no cubre.

| # | Requisito | Dónde lo pide | Estado en el PRFC |
|---|---|---|---|
| 9 | **Descubrimiento de servidores por broadcast** | Recomendaciones y Entregable ("Cliente con soporte de descubrir servidor a conectarse") | No existe. §23 además prohíbe UDP, que es lo que el broadcast necesita |
| 10 | Countdown antes del inicio | Recomendaciones | No existe; §21 deja el inicio al anfitrión |
| 11 | Unirse a una partida que aún no ha iniciado | Entregable | Cubierto por el estado `WAITING` (§20) |

El punto 9 es un entregable explícito, no una recomendación. Y choca de frente
con: *"No se utilizará UDP ni WebSocket en la versión 1.0"*. La salida
natural es acotar esa prohibición al juego: descubrimiento por UDP broadcast,
partida por TCP. Pero eso hay que escribirlo.

## Lo que sí coincide

- Arquitectura cliente-servidor con el servidor como autoridad y validador.
- Bandera única, en el centro, y todos los jugadores empiezan afuera.
- Si varios intentan robar a la vez, solo se valida el primero.
- El servidor anuncia al ganador y la partida termina para todos.
- Lenguaje y librería libres; máximo 4 por lenguaje y 2 por librería.
- Todos los proyectos deben conectarse entre sí, sin excepciones.
- Documentación desde el día 1, historial de Git y los prompts de IA usados.

## Duda sobre la organización

El reglamento dice que **el proyecto es individual** y que *"no existen grupos en
esta asignación, en otras palabras el grupo es la Clase"*. El listado que circula
tiene a los estudiantes agrupados de dos en dos, compartiendo lenguaje y
librería.

Las dos cosas pueden ser compatibles si el emparejamiento del listado solo
registra quién ocupa cada cupo de librería (máximo 2), y no equipos de trabajo.
Conviene confirmarlo, porque cambia si el entregable se sube a un repositorio por
persona o por pareja.

## Qué hacer

Tres caminos, en orden de preferencia:

**A. Consultar al catedrático antes de tocar código.** Preguntar si el juego debe
ser en mapa continuo con círculo o si acepta la formalización en rejilla. Una
respuesta de dos líneas ahorra semanas. Es la única opción que no arriesga la
nota.

**B. Reescribir el PRFC sobre el reglamento.** Mapa continuo con coordenadas
flotantes, radio del círculo como parámetro, mensaje `INTERACT` del cliente al
servidor, sin protección, descubrimiento por broadcast.

**C. Mantener el PRFC como está.** Solo si el catedrático confirma que la
formalización en rejilla es aceptable. Aun así habría que quitar la protección
y agregar el descubrimiento, porque son texto explícito del reglamento y un
entregable listado.

La calificación depende de que **todos** los proyectos se conecten entre sí, y el
reglamento advierte que si solo una minoría lo logra, esos proyectos se califican
con cero. Eso significa que esta discrepancia no la puede resolver un grupo por
su cuenta: tiene que quedar acordada por la clase completa.
