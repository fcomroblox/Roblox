# 09. Catálogo de Errores Comunes Generados por IA en Roblox y su Solución

Antes de dar por bueno un módulo o al depurar un fallo con Studio MCP y `@chrrxs/robloxstudio-mcp`, verifica que el código no incurra en ninguno de estos 7 antipatrones clásicos de los LLMs:

## Error 1: Fuga de Memoria en Tablas Indexadas por `[Player]`
- **El Problema:** Crear diccionarios en el servidor como `local playerCooldowns: { [Player]: number } = {}` o `local playerStates = {}` y olvidar eliminar la clave cuando el jugador abandona la partida. La instancia `Player` queda retenida en memoria para siempre.
- **Solución Obligatoria:** Todo script que indexe una tabla por `[Player]` debe incluir al final un bloque `Players.PlayerRemoving:Connect(function(player) tabla[player] = nil end)`.

## Error 2: Condición de Carrera en `Players.PlayerAdded` y `CharacterAdded`
- **El Problema:** Conectar `Players.PlayerAdded:Connect(onPlayerAdded)` sin iterar sobre `Players:GetPlayers()`. En Studio (`Play Solo`) o cuando un módulo arranca de forma asíncrona en `:Start()`, el primer jugador suele haber entrado **antes** de que se ejecute `:Connect()`, por lo que su perfil nunca se inicializa.
- **Solución Obligatoria:**
```luau
--!strict
local Players = game:GetService("Players")

local function onCharacterAdded(character: Model)
	-- Lógica del personaje
end

local function onPlayerAdded(player: Player)
	if player.Character then
		task.spawn(onCharacterAdded, player.Character)
	end
	player.CharacterAdded:Connect(onCharacterAdded)
end

Players.PlayerAdded:Connect(onPlayerAdded)
for _, player in Players:GetPlayers() do
	task.spawn(onPlayerAdded, player)
end
```

## Error 3: Hilos Congelados por `:WaitForChild()` Infinito en Cliente
- **El Problema:** Llamar a `workspace:WaitForChild("ZonaJefe")` sin segundo parámetro (`timeOut`) cuando `StreamingEnabled = true`. Si el jugador está lejos de esa zona, el hilo quedará pausado eternamente.
- **Solución Obligatoria:** Pasa siempre un tiempo límite explícito (`instance:WaitForChild("Hijo", 5)`) y comprueba si devolvió `nil`, o utiliza `CollectionService:GetInstanceAddedSignal(tag)`.

## Error 4: Expresiones Ternarias Falsas con `and / or`
- **El Problema:** Escribir `local visible = isMuted and false or true`. En Lua/Luau, como `false` es evaluado como falso lógico, esa expresión devuelve **siempre** `true`.
- **Solución Obligatoria:** Usa siempre la expresión ternaria nativa de Luau: `local visible = if isMuted then false else true`.

## Error 5: Modificar `Motor6D.Transform` en el Evento Incorrecto
- **El Problema:** Intentar sobrescribir `Motor6D.Transform` dentro de `RunService.Heartbeat` o `PostSimulation`. El solver de animaciones de Roblox sobrescribe `Transform` en cada fotograma antes de la física, por lo que tus cambios serán ignorados o parpadearán.
- **Solución Obligatoria:** Modifica `Motor6D.Transform` exclusivamente dentro de `RunService.PreSimulation` (que ocurre después de `PreAnimation` y antes del paso físico).

## Error 6: Clonar Plantillas o Destruir Instancias sin Limpiar Conexiones
- **El Problema:** En un bucle de interfaz, borrar elementos antiguos o desconectar pantallas dejando eventos globales (`UserInputService.InputBegan`, `RunService.PreRender`) activos apuntando a elementos destruidos.
- **Solución Obligatoria:** Vincula cada pantalla o entidad a un `Trove` con `trove:AttachToInstance(instancia)` para garantizar que la destrucción de la instancia en el DataModel desconecte todas sus señales externas.

## Error 7: Rutas Relativas Rotas en `require()` al Sincronizar con Rojo
- **El Problema:** Usar `require(script.Parent.Parent.OtroModulo)` asumiendo una estructura de carpetas ambigua cuando un archivo se llama `init.luau` (que Rojo convierte en el propio contenedor padre).
- **Solución Obligatoria:** Ancla siempre los `require()` desde el servicio raíz correspondiente:
  - `local ReplicatedStorage = game:GetService("ReplicatedStorage")`
  - `local Shared = ReplicatedStorage:WaitForChild("Shared")`
  - `local MiModulo = require(Shared.MiModulo)`