# 06. Optimización de Rendimiento Móvil (60 FPS) y StreamingEnabled

## 1. Presupuestos de Memoria y Rendimiento Mobile-First
Más del 60% de los jugadores de Roblox operan en dispositivos móviles con $2\text{ GB}$ a $4\text{ GB}$ de RAM compartida con el sistema operativo.
- **Límite de Memoria del Cliente:** Mantener el consumo total en la Developer Console (`F9` -> `Memory` -> `Total`) por debajo de **$1.1\text{ GB}$ de RAM**.
- **Presupuesto de Fotograma (60 FPS):** El tiempo total de CPU + GPU por fotograma en el MicroProfiler (`Ctrl+F6`) no debe superar **$16.6\text{ ms}$**.
- **Geometría en Escena:** Menos de $150.000\text{ triángulos}$ visibles simultáneamente en cámara; texturas base horneadas a $1024 \times 1024\text{ px}$ (1K).

## 2. Configuración y Código Compatible con `Workspace.StreamingEnabled`
`StreamingEnabled = true` es obligatorio en producción (`StreamingMinRadius = 64`, `StreamingTargetRadius = 256`). Esto significa que las instancias lejanas en el `Workspace` **no existen** en la memoria del cliente hasta que el jugador se acerca, y pueden desaparecer (`Parent = nil`) cuando se aleja.
- **Uso de `ModelStreamingMode`:**
  - Configura `ModelStreamingMode = Enum.ModelStreamingMode.Atomic` en todos los vehículos, enemigos, estaciones interactivas y muebles compuestos. Con `Atomic`, el modelo completo y todos sus hijos se envían al cliente de una sola vez, evitando que un script de cliente encuentre el `Model` pero falle porque aún no ha llegado su `PrimaryPart`.
  - Usa `Enum.ModelStreamingMode.Persistent` únicamente en marcadores globales indispensables que todos los clientes deban conocer desde el segundo cero.
- **Prohibición de Acceso Directo en Cliente:** En un `LocalScript`, nunca escribas `workspace.Mapa.Tienda.Boton`. Usa etiquetas de `CollectionService` para reaccionar dinámicamente cuando las instancias entran o salen del radio de streaming:

```luau
--!strict
local CollectionService = game:GetService("CollectionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Trove = require(ReplicatedStorage.Packages.Trove)

local TAG_NAME = "InteractiveShopStation"
local stationTroves: { [Instance]: typeof(Trove.new()) } = {}

local function onStationStreamedIn(instance: Instance)
	if not instance:IsA("Model") then
		return
	end
	local trove = Trove.new()
	stationTroves[instance] = trove

	-- Como el modelo tiene ModelStreamingMode.Atomic, sus partes internas ya están presentes
	local prompt = instance:FindFirstChildWhichIsA("ProximityPrompt", true)
	if prompt then
		trove:Connect(prompt.Triggered, function()
			-- Abrir interfaz de tienda en cliente
		end)
	end
end

local function onStationStreamedOut(instance: Instance)
	local trove = stationTroves[instance]
	if trove then
		trove:Clean()
		stationTroves[instance] = nil
	end
end

for _, inst in CollectionService:GetTagged(TAG_NAME) do
	task.spawn(onStationStreamedIn, inst)
end
CollectionService:GetInstanceAddedSignal(TAG_NAME):Connect(onStationStreamedIn)
CollectionService:GetInstanceRemovedSignal(TAG_NAME):Connect(onStationStreamedOut)
```

## 3. Optimización Física de `TriangleMeshPart` (`CollisionFidelity` y Queries)
1. **Props Estáticos con Colisión Simple:** Fija siempre `Anchored = true` y `CollisionFidelity = Enum.CollisionFidelity.Box` (o `Hull`). Nunca dejes `Default` ni `PreciseConvexDecomposition` en objetos donde el jugador no necesite caminar por huecos internos.
2. **Elementos Puramente Decorativos (Vegetación, Detalles de Techo, Rocas Lejanas, Partículas Físicas):** Desactiva las tres banderas de simulación para excluirlos por completo del árbol espacial de colisiones y raycasts:
   - `CanCollide = false`
   - `CanTouch = false` (Crítico: evita que el motor evalúe eventos `.Touched` en cada fotograma).
   - `CanQuery = false` (Crítico: hace que `workspace:Raycast()` ignore esta malla sin coste de CPU).
3. **Reutilización de Parámetros de Consulta:** Nunca instancies `RaycastParams.new()` ni `OverlapParams.new()` dentro de un evento `PreSimulation` o `Heartbeat`. Créalos una sola vez fuera del bucle y reutilízalos.

## 4. Optimización de NPCs y Multitudes (Sin `Humanoid` en Servidor)
El objeto `Humanoid` ejecuta decenas de comprobaciones de estado y raycasts de suelo por fotograma.
- Si tu juego muestra más de 15-20 enemigos o mascotas simultáneamente, **no utilices instancias `Humanoid`**.
- En el servidor, representa cada entidad como una simple tabla de datos Luau (`Position: Vector3`, `Health: number`) o un único `Part` raíz anclado movido en lote mediante `workspace:BulkMoveTo(partsList, cframesList, Enum.BulkMoveMode.FireCFrameChanged)`.
- En el cliente, renderiza el modelo visual con un `AnimationController` + `Animator`, actualiza las articulaciones mediante `Motor6D.Transform` en `RunService.PreSimulation` y recicla los modelos al morir mediante un **Object Pool** en lugar de llamar a `Clone()` y `Destroy()` continuamente.