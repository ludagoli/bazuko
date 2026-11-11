# Hoja de ruta

Estado del juego al 11 de noviembre de 2026, lo que hay que arreglar, lo que se puede acelerar y hacia dónde crece: una historia que le dé intención y progresión a lo que hoy es un sandbox.

Los hallazgos salen de leer el código y de correr `npm test`. Ninguno se midió en FPS ni se revisó a ojo en el navegador: cada fase trae su comprobación en local.

## Cómo se trabaja

1. Cada cambio va en su rama, con nombre de lo que hace: `fix/...`, `perf/...`, `feat/...`, `docs/...`.
2. Cada rama trae pruebas en verde y una nota en este archivo de qué mirar en el juego.
3. Se baja, se prueba y se ajusta en local antes de fusionar.

### Probar una rama en local

Requiere Node.js 22.12 o posterior, npm, Git y Git LFS.

```sh
git fetch origin <rama>
git checkout <rama>
git lfs pull
npm ci
npm run dev
```

Antes de fusionar:

```sh
npm test
npm run build
```

Para las pruebas de navegador: `npx playwright install chromium` y `npm run test:e2e` (puerto 5188). Para comparar rendimiento antes y después: `npm run benchmark:performance` y el panel de F3 dentro del juego, como explica `docs/rendimiento.md`. Sin `git lfs pull`, dos pruebas de audio fallan porque los cinco WAV quedan como punteros.

## Fase 0. Arreglos

### Hecho: `fix/pruebas-en-verde`

`npm test` tenía 8 fallas; quedan 0 con los WAV de LFS descargados.

| Qué | Qué mirar en el juego |
|---|---|
| La ropa limpia y el metro tienen dibujo de la letra | Costal con ropa limpia; mapa grande (M) sobre la Caracas |
| Misiones, ropero y comedor hablan de usted o en primera persona, sin puntos medios ni XP a la vista | Pausa → misiones; cumplir una; cambiarse de ropa; abrir el comedor |
| Solo se tapa la nariz quien tiene la mano libre | Con higiene bajo 30, pasar junto a un abuelo con bastón |
| Los `.DS_Store` dejan de versionarse | — |

### Pendiente: `fix/exploits-rebusque`

| Problema | Dónde | Arreglo propuesto |
|---|---|---|
| Limpiar parabrisas paga con carros parqueados, sin conductor o en marcha: $800–5.000 cada 35 s | `Car.cleanable` (`src/entities/Car.js:180`) | Solo carros en circulación, detenidos (semáforo o trancón) |
| El minijuego del espejo se reinicia con E y devuelve los intentos; un fallo dispara la rabia del conductor sin robo | `InteractionSystem.js:185-190`, `features/steal.js:162-183` y `311-324` | Ignorar E y Q mientras dura; un fallo deja el carro en enfriamiento y no emite `mirror-stolen` |
| Q junto a un carro limpia el parabrisas y le quita el robo directo a un peatón cercano | `InteractionSystem.js:200`, `steal.js:24` | El carro cede Q al peatón más cercano, como ya cede E |
| El comedor se anuncia como comida gratis durante su espera, y la espera se rompe al recargar (cada entrada fija las 22:00) | `game.js:434-446` y `1446` | Contar la espera en tiempo jugado; quitar el precio 0 mientras no sirve |
| «Cambio de pinta» se cumple sin cambiar nada | `game.js:417` | Comparar la ropa antes y después |
| El medidor de olor del HUD nunca se dibuja | `HUD.js:542-550` | Añadirlo a `NEED_META` o quitar el código |
| El espejo del minijuego se pinta como emoji | `steal.js:180-182` | Pasarlo por `picto` |

### Por conversar con sus autores

- **Metro.** El viaducto es una caja recta en X 70,8–79,2, Z 10–310: atraviesa la Torre Colpatria (X 69–93, Z 155–181), sus pilares caen sobre el tablero del viaducto de buses y sobre el lago sin llegar al lecho, y las escaleras de estación quedan en calzada sin colisión. Su prueba solo comprueba que los objetos existan. Hay que verlo en el juego y decidir si sigue el eje curvo del viaducto, si sube de cota o si se corre.
- **Olor.** Con higiene bajo 30 huyen todos los peatones a 3 metros, también los fijos. Un fijo fuera de su sitio ha causado atascos de minutos: falta medirlo con sustos como pide `docs/testing.md`.
- **Zona «Metro».** El cartel sale al cruzar la Caracas en cualquier punto y le cambia el arquetipo a quien nace ahí.

## Fase 1. Rendimiento

Ordenado por ganancia estimada. Cada punto se mide antes y después.

### `perf/audio-hud-semaforos` (seguro)

| Punto | Dónde | Cambio |
|---|---|---|
| Howler recibe `volume()` y `rate()` en cada cuadro para hasta 12 voces; `rate()` reinicia temporizadores | `VehicleAudio.js:199-204` | Llamar solo si el valor cambió más de un umbral |
| Cada muestra de frenado recorre los 24 cruces | `TrafficSystem.js:814`, `TrafficSignals.js:78` | Prefiltrar por vehículo los cruces a su alcance |
| El HUD escribe dos `title` y arma el texto de la misión en cada cuadro | `HUD.js:548-553`, `724-726` | Comparar antes de escribir |
| Se pide `models/animaciones.glb`, que no existe, antes del archivo real | `RealisticCharacter.js:31` | Quitarlo de la lista |
| El puesto del comedor son 32 mallas con 32 materiales, a 13 m del inicio | `ComedorComunitario.js:15` | Compartir materiales y agrupar |

### `perf/basura-por-cuadro`

- `SpatialGrid` arma una clave de texto por celda (`SpatialGrid.js:19`): clave numérica.
- `_coordinateIntersections` crea arreglos por cruce aunque esté vacío (`TrafficSystem.js:619-689`).
- Las cajas de vehículos se reconstruyen en cada cuadro y la OBB de los buses no descarta por distancia (`game.js:1139`, `CollisionSystem.js:88-216`).
- Parqueados y bolsas recalculan matrices sin moverse (`Renderer.js:393`).
- Las palomas se repintan aunque estén quietas (`PigeonFlock.js:93-170`).

### `perf/chato-navegacion`

`CompanionNavigation.isOpen` recorre los ~1.370 colisionadores sin usar el índice espacial (`CompanionNavigation.js:22`); el constructor lo hace 32.200 veces al cargar. Usar `collisions.queryStatic`.

### `perf/esqueleto-compartido` (el más grande y el más delicado)

Cada prenda crea su propio `THREE.Skeleton` (`RealisticCharacter.js:271`): una persona lleva de 5 a 10 y cada uno sube su textura de huesos por cuadro. En `hombre-adulto` todas las piezas comparten los mismos 52 huesos y las mismas inversas, salvo el policía. Un esqueleto por persona bajaría las subidas de unas 200–300 a unas 40 por cuadro. Falta revisar los otros cinco cuerpos; toca el descarte de personajes y `mixamoGrounding.js`.

### Peso del despliegue

`MODELOS_SIN_USO` en `vite.config.js` deja pasar a `dist/` unos 300 MB que el juego no carga (ambulancia, patrullas, motos, `models/vehiculos/`, peces). No afecta la carga del jugador; sí el tamaño de lo publicado. Revisar `tools/` y el port de `cpp/` antes de retirar algo.

## ¿Rust o Go para el rendimiento?

**Hoy no vale la pena.** Las razones, de lo visto en el código:

1. **El costo no está en cálculo puro.** Los puntos calientes son subidas a la GPU, llamadas a Howler, escrituras al DOM y objetos de vida corta. WebAssembly no toca la GPU ni el DOM: todo eso sigue pasando por JavaScript.
2. **Cruzar la frontera cuesta.** El estado vive en objetos de Three.js. Un módulo WASM necesita copiar posiciones a memoria lineal y de vuelta en cada cuadro; para 80 peatones y 50 carros la copia pesa tanto como la cuenta.
3. **Go es mala opción en navegador.** Su WASM trae el runtime y el recolector (varios MB, menos con TinyGo), y sus llamadas a JS son lentas. El juego ya descarga bastante.
4. **Rust sí sirve, pero solo para núcleos numéricos aislados**, con entrada y salida en arreglos planos. Agregaría una cadena de compilación (cargo, wasm-pack) a un proyecto que hoy es solo npm, y cada colaborador tendría que instalarla.

**Qué hacer en su lugar, sin reescribir:**

- Los arreglos de la Fase 1, que son de algoritmo y dan más que cambiar de lenguaje.
- **Web Workers en JavaScript** para lo que no necesita respuesta en el mismo cuadro: el A* del GPS (`GpsNavigation.js`), las rutas del Chato y la decodificación de modelos al cargar. Sacan trabajo del hilo principal sin lenguaje nuevo.

**Cuándo reconsiderar Rust:** si después de la Fase 1 el perfilador muestra más de ~2 ms por cuadro en una función numérica pura. Los candidatos serían la simulación de tráfico (1.140 líneas, con sus 5 semillas de prueba como red de seguridad) o una física más pesada que la caída articulada actual. Sería un módulo, con el JS actual como respaldo, nunca una reescritura.

## Fase 2. Historia

### La intención

Hoy el juego es aguantar: recoger, vender, comer, dormir, sin final. La historia le da un porqué a cada vuelta del bucle y un reloj que empuja. Se apoya en lo que ya existe (origen, cambuche, camioneta, comedor, radio, barrios del oriente) y lo ordena.

Tono: el del resto del juego. Sobrio, en primera persona o de usted, sin premiar el sufrimiento, sin marcas. La limpieza social sigue sin cuentas ni logros.

### Premisa

**El Cartucho tiene fecha.** La radio anuncia que van a tumbar el sector para hacer un parque. Quedan 30 días de juego. Lo que se haga en ese tiempo decide dónde amanece el personaje el día 31.

Da un reloj visible, es historia real del sitio y convierte cada sistema en una respuesta posible: ahorrar para una pieza, organizarse con los vecinos, hacerse a un lugar en otro barrio, o dejarse ir.

### El viaje

| Acto | Etapa | Días | Qué pasa | Qué enseña o abre |
|---|---|---|---|---|
| I | Mundo ordinario | 1–2 | Primera noche frente al cambuche. El origen elegido da el prólogo | Necesidades, basura, vender |
| I | Llamada | 2–3 | Boletín de radio y un papel pegado en el cambuche: desalojo | El reloj de la historia en el HUD |
| I | Rechazo y mentor | 3–6 | Don Efraín, reciclador de treinta años en la calle, le enseña la ruta | Carreta, bodegas, mesa: el tutorial que hoy falta |
| II | Umbral | 7–9 | Primera salida larga, a La Candelaria y Chapinero | Los barrios del oriente con algo propio |
| II | Pruebas, aliados, enemigos | 10–20 | Encargos de los vecinos; la deuda y la olla aprietan | Confianza, fiado, trabajos por turno |
| II | La cueva | 21–24 | La camioneta se lleva a alguien del sector | Decisión: buscarlo, callar o irse |
| III | Prueba suprema | 25–29 | Llegan las máquinas; una noche en que no alcanza para todo | Qué se salva y a quién |
| III | Regreso | 30–31 | Amanece el día después | Uno de los finales |

### Personajes

Todos reemplazan a fijos existentes: se mantienen los 80 peatones y 8 vendedores.

| Quién | Papel | Qué da |
|---|---|---|
| Don Efraín, reciclador | Mentor | Tutorial, rutas, el precio del día; a mitad del camino falta, y hay que seguir solo |
| Doña Marta, la de la fritanga | Aliada | Fiado, calor, noticias del sector |
| Yuli, del comedor | Heraldo y aliada | Ración, papeles, la puerta al final de «salir» |
| El Mono, de la olla | Tentador | Plata rápida y vicio; cada favor suyo cuesta después |
| El cobrador del gota a gota | Enemigo | Deuda con intereses; si el origen es «gota a gota», ya empieza debiendo |
| La camioneta | Amenaza | Sin rostro ni diálogo, como hoy |
| El Chato | Compañero | Ver «Mascota» |

### Decisiones

Pocas y con peso. Cada una mueve una o dos variables guardadas y abre o cierra misiones.

- Aceptar o no el primer préstamo del cobrador.
- Guardarle un paquete al Mono, o decirle que no.
- Cuando se llevan a alguien: preguntar por él, callar, o irse del sector.
- En la última semana: gastar lo ahorrado en una pieza, o en que los vecinos no se queden sin nada.
- Consumir con alguien para ganar su confianza.

Variables: `confianza` por personaje (0 a 3), `deuda`, `ahorro`, `vicio` (ya existe), `buscado` (ya existe).

### Finales

| Final | Condición | Cierre |
|---|---|---|
| Una pieza | Ahorro suficiente, papeles con Yuli, vicio bajo control | Amanece bajo techo; el cambuche queda atrás |
| El sector | Confianza alta con los vecinos | Se van juntos a otro lote; el cambuche se arma de nuevo, más grande |
| La olla | Deuda alta o vicio sin control | Queda trabajando para el Mono |
| La calle | Ninguna de las anteriores | Amanece en otro andén, con el costal y el Chato |

Después del final se puede seguir jugando en modo libre, con lo ganado.

### Mecánicas nuevas

| Mecánica | Para qué | Se apoya en |
|---|---|---|
| Diálogo con opciones, en cartón y marcador | Contar la historia y decidir | `Menu`, `Momentos`, `pictos.js` |
| Misiones principales encadenadas, junto a las diarias | Dar dirección | `DailyMissionSystem`, GPS |
| Reloj de la historia en el HUD | Urgencia | `TimeSystem`, radio |
| Confianza por vecino | Progresión social: fiado, precio, aviso de la camioneta | `mapServices.js` |
| Ahorro guardado (caleta, o con doña Marta) | Meta a largo plazo; lo que no roban al dormir | `SaveSystem`, cambuche |
| Deuda con intereses diarios | Presión y enemigo | `StatsSystem` |
| Trabajos por turno: descargar en la bodega, cuidar carros, lavar ollas | Variedad frente al reciclaje | Minijuego de barra ya existente |
| Rutas y precios por barrio | Planear el día con la carreta | Bodega nueva al oriente, radio |
| Cambuche, segunda etapa: candado, caleta, tendedero, olla, balde | Hogar que crece; uso para lo que hoy solo se vende | `data/cambuche.js` |
| Sucesos de calle: jornada de la Alcaldía, operativo, camión de cartón | Días distintos | Radio, avisos de zona |
| Diario | Recordar lo decidido y lo pendiente | Pestaña de pausa |

### Mascota

El juego tiene hoy una regla: **El Chato es el único perro**, y su amistad y apariencia se guardan. Dos caminos:

- **Recomendado: darle oficio al Chato.** Tres trucos por amistad: *busca* (marca la mejor bolsa cerca), *cuida* (menos robo al dormir) y *avisa* (gruñe hacia la patrulla o la camioneta antes de verlas). Hambre propia, plato y cobija en el cambuche, y un lugar en la historia: en la prueba suprema, él también cuenta.
- **Segunda mascota** (un gato del cambuche que espanta ratas y cuida lo guardado, o una paloma amaestrada): exige cambiar esa regla del proyecto y acordarlo con los demás colaboradores. Mejor después del Chato con oficio, si todavía hace falta.

### Voces y modelos

- **Voces.** Una voz por personaje con nombre, 10 a 15 líneas cada una, y el locutor de la radio para los boletines de la historia. Suena una sola voz a la vez y cada volumen pasa por `_level('voices')`. Archivos en Opus o MP3.
- **Modelos.** Un accesorio distintivo por personaje, piezas del cambuche, la báscula de la bodega nueva, el plato y la cobija del Chato. Una sola malla por accesorio, material compartido, sin marcas ni logos.
- Cada objeto nuevo lleva icono y fila en `docs/iconos.md`, y cada texto usa caracteres que existan en la letra.

## Fase 3. Sitio, cuentas y estadísticas

El juego hoy no necesita servidor: se publica como sitio estático (`.github/workflows/pages.yml`) y la partida vive en `localStorage`. Esa base se conserva: **sin cuenta se juega igual**, y la cuenta solo añade.

### Alojamiento

| Opción | A favor | En contra |
|---|---|---|
| GitHub Pages (el de hoy) | Ya funciona, sin costo, despliega con cada fusión | Límite de 1 GB por sitio y de ancho de banda mensual; `dist/` pesa 330 MB |
| Otro alojamiento estático gratuito | Más ancho de banda y dominio propio | Algunos limitan el tamaño por archivo a 25 MB; `dist/` tiene hoy 3 archivos mayores |

Lo primero, en cualquier caso, es adelgazar `dist/` retirando los ~300 MB de modelos que el juego no carga (ver «Peso del despliegue»). Con eso basta GitHub Pages por un buen tiempo. Los límites de cada servicio cambian: confirmarlos antes de elegir.

### Cuentas y datos

Un servicio con base de datos y autenticación en su plan gratuito (Supabase es el candidato) cubre lo que se quiere sin escribir servidor: el navegador habla con él directamente.

| Función | Qué hace | Riesgo |
|---|---|---|
| Entrar con GitHub | Identifica al jugador; opcional | Bajo |
| Partida en la nube | Copia del guardado para seguir en otro equipo | Decidir qué copia gana si difieren |
| Estadísticas propias | Días sobrevividos, final alcanzado, plata reciclada, causa de cada muerte | Bajo |
| Tablas del sector | Quién ha durado más, finales más vistos | Trampas: todo se calcula en el navegador |
| Página del proyecto | Presentación, capturas, colaboradores, enlace a jugar | Ninguno |

Lo que hay que cuidar:

- **Llaves.** En el navegador solo va la llave pública; cada tabla lleva reglas por fila para que cada quien escriba solo lo suyo. Ninguna llave de servicio entra al repositorio.
- **Trampas.** El juego corre entero en el cliente, así que cualquier cifra enviada se puede falsear. Las tablas son un adorno entre amigos, no una competencia; no se dan premios por ellas.
- **Datos personales.** Guardar solo el identificador de la cuenta y el nombre público; avisar qué se guarda y permitir borrarlo.
- **Tono.** Nada de tablas ni cuentas sobre muertes por limpieza social.
- **El plan gratuito.** Suele pausar los proyectos sin actividad y limitar almacenamiento y usuarios: revisar las condiciones vigentes.
- **Pruebas.** Sin red o sin cuenta, el juego y sus pruebas siguen funcionando con `localStorage`.

### Ideas anotadas: funciones y archivos fuera del sitio

Sin decidir; para retomar cuando llegue esta fase. Los planes gratuitos cambian: confirmar sus condiciones antes de elegir.

- **No hace falta un servidor propio.** Entrar, guardar la partida y las estadísticas propias se resuelven desde el navegador con reglas por fila. Solo piden código de servidor las tablas con algún control contra trampas y las tareas con llaves secretas: una o dos funciones.
- **Dónde irían esas funciones.** Funciones del mismo servicio de datos (lo más simple), o funciones en el borde del proveedor que aloje los archivos. Algunos planes gratuitos de funciones son solo para uso personal y no comercial.
- **Modelos y audio en un almacén de objetos sin cobro por transferencia** (R2 de Cloudflare es el candidato). Pide una base de recursos configurable al compilar, con el propio sitio como respaldo para que en local nada cambie (hoy `MODEL_BASE` sale de `BASE_URL`), CORS y caché larga con nombres versionados, y un paso de subida en el despliegue.
- **Cuidados.** Los modelos sin permiso de redistribución que excluye `.gitignore` tampoco van a un almacén público; los WAV de Git LFS se suben como audio real, no como punteros; los secretos viven en el panel del servicio, nunca en el repositorio.

### Orden

| Rama | Contenido |
|---|---|
| `chore/dist-liviano` | Retirar de `dist/` los modelos sin uso |
| `feat/sitio` | Página de presentación con enlace a jugar |
| `feat/cuenta-github` | Entrar con GitHub y partida en la nube, opcional |
| `feat/estadisticas` | Estadísticas propias y tablas del sector |

Va después del primer acto de la historia: las estadísticas valen más cuando hay finales y decisiones que contar.

## Orden de entrega

| # | Rama | Contenido | Tamaño |
|---|---|---|---|
| 1 | `fix/pruebas-en-verde` | Hecho | Chico |
| 2 | `fix/exploits-rebusque` | Fase 0 pendiente | Chico |
| 3 | `perf/audio-hud-semaforos` | Rendimiento seguro | Chico |
| 4 | `perf/basura-por-cuadro` y `perf/chato-navegacion` | Menos basura por cuadro | Mediano |
| 5 | `perf/esqueleto-compartido` | Un esqueleto por persona, con medición | Mediano, delicado |
| 6 | `feat/dialogo` | Motor de diálogo y datos de historia, con una escena de prueba | Mediano |
| 7 | `feat/historia-acto-1` | Llamada, mentor, reloj del desalojo, diario | Grande |
| 8 | `feat/chato-con-oficio` | Trucos, hambre, plato | Mediano |
| 9 | `feat/vecinos-y-confianza` | Aliados, fiado, ahorro, deuda | Grande |
| 10 | `feat/rutas-y-bodegas` | Barrios con basura y precios propios, trabajos por turno | Mediano |
| 11 | `feat/historia-acto-2` | Umbral, pruebas, la cueva | Grande |
| 12 | `feat/cambuche-segunda-etapa` | Piezas nuevas | Mediano |
| 13 | `feat/historia-acto-3` | Prueba suprema, finales, modo libre | Grande |

Los pasos 2 a 5 no cambian lo que se juega; del 6 en adelante cada rama deja el juego jugable aunque la historia esté a medias.

## Por decidir

1. La premisa: desalojo del Cartucho (propuesta), o la desaparición de alguien, o la deuda como eje.
2. Si la historia tiene final con modo libre después, como se propone, o queda abierta.
3. Mascota: el Chato con oficio, o cambiar la regla para una segunda.
4. Qué hacer con el Metro y con la huida por olor, junto con sus autores.
5. Quién es dueño del proyecto de datos y del dominio, y si las tablas del sector son públicas.
