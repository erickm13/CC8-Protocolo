# PRFC-CC8-2026 — Captura la Bandera

**Especificación funcional y protocolo de comunicación**

| | |
|---|---|
| Versión del documento | `2.0.0` |
| `protocolVersion` en los mensajes | `2.0` |
| Estado | Vigente |
| Última modificación | 2026-07-23 |
| Reemplaza a | `1.0.x` (rejilla de casillas), incompatible |

> El número de versión del documento y el valor de `protocolVersion` que viaja en
> los mensajes **no son lo mismo**. El documento puede llegar a `2.0.7` por
> aclaraciones de redacción mientras los mensajes siguen diciendo `"2.0"`. El
> valor en los mensajes solo cambia cuando se rompe la compatibilidad.

## Sobre esta versión

La versión 1.0 describía el juego sobre una rejilla de casillas con movimiento
tipo serpiente. **Contradecía el reglamento del curso** en la geometría, el
movimiento, la forma de tomar y robar la bandera, y la inmunidad tras el robo.
Esta versión reescribe el protocolo sobre el reglamento y no es compatible con la
anterior. Las citas por sección de la 1.0 no se corresponden con las de esta.

Las secciones se citan por número: §14 (robo de la bandera), §29.5 (mensaje
`GAME_STATE`).

---

## 1. Propósito

Este documento define las reglas del juego, el funcionamiento del servidor y el
protocolo de comunicación que deberán respetar todos los proyectos del curso.

Cada proyecto podrá utilizar cualquier lenguaje de programación, librería, motor
gráfico o interfaz, siempre que cumpla exactamente con las reglas y mensajes
definidos aquí. Todos los proyectos deben poder conectarse entre sí.

## 2. Objetivo del juego

Cada jugador compite de forma individual por una única bandera. Para ganar, un
jugador debe:

1. Entrar al círculo central.
2. Tomar la bandera.
3. Salir completamente del círculo llevándola.

El primer jugador que lo consiga gana la partida.

Tomar la bandera no significa ganar. Es obligatorio salir del círculo con ella.

## 3. Arquitectura general

El juego utilizará una arquitectura cliente-servidor.

- Existirá un único servidor por partida.
- Todos los jugadores se conectarán directamente al servidor.
- Los jugadores no se comunicarán directamente entre sí.
- El servidor mantendrá el estado oficial del juego.
- Los clientes únicamente enviarán intención de movimiento e interacción.
- El servidor validará todas las acciones.

## 4. Modos del proyecto

Cada proyecto deberá poder ejecutarse en dos modos.

**Modo servidor.** Hospeda la partida, mantiene el estado oficial y **únicamente
muestra** el juego de todos los jugadores conectados. La máquina que corre el
servidor **no participa como jugador**: no tiene entidad en el mapa, no aparece
en `players` y no puede tomar la bandera.

**Modo cliente.** Se conecta a un servidor y es el único modo en el que se juega
desde esa máquina.

Un proyecto deberá poder conectarse al servidor de cualquier otro proyecto, y su
servidor deberá aceptar clientes de cualquier otro proyecto.

## 5. El mapa

El mapa será un plano continuo, no una rejilla.

- Las coordenadas se expresan en **unidades de mundo**, con decimales.
- El origen `(0, 0)` es el centro del mapa y también el centro del círculo.
- El eje **x** crece hacia la derecha.
- El eje **y** crece hacia **abajo**, siguiendo la convención de pantalla.

El mapa es un cuadrado de `mapSize` × `mapSize` unidades centrado en el origen.
Las coordenadas válidas van de `-mapSize / 2` a `+mapSize / 2` en ambos ejes.

Ningún jugador podrá salir del mapa: el servidor recorta la posición a esos
límites.

La conversión de unidades de mundo a píxeles es decisión de cada cliente. Un
proyecto ASCII, uno 2D y uno 3D pueden verse distintos y seguir siendo
compatibles.

## 6. El círculo central

En el centro del mapa existe un área circular de radio `circleRadius`, centrada
en el origen.

Un jugador se considera **dentro del círculo** cuando la distancia de su centro
al origen es menor o igual a `circleRadius`.

Un jugador se considera **completamente fuera del círculo** cuando:

```
distancia(jugador, origen) - playerRadius > circleRadius
```

Esa distinción importa solo para la condición de victoria (§16). No basta con
tocar el borde.

El círculo no es una barrera: cualquiera puede entrar y salir libremente en
cualquier momento, tenga o no la bandera.

## 7. La bandera

Existirá una única bandera, ubicada exactamente en el centro del mapa, en `(0, 0)`.

Estados posibles:

| Estado | Significado |
|---|---|
| `AVAILABLE` | Está en el suelo y nadie la lleva. |
| `CARRIED` | Un jugador la transporta. |
| `DROPPED` | Cayó porque su portador se desconectó. |
| `OUTSIDE` | Salió del círculo y la partida terminó. |

Mientras está `CARRIED`, la posición de la bandera es la del portador.

Mientras está `DROPPED`, permanece fija en el punto donde cayó y puede volver a
tomarse.

## 8. Jugadores

El servidor aceptará hasta `maximumPlayers` jugadores simultáneos.

Cada jugador tendrá:

| Campo | Tipo | Descripción |
|---|---|---|
| `playerId` | string | Identificador único asignado por el servidor. |
| `name` | string | Nombre visible del jugador. |
| `x` | number | Posición horizontal en unidades de mundo. |
| `y` | number | Posición vertical en unidades de mundo. |
| `moveX` | number | Componente horizontal de la intención de movimiento, de `-1` a `1`. |
| `moveY` | number | Componente vertical de la intención de movimiento, de `-1` a `1`. |
| `hasFlag` | boolean | Indica si lleva la bandera. |

No existe campo de protección ni de inmunidad. El reglamento no las contempla
(§14).

## 9. Posición inicial

Cada jugador aparecerá en una posición aleatoria **fuera del círculo**.

El servidor elegirá un ángulo aleatorio y colocará al jugador a una distancia del
origen de `circleRadius + spawnMargin`, dentro de los límites del mapa.

Todos los jugadores comienzan quietos: `moveX` y `moveY` en `0`. Ningún jugador
comienza con la bandera.

## 10. Movimiento

El movimiento es libre y continuo. No hay direcciones fijas ni casillas.

El cliente envía su **intención de movimiento** como un vector `(moveX, moveY)`
con componentes entre `-1` y `1`. Ese vector representa las teclas que el jugador
tiene presionadas en ese instante; qué teclas son es decisión de cada proyecto.

Reglas:

- Si la magnitud del vector es mayor que `1`, el servidor la normaliza a `1`. Así
  moverse en diagonal no es más rápido.
- Un vector `(0, 0)` significa que el jugador está quieto. Detenerse es una
  acción válida.
- El vector se mantiene vigente hasta que el cliente envíe otro. El cliente no
  necesita reenviarlo en cada ciclo.
- El cliente **nunca** envía posiciones. Solo intención.

En cada ciclo el servidor mueve a cada jugador así:

```
dt = tickIntervalMs / 1000
x  = x + moveX * playerSpeed * dt
y  = y + moveY * playerSpeed * dt
```

y después recorta la posición a los límites del mapa.

Los jugadores **no colisionan entre sí**. Pueden ocupar el mismo punto. La
cercanía solo importa para interactuar (§12).

## 11. Ciclo del servidor

El servidor ejecutará ciclos automáticos. Intervalo recomendado:
`tickIntervalMs`, con valor inicial de **50 milisegundos**, es decir 20 ciclos
por segundo.

Cada ciclo tendrá un número entero consecutivo, `tick`, que comienza en `1`.

Al final de cada ciclo el servidor envía `GAME_STATE` a todos los clientes.

## 12. Interacción

Tomar y robar la bandera requieren que el jugador **presione la tecla de
interacción**. Nunca ocurren de forma automática por acercarse.

Cuando el jugador la presiona, el cliente envía un mensaje `INTERACT` (§28.3). La
tecla concreta es decisión de cada proyecto.

El servidor procesará la interacción si la distancia entre el jugador y el
objetivo es menor o igual a `interactionRadius`.

Si el jugador envía varios `INTERACT` antes del mismo ciclo, el servidor
procesará **únicamente uno**. Esto no es una inmunidad: es evitar que un cliente
que envía cien mensajes por segundo tenga ventaja sobre uno que envía veinte.

Si un `INTERACT` no cumple las condiciones —está lejos, la bandera ya la lleva él
mismo, no hay nada cerca— el servidor lo **ignora en silencio**. No envía `ERROR`.
Presionar la tecla sin resultado es parte normal del juego y responder con un
error inundaría la conexión.

## 13. Tomar la bandera

Si un jugador envía `INTERACT`, la bandera está en estado `AVAILABLE` o `DROPPED`,
y la distancia entre el jugador y la bandera es menor o igual a
`interactionRadius`:

- la bandera pasa inmediatamente a pertenecer a ese jugador;
- deja de estar en el suelo y pasa a estado `CARRIED`;
- desde ese momento acompaña al jugador.

El servidor envía `FLAG_PICKED_UP` (§29.6).

## 14. Robo de la bandera

Si un jugador posee la bandera, cualquier otro jugador puede robársela.

Condiciones:

- el atacante debe estar a una distancia del portador menor o igual a
  `interactionRadius`;
- el atacante debe enviar `INTERACT`.

Si ambas se cumplen, la bandera cambia inmediatamente de propietario. Ninguno de
los dos cambia de posición y ninguno es eliminado.

**No existe tiempo de espera. No existe inmunidad. El robo es instantáneo.**

La bandera podrá cambiar de dueño tantas veces como haga falta durante la
partida, incluso en ciclos consecutivos.

El servidor envía `FLAG_STOLEN` (§29.7).

## 15. Conflictos simultáneos

Todas las interacciones de un ciclo se evalúan contra el mismo estado inicial.

Si varios jugadores envían `INTERACT` sobre el portador en el mismo ciclo:

- el servidor los ordena por `playerId` ascendente;
- **únicamente el primer robo válido se aplica**;
- los demás no producen ningún efecto.

Quien quiera robarle al nuevo portador deberá presionar la tecla otra vez en un
ciclo posterior. El orden por `playerId` debe ser estable para que todos los
servidores resuelvan igual el mismo empate.

## 16. Condición de victoria

Un jugador gana cuando:

1. tiene la bandera; y
2. cruza **completamente** el límite del círculo hacia el exterior, es decir
   `distancia(jugador, origen) - playerRadius > circleRadius`.

No basta con tocar el borde. Debe encontrarse totalmente fuera del área central.

El servidor evalúa esta condición al final de cada ciclo, después de mover a los
jugadores. Cuando se cumple:

- el estado de la partida cambia a `FINISHED`;
- la bandera cambia a estado `OUTSIDE`;
- no se procesan más movimientos ni interacciones;
- el servidor envía el `GAME_STATE` final y después `GAME_OVER` (§29.11).

## 17. Desconexiones

Si un jugador se desconecta:

- será eliminado del mapa;
- dejará de aparecer en `players` a partir del siguiente `GAME_STATE`;
- el servidor notificará a los demás con `PLAYER_DISCONNECTED` (§29.9).

Si el jugador llevaba la bandera:

- la bandera caerá en su última posición válida;
- cambiará a estado `DROPPED`;
- podrá ser recogida por cualquier jugador con `INTERACT` (§13).

No habrá reconexión automática.

## 18. Estados de la partida

| Estado | Significado |
|---|---|
| `WAITING` | Acepta jugadores. Aparece en el descubrimiento. |
| `STARTING` | Cuenta regresiva en curso. Ya no acepta jugadores. |
| `RUNNING` | La partida está en juego. |
| `FINISHED` | Existe un ganador. |
| `CANCELLED` | El servidor canceló la partida. |

No se aceptarán jugadores después de que inicie la cuenta regresiva.

## 19. Descubrimiento de servidores

El cliente deberá poder **descubrir servidores disponibles** sin que el usuario
escriba una dirección IP.

El descubrimiento usa **UDP broadcast** en `discoveryPort`. Es el único uso de
UDP en el protocolo: la partida completa viaja por TCP (§22).

Flujo:

1. El cliente envía por broadcast un `DISCOVER_REQUEST` (§27.1) a
   `255.255.255.255:discoveryPort`.
2. Todo servidor en estado `WAITING` responde por UDP directo al remitente con un
   `DISCOVER_RESPONSE` (§27.2), que incluye su puerto TCP.
3. El cliente muestra la lista y el usuario elige.
4. El cliente abre una conexión TCP al servidor elegido y envía `JOIN` (§28.1).

Un servidor que no esté en `WAITING` no responde. El cliente deberá permitir
también la conexión manual por IP y puerto, para redes donde el broadcast esté
bloqueado.

## 20. Inicio de la partida

El anfitrión —quien ejecuta el modo servidor— controla el inicio.

Flujo:

1. El servidor entra en estado `WAITING` y comienza a responder el descubrimiento.
2. Los clientes envían `JOIN`; el servidor acepta o rechaza cada conexión.
3. Cada vez que la lista de jugadores cambia, el servidor envía `LOBBY_STATE`
   (§29.3) a todos.
4. El anfitrión inicia la partida.
5. El servidor cambia a `STARTING` y envía `GAME_COUNTDOWN` (§29.4) una vez por
   segundo, desde `countdownSeconds` hasta `1`.
6. Al terminar la cuenta, genera las posiciones iniciales y envía `GAME_STARTED`
   (§29.5).
7. Cambia a `RUNNING` y comienza a enviar `GAME_STATE`.

## 21. Parámetros configurables

| Parámetro | Valor inicial | Descripción |
|---|---|---|
| `mapSize` | 2000 | Lado del mapa cuadrado, en unidades de mundo. |
| `circleRadius` | 500 | Radio del círculo central. |
| `playerRadius` | 15 | Radio del jugador, usado en la condición de victoria. |
| `spawnMargin` | 80 | Distancia extra fuera del círculo donde aparecen los jugadores. |
| `playerSpeed` | 220 | Unidades de mundo por segundo. |
| `interactionRadius` | 60 | Alcance de la tecla de interacción. |
| `tickIntervalMs` | 50 | Duración del ciclo del servidor. |
| `countdownSeconds` | 5 | Duración de la cuenta regresiva. |
| `maximumPlayers` | 100 | Máximo de jugadores simultáneos. |
| `serverPort` | 5000 | Puerto TCP de la partida. |
| `discoveryPort` | 5001 | Puerto UDP del descubrimiento. |

Los valores efectivos viajan en `GAME_STARTED`. El cliente **no** debe asumir los
valores por defecto: debe leer los que le manda el servidor.

## 22. Protocolo de transporte

| Uso | Transporte | Puerto |
|---|---|---|
| Descubrimiento de servidores | UDP broadcast | `discoveryPort` |
| Partida completa | TCP | `serverPort` |

TCP para la partida porque garantiza entrega y orden, permite detectar
desconexiones y está disponible en todos los lenguajes.

UDP únicamente para el descubrimiento, porque el broadcast lo requiere y la
pérdida de un datagrama de descubrimiento no tiene consecuencias: el cliente
vuelve a preguntar.

No se utilizará WebSocket. Un cliente que corra dentro de un navegador necesitará
un proceso puente que hable TCP y UDP del lado del sistema operativo.

## 23. Formato de comunicación

Todos los mensajes, TCP y UDP, utilizarán **JSON** codificado en **UTF-8**.

Sobre **TCP**: un mensaje JSON por línea, terminado en `\n`. El receptor lee hasta
encontrar `\n`. El carácter `\n` no forma parte del JSON. No se permiten saltos de
línea dentro del mensaje.

Ejemplo transmitido:

```
{"type":"INTERACT","protocolVersion":"2.0","gameId":"GAME-001","playerId":"P07"}\n
```

Sobre **UDP**: un datagrama contiene exactamente un mensaje JSON. El `\n` final es
opcional y el receptor debe tolerarlo.

## 24. Convenciones JSON

Los nombres de campos utilizarán **camelCase**: `playerId`, `gameId`, `moveX`,
`circleRadius`.

Tipos permitidos:

- texto: JSON string;
- números: JSON number, enteros o con decimales;
- verdadero o falso: JSON boolean;
- listas: JSON array;
- objetos: JSON object;
- ausencia de valor: JSON null.

Los valores de enumeraciones utilizarán mayúsculas: `RUNNING`, `CARRIED`,
`AVAILABLE`.

Las coordenadas y demás números con decimales se enviarán **redondeados a dos
decimales**. Es suficiente para dibujar y mantiene los mensajes cortos. El emisor
redondea; el receptor acepta cualquier cantidad de decimales.

## 25. Estructura común de mensajes

Todos los mensajes deberán contener:

```json
{
  "type": "MESSAGE_TYPE",
  "protocolVersion": "2.0"
}
```

Campos comunes adicionales cuando corresponda: `gameId`, `playerId`, `tick`.

El servidor deberá rechazar versiones de protocolo incompatibles.

## 26. Identificadores

- Jugador, asignado por el servidor: `P01`, `P02`, `P03`.
- Partida, asignado por el servidor: `GAME-001`.
- Ciclo: entero consecutivo desde `1`. El valor `tick` permite identificar el
  estado más reciente.

## 27. Mensajes de descubrimiento (UDP)

### 27.1 DISCOVER_REQUEST

Enviado por el cliente a la dirección de broadcast.

```json
{
  "type": "DISCOVER_REQUEST",
  "protocolVersion": "2.0"
}
```

### 27.2 DISCOVER_RESPONSE

Enviado por cada servidor en estado `WAITING`, por UDP directo al remitente.

```json
{
  "type": "DISCOVER_RESPONSE",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "serverName": "Partida de Ana",
  "tcpPort": 5000,
  "state": "WAITING",
  "playerCount": 3,
  "maximumPlayers": 100
}
```

El cliente obtiene la dirección IP del servidor del propio datagrama; no viaja en
el JSON, porque un servidor con varias interfaces no sabe cuál ve el cliente.

## 28. Mensajes del cliente al servidor (TCP)

### 28.1 JOIN

Solicita ingresar a la partida.

```json
{
  "type": "JOIN",
  "protocolVersion": "2.0",
  "name": "Pepito"
}
```

Campos: `name`, texto no vacío.

### 28.2 INPUT

Actualiza la intención de movimiento. Se envía cuando cambia, no en cada cuadro.

```json
{
  "type": "INPUT",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "playerId": "P07",
  "moveX": 0.0,
  "moveY": -1.0
}
```

`moveX` y `moveY` van de `-1` a `1`. El ejemplo anterior es "arriba", porque el
eje `y` crece hacia abajo (§5).

Si el cliente envía varios `INPUT` antes del mismo ciclo, solo se aplica el
último.

### 28.3 INTERACT

El jugador presionó la tecla de interacción.

```json
{
  "type": "INTERACT",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "playerId": "P07"
}
```

El cliente no indica el objetivo. El servidor decide qué corresponde según §13 y
§14.

### 28.4 LEAVE

El jugador abandona voluntariamente.

```json
{
  "type": "LEAVE",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "playerId": "P07"
}
```

## 29. Mensajes del servidor al cliente (TCP)

### 29.1 JOIN_ACCEPTED

```json
{
  "type": "JOIN_ACCEPTED",
  "protocolVersion": "2.0",
  "playerId": "P07",
  "gameId": "GAME-001"
}
```

### 29.2 JOIN_REJECTED

```json
{
  "type": "JOIN_REJECTED",
  "protocolVersion": "2.0",
  "reason": "GAME_ALREADY_STARTED"
}
```

Motivos posibles: `GAME_ALREADY_STARTED`, `GAME_FULL`, `INVALID_NAME`,
`UNSUPPORTED_PROTOCOL_VERSION`.

### 29.3 LOBBY_STATE

Enviado cada vez que la lista de jugadores cambia durante `WAITING`.

```json
{
  "type": "LOBBY_STATE",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "state": "WAITING",
  "players": [
    { "playerId": "P01", "name": "Ana" },
    { "playerId": "P02", "name": "Beto" }
  ]
}
```

### 29.4 GAME_COUNTDOWN

Enviado una vez por segundo durante `STARTING`.

```json
{
  "type": "GAME_COUNTDOWN",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "secondsRemaining": 3
}
```

### 29.5 GAME_STARTED

Envía la configuración completa al iniciar.

```json
{
  "type": "GAME_STARTED",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "mapSize": 2000,
  "circleRadius": 500,
  "playerRadius": 15,
  "playerSpeed": 220,
  "interactionRadius": 60,
  "tickIntervalMs": 50,
  "flag": {
    "x": 0,
    "y": 0,
    "status": "AVAILABLE",
    "carrierId": null
  },
  "players": [
    {
      "playerId": "P01",
      "name": "Ana",
      "x": -410.5,
      "y": -410.5,
      "moveX": 0,
      "moveY": 0,
      "hasFlag": false
    }
  ]
}
```

### 29.6 GAME_STATE

Enviado al final de cada ciclo. Es el estado oficial.

```json
{
  "type": "GAME_STATE",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "tick": 185,
  "players": [
    {
      "playerId": "P01",
      "name": "Ana",
      "x": -120.75,
      "y": 44.2,
      "moveX": 1,
      "moveY": 0,
      "hasFlag": false
    },
    {
      "playerId": "P07",
      "name": "Edgar",
      "x": 318.4,
      "y": -95.1,
      "moveX": 0.71,
      "moveY": -0.71,
      "hasFlag": true
    }
  ],
  "flag": {
    "status": "CARRIED",
    "x": 318.4,
    "y": -95.1,
    "carrierId": "P07"
  }
}
```

El arreglo `players` contiene únicamente jugadores conectados. El cliente deberá
mostrar siempre el estado con el `tick` más reciente e ignorar los que lleguen
con un `tick` menor al último recibido.

### 29.7 FLAG_PICKED_UP

```json
{
  "type": "FLAG_PICKED_UP",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "tick": 90,
  "playerId": "P07"
}
```

### 29.8 FLAG_STOLEN

```json
{
  "type": "FLAG_STOLEN",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "tick": 105,
  "previousCarrierId": "P01",
  "newCarrierId": "P07"
}
```

No lleva tiempo de protección: no existe (§14).

### 29.9 PLAYER_DISCONNECTED

```json
{
  "type": "PLAYER_DISCONNECTED",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "playerId": "P07"
}
```

### 29.10 GAME_OVER

```json
{
  "type": "GAME_OVER",
  "protocolVersion": "2.0",
  "gameId": "GAME-001",
  "winnerId": "P07",
  "winnerName": "Edgar",
  "reason": "EXITED_CIRCLE_WITH_FLAG"
}
```

### 29.11 Orden de los mensajes en un mismo ciclo

Dentro de un ciclo, el servidor envía primero los eventos —`FLAG_PICKED_UP`,
`FLAG_STOLEN`, `PLAYER_DISCONNECTED`— y al final el `GAME_STATE` de ese `tick`.
Si hay varios eventos, van en orden ascendente de `playerId` del jugador afectado.

En el ciclo de la victoria se envía el `GAME_STATE` final y **después**
`GAME_OVER`. Sin ese último estado el cliente nunca dibujaría el momento en que
el ganador cruza el borde.

### 29.12 ERROR

```json
{
  "type": "ERROR",
  "protocolVersion": "2.0",
  "code": "INVALID_INPUT",
  "description": "El vector de movimiento no es válido."
}
```

Códigos mínimos: `INVALID_MESSAGE`, `INVALID_JSON`, `INVALID_INPUT`,
`UNKNOWN_PLAYER`, `GAME_NOT_STARTED`, `GAME_ALREADY_STARTED`, `GAME_FINISHED`,
`UNSUPPORTED_PROTOCOL_VERSION`.

Un `INTERACT` que no cumple condiciones **no** genera error (§12).

## 30. Funcionamiento interno del servidor

En cada ciclo, el servidor deberá:

1. recopilar los `INPUT` pendientes y conservar solo el último de cada jugador;
2. recopilar los `INTERACT` pendientes y conservar solo uno por jugador;
3. aplicar los vectores de movimiento, normalizando los que excedan magnitud 1;
4. calcular la nueva posición de cada jugador;
5. recortar las posiciones a los límites del mapa;
6. resolver las interacciones en orden ascendente de `playerId`;
7. actualizar el portador y la posición de la bandera;
8. verificar desconexiones;
9. verificar la condición de victoria;
10. incrementar el `tick`;
11. enviar los eventos del ciclo y después el `GAME_STATE`.

## 31. Reglas de sincronización

- El servidor será la única fuente oficial.
- Los clientes no calcularán posiciones definitivas.
- Los clientes solo enviarán intención de movimiento e interacción.
- Todos los jugadores se evaluarán una vez por ciclo.
- Los movimientos e interacciones de un ciclo usarán el mismo estado inicial.
- Todos los clientes recibirán el mismo resultado.
- Un cliente deberá ignorar estados con un `tick` menor al último recibido.

Un cliente **puede** predecir localmente el movimiento de su propio jugador para
que se sienta fluido entre ciclos, siempre que corrija su posición cuando llegue
el `GAME_STATE`. Esa predicción es opcional y nunca es autoritativa.

## 32. Validaciones obligatorias del servidor

El servidor deberá validar:

- formato JSON correcto;
- tipo de mensaje conocido;
- versión de protocolo compatible;
- jugador registrado;
- que el `playerId` corresponde a la conexión que envió el mensaje;
- rango de `moveX` y `moveY`;
- partida en el estado correcto;
- límites del mapa;
- distancia para interactuar;
- condición de robo;
- condición de victoria.

Nunca deberá confiar en posiciones enviadas por el cliente.

## 33. Mensajes mínimos obligatorios

Descubrimiento (UDP): `DISCOVER_REQUEST`, `DISCOVER_RESPONSE`.

Cliente hacia servidor (TCP): `JOIN`, `INPUT`, `INTERACT`, `LEAVE`.

Servidor hacia cliente (TCP): `JOIN_ACCEPTED`, `JOIN_REJECTED`, `LOBBY_STATE`,
`GAME_COUNTDOWN`, `GAME_STARTED`, `GAME_STATE`, `FLAG_PICKED_UP`, `FLAG_STOLEN`,
`PLAYER_DISCONNECTED`, `GAME_OVER`, `ERROR`.

## 34. Compatibilidad entre lenguajes

Cada proyecto podrá usar cualquier lenguaje. Ejemplos de soporte TCP y UDP:

| Lenguaje | TCP | UDP |
|---|---|---|
| C# | `TcpClient` / `TcpListener` | `UdpClient` |
| Java | `Socket` / `ServerSocket` | `DatagramSocket` |
| Python | `socket` | `socket` con `SO_BROADCAST` |
| C/C++ | sockets del sistema | sockets del sistema |
| Go | `net.Dial` / `net.Listen` | `net.ListenPacket` |
| Node.js | `net` | `dgram` |
| Rust | `TcpStream` / `TcpListener` | `UdpSocket` |

Podrán usarse librerías, siempre que la conexión final sea **TCP + UTF-8 + JSON
por línea** para la partida y **UDP broadcast + JSON** para el descubrimiento.

## 35. Prueba mínima de compatibilidad

Antes de desarrollar el juego completo, cada proyecto deberá comprobar:

1. Envío de `DISCOVER_REQUEST` por broadcast y recepción de al menos un
   `DISCOVER_RESPONSE`.
2. Conexión TCP al servidor descubierto.
3. Envío de `JOIN` y recepción de `JOIN_ACCEPTED`.
4. Recepción de `LOBBY_STATE` al entrar otro jugador.
5. Recepción de `GAME_COUNTDOWN` y `GAME_STARTED`.
6. Envío de `INPUT` y comprobación de que la posición cambia en el `GAME_STATE`.
7. Envío de `INTERACT` cerca de la bandera y recepción de `FLAG_PICKED_UP`.
8. Lectura correcta de múltiples mensajes consecutivos.
9. Cierre correcto de la conexión y recepción de `PLAYER_DISCONNECTED` del otro
   lado.

## 36. Resumen técnico

| | |
|---|---|
| Arquitectura | Cliente-servidor |
| Servidor | Único, no juega, solo muestra |
| Descubrimiento | UDP broadcast, puerto 5001 |
| Partida | TCP, puerto 5000 |
| Formato | JSON, UTF-8, un mensaje por línea en TCP |
| Terminador TCP | `\n` |
| Mapa | Plano continuo de 2000 × 2000 unidades |
| Origen | Centro del mapa y del círculo |
| Ejes | `x` a la derecha, `y` hacia abajo |
| Círculo central | Radio 500 |
| Movimiento | Libre, vector de intención de `-1` a `1` |
| Velocidad | 220 unidades por segundo |
| Interacción | Tecla, alcance 60 unidades |
| Inmunidad | No existe |
| Ciclo | 50 ms, 20 por segundo |
| Jugadores | Hasta 100 |
| Estado oficial | Servidor |
| Lenguaje y librerías | Libres |
| Protocolo | Versión 2.0 |

Todos los proyectos deberán respetar esta especificación para garantizar que
puedan comunicarse entre sí.
