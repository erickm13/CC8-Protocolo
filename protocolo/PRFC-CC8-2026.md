# PRFC-CC8-2026 — Captura la Bandera

**Especificación funcional y protocolo de comunicación**

| | |
|---|---|
| Versión del documento | `1.0.0` |
| `protocolVersion` en los mensajes | `1.0` |
| Estado | Vigente |
| Última modificación | 2026-07-23 |

> El número de versión del documento y el valor de `protocolVersion` que viaja en
> los mensajes **no son lo mismo**. El documento puede llegar a `1.0.7` por
> aclaraciones de redacción mientras los mensajes siguen diciendo `"1.0"`. El
> valor en los mensajes solo cambia cuando se rompe la compatibilidad.

## Sobre este documento

Las secciones se citan por número: §14 (robo de bandera), §29.4 (mensaje
`GAME_STATE`). Esa numeración es estable; si se agrega una sección nueva va al
final, no en medio.

Las decisiones sobre casos que este documento no resolvía viven en
[`../acuerdos/`](../acuerdos/) y **siempre terminan editando este archivo**. Si
una regla está en un acuerdo pero no aquí, es un error del proceso: reportalo.

Ejemplos ejecutables de mensajes y sesiones completas en
[`../ejemplos/`](../ejemplos/). Si un ejemplo contradice este documento, gana
este documento.

---

## 1. Propósito

Este documento define las reglas generales del juego, el funcionamiento del
servidor y el protocolo de comunicación que deberán respetar todos los grupos.

Cada grupo podrá utilizar cualquier lenguaje de programación, librería, motor
gráfico o interfaz, siempre que cumpla exactamente con las reglas y mensajes
definidos en este documento.

## 2. Objetivo del juego

Todos los jugadores compiten individualmente por una única bandera.

Para ganar, un jugador debe:

1. Entrar al tablero.
2. Llegar hasta la bandera.
3. Tomarla automáticamente.
4. Transportarla hasta cualquier borde.
5. Salir del tablero con la bandera.

La partida termina inmediatamente cuando un jugador sale del tablero con la
bandera.

Tomar la bandera no significa ganar. Es obligatorio salir con ella.

## 3. Arquitectura general

El juego utilizará una arquitectura cliente-servidor.

- Existirá un único servidor por partida.
- Todos los jugadores se conectarán directamente al servidor.
- Los jugadores no se comunicarán directamente entre sí.
- El servidor mantendrá el estado oficial del juego.
- Los clientes únicamente enviarán solicitudes y cambios de dirección.
- El servidor validará todas las acciones.

El servidor podrá ejecutarse en el mismo programa que el cliente, mediante dos
modos:

- CREAR PARTIDA
- UNIRSE A PARTIDA

## 4. Tablero

El tablero será una matriz rectangular configurable.

Parámetros:

- `rows`
- `columns`

Tamaño inicial recomendado: **20 filas × 20 columnas**.

Cada posición se representará mediante `[fila, columna]`. Las coordenadas
comenzarán en cero.

Ejemplo para un tablero de 20 × 20:

- Primera casilla: `[0, 0]`
- Última casilla: `[19, 19]`

## 5. Tipos de casilla

Una casilla podrá contener:

- espacio libre;
- obstáculo;
- bandera;
- jugador.

Una casilla no podrá contener más de un jugador al mismo tiempo.

## 6. Jugadores

No existirá un límite lógico fijo de jugadores simultáneos. El servidor podrá
definir un máximo técnico configurable según sus recursos.

Cada jugador tendrá:

| Campo | Descripción |
|---|---|
| `playerId` | Identificador único asignado por el servidor. |
| `name` | Nombre visible del jugador. |
| `row` | Fila actual. |
| `column` | Columna actual. |
| `direction` | Dirección activa. |
| `connected` | Indica si continúa conectado. |
| `insideBoard` | Indica si ya ingresó al tablero. |
| `hasFlag` | Indica si posee la bandera. |
| `protected` | Indica si tiene protección temporal después de un robo. |

## 7. Posición inicial

Cada jugador iniciará en una posición aleatoria fuera del tablero, junto a uno de
sus bordes.

Las posiciones externas podrán representarse así:

- Fila `-1`: arriba del tablero
- Fila igual a `rows`: debajo del tablero
- Columna `-1`: izquierda del tablero
- Columna igual a `columns`: derecha del tablero

La dirección inicial deberá apuntar hacia el interior.

Ejemplo:

- Posición inicial: `[-1, 8]`
- Dirección inicial: `DOWN`

El jugador ingresará automáticamente al tablero durante el primer ciclo en que la
casilla de entrada esté libre.

## 8. Movimiento

Las direcciones permitidas serán:

- `UP`
- `DOWN`
- `LEFT`
- `RIGHT`

No se permitirán movimientos diagonales.

El movimiento será continuo, similar al juego de la serpiente:

- cada jugador mantiene una dirección activa;
- continúa avanzando automáticamente;
- solo cambia de dirección cuando el cliente envía una nueva;
- no existirá una acción para detenerse.

Cada cliente únicamente enviará cambios de dirección. El cliente nunca enviará
posiciones nuevas.

## 9. Ciclo del juego

El servidor ejecutará ciclos automáticos de movimiento.

Intervalo inicial recomendado: **200 milisegundos**.

En cada ciclo, el servidor intentará mover una casilla a cada jugador según su
dirección activa.

El intervalo será configurable mediante `movementIntervalMs`.

## 10. Obstáculos

Los obstáculos:

- se generarán aleatoriamente al iniciar la partida;
- permanecerán fijos;
- no podrán atravesarse;
- no podrán colocarse sobre la bandera;
- no podrán bloquear completamente el acceso al centro;
- no podrán impedir que existan rutas hacia los bordes.

Porcentaje inicial recomendado: **10 %**. Parámetro: `obstaclePercentage`.

El servidor deberá comprobar que el tablero tenga rutas válidas antes de iniciar
la partida.

## 11. Bandera

Existirá una única bandera. La bandera se colocará aleatoriamente cerca del
centro del tablero. El área sugerida será el porcentaje central definido por
`centralFlagAreaPercentage`, con valor inicial recomendado de **30 %**.

La bandera se tomará automáticamente cuando un jugador llegue a su casilla.

Estados posibles:

| Estado | Significado |
|---|---|
| `AVAILABLE` | Está en su posición inicial. |
| `CARRIED` | Un jugador la transporta. |
| `DROPPED` | Cayó por desconexión del portador. |
| `OUTSIDE` | Salió del tablero y la partida terminó. |

## 12. Bloqueos

Si el siguiente movimiento encuentra:

- un obstáculo;
- una casilla ocupada;
- un límite por el cual el jugador no puede salir;

el jugador permanecerá en su posición actual.

El jugador seguirá intentando avanzar en la misma dirección durante los
siguientes ciclos. Para desbloquearse deberá cambiar de dirección o esperar a que
la casilla quede libre.

## 13. Colisiones entre jugadores

Dos jugadores no podrán ocupar la misma casilla.

Cuando un jugador intente avanzar hacia una casilla ocupada:

- el movimiento no se completará;
- el jugador atacante permanecerá en su casilla;
- el jugador bloqueador permanecerá en su casilla;
- el servidor evaluará si corresponde un robo de bandera.

## 14. Robo de bandera

Si un jugador intenta avanzar hacia la casilla ocupada por el portador:

- la bandera pasará al jugador atacante;
- ninguno de los jugadores cambiará de posición;
- ninguno será eliminado;
- ninguno regresará al inicio.

El robo será válido únicamente si el portador no tiene protección activa.

La bandera podrá cambiar de dueño múltiples veces durante la partida.

## 15. Protección después del robo

Después de robar la bandera, el nuevo portador obtendrá protección temporal.

Valor inicial: **1000 milisegundos**. Parámetro: `protectionTimeMs`.

Durante ese período:

- ningún jugador podrá robarle la bandera;
- los intentos de contacto quedarán bloqueados;
- el portador continuará moviéndose normalmente.

El servidor será responsable de calcular el inicio y final de la protección.

## 16. Conflictos simultáneos

Todos los movimientos de un ciclo se calcularán utilizando el mismo estado
inicial.

Si dos o más jugadores intentan ingresar a la misma casilla libre durante el
mismo ciclo:

- ninguno ocupará la casilla;
- todos permanecerán en sus posiciones actuales.

Si varios jugadores intentan robar la bandera en el mismo ciclo:

- el servidor resolverá el conflicto utilizando el orden interno de jugadores;
- dicho orden deberá ser estable, por ejemplo, orden ascendente de `playerId`;
- únicamente el primer robo válido será aplicado;
- inmediatamente comenzará la protección del nuevo portador.

## 17. Salida del tablero

Un jugador sin bandera no podrá salir después de haber ingresado.

El portador podrá salir desde cualquier borde. Condiciones:

- Fila `0` + dirección `UP`
- Fila `rows - 1` + dirección `DOWN`
- Columna `0` + dirección `LEFT`
- Columna `columns - 1` + dirección `RIGHT`

Cuando el portador complete ese movimiento, habrá ganado.

## 18. Victoria

El servidor declarará ganador al jugador que:

- tenga la bandera;
- se encuentre en una casilla del borde;
- complete un movimiento hacia el exterior.

Después de declarar al ganador:

- el estado cambiará a `FINISHED`;
- no se procesarán más movimientos;
- se enviará el resultado a todos los clientes;
- la bandera cambiará a estado `OUTSIDE`.

## 19. Desconexiones

Si un jugador se desconecta:

- será eliminado del tablero;
- dejará de participar;
- el servidor notificará a los demás jugadores.

Si el jugador llevaba la bandera:

- la bandera caerá en su última posición válida;
- cambiará a estado `DROPPED`;
- podrá ser recogida por otro jugador.

No habrá reconexión automática.

## 20. Estados de la partida

| Estado | Significado |
|---|---|
| `WAITING` | Acepta jugadores. |
| `STARTING` | Genera tablero, obstáculos y posiciones. |
| `RUNNING` | Ejecuta el juego. |
| `FINISHED` | Existe un ganador. |
| `CANCELLED` | El servidor canceló la partida. |

No se aceptarán jugadores después de iniciar la partida.

## 21. Inicio de la partida

El servidor controlará el inicio. Flujo:

1. El servidor entra en estado `WAITING`.
2. Los jugadores envían `JOIN`.
3. El servidor acepta o rechaza cada conexión.
4. El anfitrión inicia la partida.
5. El servidor cambia a `STARTING`.
6. Genera tablero, obstáculos, bandera y posiciones.
7. Envía `GAME_STARTED`.
8. Cambia a `RUNNING`.

## 22. Parámetros configurables

| Parámetro | Valor inicial sugerido |
|---|---|
| `rows` | 20 |
| `columns` | 20 |
| `obstaclePercentage` | 10 |
| `movementIntervalMs` | 200 |
| `protectionTimeMs` | 1000 |
| `maximumPlayers` | 30 |
| `centralFlagAreaPercentage` | 30 |
| `serverPort` | 5000 |

## 23. Protocolo de transporte

Se utilizará **TCP**.

TCP fue seleccionado porque:

- garantiza que los mensajes lleguen;
- mantiene el orden;
- evita pérdidas normales de mensajes;
- permite detectar desconexiones;
- está disponible en prácticamente todos los lenguajes;
- puede utilizarse con funciones nativas o librerías;
- simplifica la comunicación entre implementaciones distintas.

No se utilizará UDP ni WebSocket en la versión 1.0.

## 24. Formato de comunicación

Todos los mensajes utilizarán **JSON**, codificación **UTF-8**, un mensaje JSON
por línea, terminado en `\n`.

Ejemplo transmitido:

```
{"type":"CHANGE_DIRECTION","protocolVersion":"1.0","gameId":"GAME-001","playerId":"P07","direction":"LEFT"}\n
```

Reglas:

- cada JSON deberá escribirse en una sola línea;
- no se permitirán saltos de línea dentro del mensaje;
- el receptor deberá leer hasta encontrar `\n`;
- el carácter `\n` no será parte del JSON.

## 25. Convenciones JSON

Los nombres de campos utilizarán **camelCase**. Ejemplos: `playerId`, `gameId`,
`movementIntervalMs`, `protectionTimeMs`.

Tipos permitidos:

- texto: JSON string;
- números enteros: JSON number;
- verdadero o falso: JSON boolean;
- listas: JSON array;
- objetos: JSON object;
- ausencia de valor: JSON null.

Los valores de enumeraciones utilizarán mayúsculas: `LEFT`, `RUNNING`, `CARRIED`.

## 26. Estructura común de mensajes

Todos los mensajes deberán contener:

```json
{
  "type": "MESSAGE_TYPE",
  "protocolVersion": "1.0"
}
```

Campos comunes adicionales cuando corresponda: `gameId`, `playerId`, `tick`.

El servidor deberá rechazar versiones de protocolo incompatibles.

## 27. Identificadores

Jugador, asignado por el servidor: `P01`, `P02`, `P03`.

Partida, asignado por el servidor: `GAME-001`.

Ciclo: cada ciclo tendrá un número entero consecutivo (`tick: 1`, `tick: 2`,
`tick: 3`). El valor `tick` permitirá identificar el estado más reciente.

## 28. Mensajes del cliente al servidor

### 28.1 JOIN

Solicita ingresar a la partida.

```json
{
  "type": "JOIN",
  "protocolVersion": "1.0",
  "name": "Pepito"
}
```

Campos:

- `name`: texto no vacío.

### 28.2 CHANGE_DIRECTION

Cambia la dirección activa.

```json
{
  "type": "CHANGE_DIRECTION",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "playerId": "P07",
  "direction": "LEFT"
}
```

Direcciones válidas: `UP`, `DOWN`, `LEFT`, `RIGHT`.

El servidor deberá comprobar que el `playerId` pertenece a la conexión que envió
el mensaje.

Si el cliente envía varios cambios antes del mismo ciclo, únicamente se aplicará
el último.

### 28.3 LEAVE

Indica que el jugador abandona voluntariamente.

```json
{
  "type": "LEAVE",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "playerId": "P07"
}
```

## 29. Mensajes del servidor al cliente

### 29.1 JOIN_ACCEPTED

```json
{
  "type": "JOIN_ACCEPTED",
  "protocolVersion": "1.0",
  "playerId": "P07",
  "gameId": "GAME-001"
}
```

### 29.2 JOIN_REJECTED

```json
{
  "type": "JOIN_REJECTED",
  "protocolVersion": "1.0",
  "reason": "GAME_ALREADY_STARTED"
}
```

Motivos posibles: `GAME_ALREADY_STARTED`, `GAME_FULL`, `INVALID_NAME`,
`UNSUPPORTED_PROTOCOL_VERSION`.

### 29.3 GAME_STARTED

Envía la configuración completa al iniciar.

```json
{
  "type": "GAME_STARTED",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "rows": 20,
  "columns": 20,
  "movementIntervalMs": 200,
  "protectionTimeMs": 1000,
  "obstacles": [
    { "row": 2, "column": 8 },
    { "row": 2, "column": 9 }
  ],
  "flag": {
    "row": 10,
    "column": 11,
    "status": "AVAILABLE",
    "carrierId": null
  },
  "players": [
    {
      "playerId": "P01",
      "name": "Jugador 1",
      "row": -1,
      "column": 5,
      "direction": "DOWN",
      "insideBoard": false,
      "hasFlag": false,
      "protected": false
    }
  ]
}
```

### 29.4 GAME_STATE

El servidor enviará el estado oficial después de cada ciclo.

```json
{
  "type": "GAME_STATE",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "tick": 185,
  "players": [
    {
      "playerId": "P01",
      "name": "Jugador 1",
      "row": 4,
      "column": 7,
      "direction": "RIGHT",
      "insideBoard": true,
      "hasFlag": false,
      "protected": false
    },
    {
      "playerId": "P07",
      "name": "Edgar",
      "row": 10,
      "column": 13,
      "direction": "LEFT",
      "insideBoard": true,
      "hasFlag": true,
      "protected": true
    }
  ],
  "flag": {
    "status": "CARRIED",
    "row": 10,
    "column": 13,
    "carrierId": "P07"
  }
}
```

El cliente deberá mostrar siempre el estado con el `tick` más reciente.

### 29.5 FLAG_PICKED_UP

```json
{
  "type": "FLAG_PICKED_UP",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "tick": 90,
  "playerId": "P07"
}
```

### 29.6 FLAG_STOLEN

```json
{
  "type": "FLAG_STOLEN",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "tick": 105,
  "previousCarrierId": "P01",
  "newCarrierId": "P07",
  "protectionTimeMs": 1000
}
```

### 29.7 PLAYER_DISCONNECTED

```json
{
  "type": "PLAYER_DISCONNECTED",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "playerId": "P07"
}
```

### 29.8 GAME_OVER

```json
{
  "type": "GAME_OVER",
  "protocolVersion": "1.0",
  "gameId": "GAME-001",
  "winnerId": "P07",
  "winnerName": "Edgar",
  "reason": "EXITED_WITH_FLAG"
}
```

### 29.9 ERROR

```json
{
  "type": "ERROR",
  "protocolVersion": "1.0",
  "code": "INVALID_DIRECTION",
  "description": "La dirección recibida no es válida."
}
```

Códigos mínimos: `INVALID_MESSAGE`, `INVALID_JSON`, `INVALID_DIRECTION`,
`UNKNOWN_PLAYER`, `GAME_NOT_STARTED`, `GAME_ALREADY_STARTED`, `GAME_FINISHED`,
`UNSUPPORTED_PROTOCOL_VERSION`.

## 30. Funcionamiento interno del servidor

En cada ciclo, el servidor deberá:

1. recopilar los cambios de dirección pendientes;
2. conservar únicamente el último cambio de cada jugador;
3. aplicar los cambios de dirección;
4. calcular la posición propuesta de cada jugador;
5. validar límites;
6. validar obstáculos;
7. detectar conflictos de casilla;
8. resolver contactos y robos;
9. actualizar las posiciones válidas;
10. actualizar la ubicación de la bandera;
11. actualizar la protección;
12. verificar desconexiones;
13. verificar la condición de victoria;
14. incrementar el `tick`;
15. enviar `GAME_STATE` a todos los clientes.

## 31. Reglas de sincronización

- El servidor será la única fuente oficial.
- Los clientes no calcularán posiciones definitivas.
- Los clientes solo enviarán cambios de dirección.
- Todos los movimientos se resolverán en el servidor.
- Todos los jugadores se evaluarán una vez por ciclo.
- Los movimientos de un mismo ciclo usarán el mismo estado inicial.
- Todos los clientes recibirán el mismo resultado.
- Un cliente deberá ignorar estados con un `tick` menor al último recibido.

## 32. Validaciones obligatorias del servidor

El servidor deberá validar:

- formato JSON correcto;
- tipo de mensaje conocido;
- versión compatible;
- jugador registrado;
- conexión correspondiente al `playerId`;
- dirección permitida;
- partida en estado correcto;
- coordenadas válidas;
- obstáculos válidos;
- colisiones;
- robo permitido;
- protección activa;
- condición de salida;
- condición de victoria.

Nunca deberá confiar en posiciones enviadas por el cliente.

## 33. Mensajes mínimos obligatorios

Cliente hacia servidor:

- `JOIN`
- `CHANGE_DIRECTION`
- `LEAVE`

Servidor hacia cliente:

- `JOIN_ACCEPTED`
- `JOIN_REJECTED`
- `GAME_STARTED`
- `GAME_STATE`
- `FLAG_PICKED_UP`
- `FLAG_STOLEN`
- `PLAYER_DISCONNECTED`
- `GAME_OVER`
- `ERROR`

## 34. Compatibilidad entre lenguajes

Cada grupo podrá usar cualquier lenguaje. Ejemplos de soporte TCP:

| Lenguaje | API |
|---|---|
| C# | `TcpClient` / `TcpListener` |
| Java | `Socket` / `ServerSocket` |
| Python | `socket` |
| C/C++ | sockets del sistema |
| Go | `net` |
| Node.js | `net` |
| Rust | `TcpStream` / `TcpListener` |

También podrán utilizar librerías, siempre que la conexión final sea
**TCP + UTF-8 + JSON por línea**.

Un cliente que utilice UDP o WebSocket no será compatible directamente con el
servidor definido en este documento.

## 35. Prueba mínima de compatibilidad

Antes de desarrollar el juego completo, cada grupo deberá comprobar:

1. Conexión TCP al servidor.
2. Envío de un mensaje `JOIN`.
3. Recepción de `JOIN_ACCEPTED`.
4. Envío de `CHANGE_DIRECTION`.
5. Recepción de `GAME_STATE`.
6. Lectura correcta de múltiples mensajes consecutivos.
7. Cierre correcto de la conexión.

## 36. Resumen técnico

| | |
|---|---|
| Arquitectura | Cliente-servidor |
| Servidor | Único |
| Transporte | TCP |
| Formato | JSON |
| Codificación | UTF-8 |
| Separación | Un mensaje por línea |
| Terminador | `\n` |
| Coordenadas | Fila y columna desde cero |
| Movimiento | Continuo |
| Direcciones | `UP`, `DOWN`, `LEFT`, `RIGHT` |
| Sincronización | Ciclos controlados por el servidor |
| Intervalo inicial | 200 ms |
| Protección por robo | 1000 ms |
| Estado oficial | Servidor |
| Lenguaje | Libre |
| Librerías | Libres |
| Protocolo | Versión 1.0 |

Todos los grupos deberán respetar esta especificación para garantizar que sus
implementaciones puedan comunicarse entre sí.
