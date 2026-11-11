---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: critica
ultima_verificacion: 2026-04-09
---

# 30.07 Librería `task` (`spawn`, `defer`, `delay`, `wait`, `cancel`)

> **Cuándo leer este nodo:** Al programar corrutinas, temporizadores, ejecución asíncrona no bloqueante o cancelación de hilos activos.

## 1. Reglas duras
- **HACER:** Usar exclusivamente `task.spawn`, `task.defer`, `task.delay`, `task.wait` y `task.cancel`.
- **HACER:** Guardar el objeto `thread` devuelto por `task.delay` o `task.spawn` (o registrarlo en un `Trove` con `trove:Add(thread)`) cuando la tarea deba cancelarse si el jugador muere, cierra una ventana o sale del servidor.
- **HACER:** Preferir `task.defer` sobre `task.spawn` al disparar callbacks desde constructores o eventos de destrucción de instancias para evitar problemas de reentrancia a mitad de un cambio de estado.
- **NO HACER:** Usar jamás las funciones globales heredadas `wait()`, `spawn()` ni `delay()`.
- **NO HACER:** Usar bucles `while true do task.wait() end` por cada entidad del mundo; centraliza la actualización en un único listener de `RunService.Heartbeat` o `PreSimulation`.

## 2. API y firmas verificadas

| Función de `task` | Firma Verificada | Comportamiento en el Task Scheduler | Fuente |
| :--- | :--- | :--- | :--- |
| `task.spawn` | `task.spawn<A..., R...>(f: ((A...) -> R...) \| thread, ...: A...): thread` | Ejecuta el hilo **inmediatamente** de forma síncrona hasta su primer `yield` (`task.wait`, petición de red, etc.). | `https://create.roblox.com/docs/reference/engine/libraries/task#spawn` |
| `task.defer` | `task.defer<A..., R...>(f: ((A...) -> R...) \| thread, ...: A...): thread` | Encola el hilo para ejecutarse al final del punto de reanudación actual dentro del mismo fotograma, sin interrumpir el flujo en curso. | `https://create.roblox.com/docs/reference/engine/libraries/task#defer` |
| `task.delay` | `task.delay<A..., R...>(duration: number?, f: ((A...) -> R...) \| thread, ...: A...): thread` | Programa la ejecución en el primer paso de `Heartbeat` una vez transcurridos `duration` segundos sin estrangulamiento artificial. | `https://create.roblox.com/docs/reference/engine/libraries/task#delay` |
| `task.wait` | `task.wait(duration: number?): number` | Suspende la corrutina actual durante `duration` segundos (mínimo 1 fotograma si es `nil` o $0$) y devuelve el tiempo real transcurrido (`deltaTime`). | `https://create.roblox.com/docs/reference/engine/libraries/task#wait` |
| `task.cancel` | `task.cancel(thread: thread): ()` | Cancela un hilo suspendido (`status == "suspended"`) para que nunca se reanude. | `https://create.roblox.com/docs/reference/engine/libraries/task#cancel` |

## 3. Patrón recomendado

~~~luau
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Trove = require(ReplicatedStorage.Packages.Trove)

local RespawnTimer = {}

-- Programa la regeneración de un nodo de recursos garantizando cancelación si el nodo se destruye
function RespawnTimer.ScheduleNodeRegeneration(
	nodeModel: Model,
	respawnDelaySeconds: number,
	onRegenerated: (model: Model) -> ()
): typeof(Trove.new())
	local trove = Trove.new()
	trove:AttachToInstance(nodeModel)

	local scheduledThread: thread = task.delay(respawnDelaySeconds, function()
		if nodeModel.Parent ~= nil then
			onRegenerated(nodeModel)
		end
	end)

	-- Si nodeModel:Destroy() ocurre antes de que venza el delay, Trove ejecuta task.cancel automáticamente
	trove:Add(scheduledThread)

	return trove
end

-- Uso de task.defer para destruir o mutar jerarquía de forma segura tras un evento
function RespawnTimer.SafeDestroyDeferred(inst: Instance): ()
	task.defer(function()
		if inst.Parent ~= nil then
			inst:Destroy()
		end
	end)
end

return table.freeze(RespawnTimer)
~~~

## 4. Errores comunes y antipatrones
- **Asumir que `task.wait(1)` dura exactamente $1.0000\text{ segundos}$:** `task.wait(n)` reanuda en el primer fotograma en el que se alcanza o supera `n`. Si necesitas calcular progreso preciso (ej. barra de oxígeno o cooldown), acumula el valor devuelto `local dt = task.wait(...)` o mide `os.clock() - startTime`.
- **No cancelar un `task.delay` cuando el jugador se desconecta:** Si programas `task.delay(30, function() giveReward(player) end)` y el jugador sale del juego en el segundo 10, la función se ejecutará 20 segundos después sobre un `Player` desconectado cuyo perfil de `ProfileStore` ya cerró sesión.
- **Llamar a `task.cancel(coroutine.running())` sobre el propio hilo en ejecución:** `task.cancel` no puede cancelar el hilo que se está ejecutando actualmente; lanza un error en tiempo de ejecución.

## 5. Cómo comprobar que está bien
1. Ejecutar `selene src/` (Selene marca como error el uso de `wait`, `spawn` o `delay` heredados bajo la librería estándar de Roblox).
2. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/`.

## 6. Fuentes
- Roblox Creator Docs — `task` Library Reference: `https://create.roblox.com/docs/reference/engine/libraries/task` (Consultado: 2026-04-09).
- Roblox Creator Docs — Task Scheduler: `https://create.roblox.com/docs/studio/microprofiler/task-scheduler` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[01_API_Roblox_y_Servicios]] | [[02_Guia_Luau_Tipado_Estricto]] | [[31_04_RunService_Fases_del_Frame_y_Contexto]] | [[32_04_Patron_Trove_Limpieza_Determinista]]

---
**Ver también (enciclopedia):** [[RunService - Engine Reference Card|Task & RunService]]
