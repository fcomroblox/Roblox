---
tags: [fisica-constraints-y-raycasts-avanzados]
ultima_verificacion: pendiente
---
# 11. Física Moderna de Ensamblajes, Constraints y Consultas Espaciales (`Shapecasts`)

Relacionado con: [[00_Mapa_del_Vault]] | [[01_API_Roblox_y_Servicios]] | [[02_Guia_Luau_Tipado_Estricto]] | [[06_Optimizacion_Movil_Streaming]]

---

## 1. Modelo de Ensamblajes Rígidos (`Assemblies`)
En el motor físico moderno de Roblox, un conjunto de partes (`BasePart`) unidas por `WeldConstraint` o `Motor6D` forma un único cuerpo rígido llamado **Assembly**.
- **Propiedades de Masa y Velocidad:** Nunca uses las propiedades deprecadas `Part.Velocity` ni `Part.RotVelocity`. Utiliza siempre:
  - `part.AssemblyLinearVelocity = Vector3.new(x, y, z)`
  - `part.AssemblyAngularVelocity = Vector3.new(rx, ry, rz)`
  - `part.AssemblyMass` (lectura de la masa total combinada del ensamblaje) y `part.AssemblyCenterOfMass`.
- **Impulsos Instantáneos (Saltos, Explosiones, Retroceso):** Para aplicar un cambio brusco de velocidad respetando la masa física del objeto, nunca asignes `CFrame` directamente; aplica un impulso en `RunService.PreSimulation` o en respuesta a un evento:
  - `part:ApplyImpulse(direccionUnitaria * magnitudFuerza * part.AssemblyMass)`
  - `part:ApplyAngularImpulse(torqueVector * part.AssemblyMass)`

---

## 2. Sustitución de BodyMovers por Constraints Basadas en `Attachment`
Los antiguos `BodyVelocity`, `BodyPosition`, `BodyGyro` y `BodyForce` están **eliminados del flujo moderno**. Todo control físico continuo se construye instanciando un `Attachment` dentro de la pieza raíz (`AssemblyRootPart`) y vinculando una Constraint moderna:

| Caso de Uso | Constraint Moderna Oficial | Configuración Clave Recomendada |
| :--- | :--- | :--- |
| Movimiento a velocidad constante (Proyectiles físicos, cintas, dashes) | `LinearVelocity` | `VelocityConstraintMode = Enum.VelocityConstraintMode.Vector` (o `Line`/`Plane`), `RelativeTo = Enum.ActuatorRelativeTo.World`, `MaxForce = part.AssemblyMass * 5000`. |
| Mover suavemente un objeto hacia un punto (Mascotas flotantes, plataformas, telequinesis) | `AlignPosition` | `Mode = Enum.PositionAlignmentMode.OneAttachment`, `RigidityEnabled = false`, `Responsiveness = 25` (entre $5\text{ y }200$), `MaxVelocity = 40`. |
| Mantener o girar la orientación de un objeto (Vehículos, torretas, avatares mirando a objetivo) | `AlignOrientation` | `Mode = Enum.OrientationAlignmentMode.OneAttachment`, `AlignType = Enum.AlignType.AllAxes`, `Responsiveness = 30`. |
| Contrarrestar la gravedad (Flotación parcial, baja gravedad lunar, empuje de motor) | `VectorForce` | `RelativeTo = Enum.ActuatorRelativeTo.World`, `Force = Vector3.new(0, part.AssemblyMass * workspace.Gravity * factorFlotacion, 0)`. |
| Unir piezas estáticas o herramientas sin articulación | `WeldConstraint` | Más rápido y estable que el antiguo `Weld`. Solo requiere asignar `Part0` y `Part1`. |

---

## 3. Consultas Espaciales Avanzadas: `Raycast`, `Blockcast`, `Spherecast` y `OverlapParams`
Un `Raycast` tradicional tiene grosor cero (un hilo infinitesimal), lo que provoca que ataques cuerpo a cuerpo, proyectiles anchos o comprobaciones de suelo en bordes fallen al pasar por rendijas.
1. **Cuándo usar `Shapecasts` (`Blockcast` y `Spherecast`):**
   - **Detección de Suelo de Personajes/Vehículos y Proyectiles Gruesos:** Usa `workspace:Spherecast(origen, radio, direccion, raycastParams)`.
   - **Hitboxes de Armas Cuerpo a Cuerpo, Láseres Anchos o Colisión de Cajas:** Usa `workspace:Blockcast(cframeInicial, tamanoVector3, direccion, raycastParams)`.
   - *Límite de Rendimiento:* La distancia máxima de un `Shapecast` debe mantenerse por debajo de $500\text{ studs}$ para no penalizar el solver espacial.
2. **Consultas de Volumen Instantáneo (`GetPartBoundsInBox` / `GetPartBoundsInRadius`):**
   - Para explosiones de área (AoE) o zonas de captura, usa `workspace:GetPartBoundsInRadius(centro, radio, overlapParams)`.
   - Para optimizar aún más cuando solo necesitas una pieza por cada modelo de enemigo, lee el `Model` padre de cada parte devuelta y regístralo en un diccionario `hitModels[model] = true` para no aplicar daño múltiple al mismo enemigo por sus distintas extremidades.

~~~luau
--!strict
local Workspace = game:GetService("Workspace")

local CombatSpatialQuery = {}

-- Prealojamiento de parámetros fuera del bucle para evitar presión en el Garbage Collector
local sharedRaycastParams = RaycastParams.new()
sharedRaycastParams.FilterType = Enum.RaycastFilterType.Exclude
sharedRaycastParams.IgnoreWater = true

local sharedOverlapParams = OverlapParams.new()
sharedOverlapParams.FilterType = Enum.RaycastFilterType.Exclude
sharedOverlapParams.MaxParts = 64

function CombatSpatialQuery.PerformMeleeSweep(
	attackerCharacter: Model,
	originCFrame: CFrame,
	boxSize: Vector3,
	sweepDistance: number
): RaycastResult?
	sharedRaycastParams.FilterDescendantsInstances = { attackerCharacter }
	local direction = originCFrame.LookVector * sweepDistance
	return Workspace:Blockcast(originCFrame, boxSize, direction, sharedRaycastParams)
end

function CombatSpatialQuery.GetUniqueModelsInExplosion(
	center: Vector3,
	radius: number,
	 excludedInstances: { Instance }
): { Model }
	sharedOverlapParams.FilterDescendantsInstances = excludedInstances
	local parts = Workspace:GetPartBoundsInRadius(center, radius, sharedOverlapParams)

	local seenModels: { [Model]: boolean } = {}
	local resultModels: { Model } = {}

	for _, part in parts do
		local ancestorModel = part:FindFirstAncestorOfClass("Model")
		if ancestorModel and not seenModels[ancestorModel] then
			seenModels[ancestorModel] = true
			table.insert(resultModels, ancestorModel)
		end
	end

	return resultModels
end

return table.freeze(CombatSpatialQuery)
~~~

---

## 4. Gestión de `CollisionGroups` en `PhysicsService`
Para evitar que los jugadores se empujen entre sí en zonas sociales o se atasquen con sus propias mascotas:
1. Registra los grupos en el servidor al iniciar (`PhysicsService:RegisterCollisionGroup("Players")`, `PhysicsService:RegisterCollisionGroup("Pets")`).
2. Configura la matriz con `PhysicsService:CollisionGroupSetCollidable("Players", "Players", false)` y `PhysicsService:CollisionGroupSetCollidable("Players", "Pets", false)`.
3. Asigna la propiedad `basePart.CollisionGroup = "Players"` a todas las partes del `Character` cuando se dispara `CharacterAdded`.

---
**Ver también (enciclopedia):** [[Physics Assemblies and Constraints|Physics]], [[Game Math Start Here|Game Math]]
