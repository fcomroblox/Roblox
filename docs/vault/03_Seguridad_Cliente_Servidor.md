---
tags: [seguridad-cliente-servidor]
ultima_verificacion: pendiente
---
# 03. Seguridad Cliente-Servidor y Prevención de Exploits

## 1. Principio de Autoridad del Servidor
En Roblox, el dispositivo del cliente es un entorno hostil: un usuario malintencionado puede leer todos los `LocalScripts` y `ModuleScripts` replicados en `ReplicatedStorage` o `StarterPlayerScripts`, alterar el `CFrame` y la velocidad de su propio personaje, e invocar cualquier `RemoteEvent:FireServer(...)` con datos falsificados. En cambio, el cliente jamás puede leer ni ejecutar nada alojado en `ServerScriptService` o `ServerStorage`.

## 2. Protocolo de Validación de 5 Capas para RemoteEvents y RemoteFunctions
Todo evento que llegue desde el cliente al servidor DEBE atravesar secuencialmente estas cinco verificaciones antes de mutar cualquier estado o perfil de datos:
1. **Limitación de Frecuencia por Jugador (Rate Limiting):** Registrar `os.clock()` por cada `[Player]` y descartar llamadas que lleguen antes del intervalo mínimo permitido. Limpiar siempre la clave del jugador en `Players.PlayerRemoving` para evitar fugas de memoria.
2. **Validación Estricta de Tipos en Tiempo de Ejecución (`typeof` y Saneamiento Numérico):** El tipado `--!strict` es estático; en tiempo de ejecución un atacante puede enviar `nil`, tablas anidadas gigantes o `0/0` (`NaN`). Todo número recibido debe comprobarse con: `typeof(x) == "number" and x == x and math.abs(x) < math.huge`.
3. **Cero Confianza en Valores de Negocio:** El cliente NUNCA envía el precio de un artículo, la cantidad de monedas ganadas ni el daño de un ataque; solo envía la intención (ej. `action = "Buy"`, `itemId = "RubyPickaxe"`). El servidor consulta su propia tabla congelada de catálogo para obtener el precio real.
4. **Verificación de Estado y Distancia Espacial en el Servidor:** Si la acción requiere estar cerca de un objeto o enemigo en el `Workspace`, el servidor comprueba que el personaje exista, tenga `Humanoid.Health > 0` y que `(hrp.Position - targetPart.Position).Magnitude <= MAX_INTERACTION_DISTANCE + LATENCY_TOLERANCE`.
5. **Prohibición Absoluta de `RemoteFunction:InvokeClient()`:** El servidor nunca debe invocar `InvokeClient(player)` porque un cliente modificado que nunca devuelva el control dejará el hilo del servidor suspendido indefinidamente.

## 3. Control de Propiedad de Red (`SetNetworkOwner`)
- Por defecto, Roblox asigna automáticamente la propiedad física (`NetworkOwnership`) de las partes no ancladas (`Anchored = false`) al jugador más cercano para distribuir el cálculo físico. Un exploiter puede teletransportar o lanzar al vacío cualquier objeto del cual sea propietario de red.
- **Regla de Seguridad:** En proyectiles autoritativos, vehículos desocupados, objetos de misión, jefes o físicas críticas de juego, fuerza siempre la autoridad del servidor inmediatamente después de instanciar el objeto en el `Workspace` llamando a `basePart:SetNetworkOwner(nil)`.

## 4. Implementación de Referencia para Receptor de Red Seguro

```luau
--!strict
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local UpgradeToolRemote = ReplicatedStorage.Remotes.UpgradeTool :: RemoteEvent

local UPGRADE_COSTS: { [number]: number } = table.freeze({
	[1] = 100,
	[2] = 250,
	[3] = 600,
})

local cooldowns: { [Player]: number } = {}
local MIN_INTERVAL = 0.3

local function isFiniteInteger(val: unknown): boolean
	return typeof(val) == "number"
		and val == val
		and math.abs(val) < math.huge
		and math.floor(val) == val
end

UpgradeToolRemote.OnServerEvent:Connect(function(player: Player, requestedTier: unknown)
	local now = os.clock()
	if (now - (cooldowns[player] or 0)) < MIN_INTERVAL then
		return
	end
	cooldowns[player] = now

	if not isFiniteInteger(requestedTier) then
		warn(`[Security] Argumento no válido de {player.Name}`)
		return
	end

	local tier = requestedTier :: number
	local requiredCost = UPGRADE_COSTS[tier]
	if not requiredCost then
		return
	end

	-- Verificar proximidad a la estación de mejora en el servidor
	local character = player.Character
	local hrp = if character then character:FindFirstChild("HumanoidRootPart") :: BasePart? else nil
	local forgePart = workspace:FindFirstChild("UpgradeForgeStation") :: BasePart?

	if not hrp or not forgePart or (hrp.Position - forgePart.Position).Magnitude > 18 then
		return
	end

	-- A partir de aquí se invoca al DataService (ProfileStore) para verificar fondos y aplicar mejora
end)

Players.PlayerRemoving:Connect(function(player: Player)
	cooldowns[player] = nil
end)
```

---
**Conexiones del Vault:** [[00_Mapa_del_Vault]] | [[05_Datos_ProfileStore_Monetizacion]] | [[14_Redes_Avanzadas_Buffers_y_Compresion]] | [[21_Seguridad_Avanzada_AntiCheat_Heuristico]]

---
**Ver también (enciclopedia):** [[Client-Server Model|Client-Server Architecture]], [[Anti-Exploit Start Here|Anti-Exploit]]
