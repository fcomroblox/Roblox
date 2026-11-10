# 01. Referencia de API de Roblox, Servicios y Task Scheduler

## 1. Fuente Oficial para Modelos de Lenguaje (`llms.txt`)
Ante cualquier duda sobre firmas de métodos nuevos, propiedades de clases o cuotas del motor, la fuente primaria oficial mantenida por Roblox es:
- **Índice Markdown para LLMs:** `https://create.roblox.com/docs/llms.txt`
- **Repositorio Oficial Creator Docs:** `https://github.com/Roblox/creator-docs`

## 2. Jerarquía y Responsabilidades de los Servicios Principales
Obtén siempre los servicios mediante `game:GetService("NombreServicio")` en la cabecera del script:

| Servicio del DataModel | Ubicación / Alcance | Uso Correcto en Producción |
| :--- | :--- | :--- |
| `Workspace` | Servidor y Cliente (Replicado) | Contiene la geometría 3D física activa y los modelos de personajes (`Character`). Se accede directamente mediante el global `workspace`. |
| `ReplicatedStorage` | Servidor y Cliente (Replicado) | Aloja módulos compartidos (`ReplicatedStorage.Shared`), paquetes de Wally (`ReplicatedStorage.Packages`), objetos `RemoteEvent` y assets que el cliente necesita clonar frecuentemente. |
| `ServerScriptService` | Exclusivo Servidor (No Replicado) | Aloja todos los scripts y módulos de lógica de servidor (`ServerScriptService.Server`) y paquetes exclusivos de servidor como `ProfileStore` (`ServerPackages`). El cliente jamás puede ver su contenido. |
| `ServerStorage` | Exclusivo Servidor (No Replicado) | Almacén seguro en servidor para mapas pesados, plantillas de jefes o herramientas que solo se clonan al `Workspace` cuando se necesitan, ahorrando memoria inicial en el cliente. |
| `StarterPlayerScripts` | Cliente (`StarterPlayer`) | Contenedor donde Rojo sincroniza el código del cliente (`Client`). Se clona una sola vez por sesión dentro de `Player.PlayerScripts` al conectarse el jugador (no se reinicia al morir el avatar). |
| `DataStoreService` | Exclusivo Servidor | Almacenamiento persistente en base de datos clave-valor. **Regla del proyecto:** Nunca se invoca en crudo; se accede siempre a través de `ProfileStore`. |
| `MemoryStoreService` | Exclusivo Servidor | Datos efímeros en RAM de alta velocidad y baja latencia entre servidores (colas de emparejamiento / matchmaking, subastas globales o bloqueos distribuidos). |
| `MessagingService` | Exclusivo Servidor | Comunicación pub/sub en tiempo real entre distintos servidores activos de la misma experiencia (`PublishAsync` / `SubscribeAsync`). |
| `CollectionService` | Servidor y Cliente | Gestión de etiquetas (*Tags*) sobre instancias (`GetTagged`, `GetInstanceAddedSignal`, `GetInstanceRemovedSignal`) para aplicar comportamientos sin depender de jerarquías rígidas de carpetas. |

## 3. Instancias de Red (`RemoteEvent`, `RemoteFunction`, `UnreliableRemoteEvent`)
1. **`RemoteEvent`:** Canal asíncrono fiable y ordenado (TCP/Reliable UDP) para acciones de juego críticas (compras, uso de habilidades, interacción con objetos).
2. **`UnreliableRemoteEvent`:** Canal asíncrono no fiable y sin garantía de orden para eventos de alta frecuencia que se actualizan constantemente (orientación de linterna/apuntado del jugador, efectos visuales efímeros, telemetría de posición secundaria). Su payload debe ser siempre inferior a $900\text{ bytes}$.
3. **`RemoteFunction`:** Petición/respuesta síncrona. **Regla Crítica:** Solo se permite `InvokeServer()` (Cliente -> Servidor) con rate-limiting estricto. Está **TERMINANTEMENTE PROHIBIDO** usar `InvokeClient()` (Servidor -> Cliente).

## 4. El Ciclo Moderno del Task Scheduler en `RunService`
Los eventos antiguos `RunService.Stepped` y `RunService.RenderStepped` están **deprecados** para nuevo código. Utiliza siempre las fases explícitas del Task Scheduler por cada fotograma en este orden exacto de ejecución:

```text
[Entrada Usuario] -> PreRender (Cliente) -> PreAnimation -> PreSimulation -> [Paso Físico Motor] -> PostSimulation -> Heartbeat -> [Renderizado GPU]
```

| Evento de `RunService` | Contexto | Cuándo Utilizarlo Obligatoriamente |
| :--- | :--- | :--- |
| `RunService.PreRender` | Solo Cliente |Reemplazo moderno de `RenderStepped`. Úsalo exclusivamente para actualizar la posición de la `Camera` o elementos visuales pegados al viewport antes de dibujar el fotograma. No ejecutes lógica pesada aquí o bloquearás los FPS. |
| `RunService.PreAnimation` | Servidor y Cliente | Se ejecuta antes de que el motor evalúe los `Animator` y aplique las poses de animación. Útil para modificar pesos de animación proceduralmente. |
| `RunService.PreSimulation` | Servidor y Cliente | Reemplazo moderno de `Stepped`. Se ejecuta **justo antes** de que el solver físico calcule colisiones y movimiento. Úsalo para aplicar fuerzas, cambiar velocidades (`AssemblyLinearVelocity`) o sobrescribir `Motor6D.Transform`. |
| `RunService.PostSimulation` | Servidor y Cliente | Se ejecuta **inmediatamente después** del cálculo físico del fotograma. Úsalo para leer las posiciones finales resultantes de la física y ejecutar validaciones anti-cheat de movimiento en el servidor. |
| `RunService.Heartbeat` | Servidor y Cliente | Se ejecuta al final del fotograma tras la simulación física (en paralelo con el renderizado). Es el lugar estándar para temporizadores de lógica general, regeneración de recursos o sistemas de IA de baja frecuencia. |

## 5. Sistema Modular Moderno de Audio
Para nuevos sistemas de sonido avanzado, prioriza el pipeline de audio modular (`AudioPlayer` -> `Wire` -> `AudioEmitter` / `AudioDeviceOutput` y efectos como `AudioCompressor`, `AudioReverb` o `AudioPitchShifter`) o instancias `Sound` agrupadas bajo un `SoundGroup` maestro para control de volumen en ajustes del jugador.