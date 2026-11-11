---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: alta
ultima_verificacion: 2026-04-09
---

# 30.06 Estructura de `ModuleScript`, Caché de `require()` y Prevención de Ciclos

> **Cuándo leer este nodo:** Al estructurar nuevos módulos en `src/server/`, `src/client/` o `src/shared/`, o al diagnosticar un cuelgue infinito durante el arranque del juego causado por dependencias circulares.

## 1. Reglas duras
- **HACER:** Devolver exactamente un valor (una tabla congelada con `return table.freeze(Modulo)`) al final de todo `ModuleScript`.
- **HACER:** Colocar todos los `game:GetService(...)` y `require(...)` en la parte superior del archivo, antes de declarar funciones, formando un grafo acíclico dirigido (DAG).
- **HACER:** Extraer los tipos y constantes compartidas a un tercer módulo en `src/shared/` (`*Types.luau` o `*Config.luau`) si dos servicios necesitan conocerse entre sí.
- **NO HACER:** Crear dependencias circulares de `require()` (`ModuloA` requiere `ModuloB` y `ModuloB` requiere `ModuloA`): en Roblox, esto deja ambos hilos suspendidos permanentemente (`Requested module experienced an error while loading` o interbloqueo silencioso).
- **NO HACER:** Ejecutar llamadas que cedan el hilo (`task.wait()`, `WaitForChild` sobre instancias no creadas aún, o peticiones de red/DataStore) en el ámbito superior (top-level) de un `ModuleScript` antes del `return`.

## 2. API y firmas verificadas

| Comportamiento del Motor | Regla Verificada en Roblox | Contexto | Fuente |
| :--- | :--- | :--- | :--- |
| Ejecución única por contexto | Un `ModuleScript` se ejecuta **una sola vez** por cada entorno Lua VM (1 vez en Servidor, 1 vez en cada Cliente, 1 vez por cada `Actor` paralelo). Llamadas posteriores a `require()` devuelven la misma referencia en caché. | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/classes/ModuleScript` |
| Aislamiento Servidor vs Cliente | Si tanto un `Script` como un `LocalScript` requieren un `ModuleScript` en `ReplicatedStorage.Shared`, cada uno obtiene su propia copia en su propia memoria; mutar una tabla en el cliente NO la muta en el servidor. | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/classes/ModuleScript` |
| Propagación de errores en `require` | Si el código top-level de un `ModuleScript` lanza un `error()`, cualquier otro script que intente hacer `require()` sobre ese módulo fallará con `"Requested module experienced an error while loading"`. | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/classes/ModuleScript` |

## 3. Patrón recomendado

~~~luau
--!strict
-- Orden canónico de un ModuleScript en este proyecto:
-- 1. Servicios del motor (orden alfabético)
local CollectionService = game:GetService("CollectionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- 2. Paquetes externos (Wally) y módulos compartidos (sin ciclos)
local Trove = require(ReplicatedStorage.Packages.Trove)
local CoreLoopTypes = require(ReplicatedStorage.Shared.CoreLoopTypes)

-- 3. Tipos exportados e internos
export type NodeMetadata = {
	instance: Model,
	tier: number,
}

-- 4. Constantes locales inmutables (UPPER_SNAKE_CASE)
local NODE_TAG = "ResourceNode"
local DEFAULT_TIER = 1

-- 5. Tabla del módulo y estado privado de sesión
local NodeRegistry = {}
local registeredNodes: { [Model]: NodeMetadata } = {}
local moduleTrove = Trove.new()

-- 6. Funciones privadas auxiliares
local function registerModel(instance: Instance): ()
	if not instance:IsA("Model") then
		return
	end
	local rawTier = instance:GetAttribute("NodeTier")
	local tier = if typeof(rawTier) == "number" then rawTier else DEFAULT_TIER
	registeredNodes[instance] = {
		instance = instance,
		tier = tier,
	}
end

-- 7. API pública y ciclo de vida (:Init / :Start)
function NodeRegistry.Init(): ()
	for _, inst in CollectionService:GetTagged(NODE_TAG) do
		registerModel(inst)
	end
	moduleTrove:Connect(CollectionService:GetInstanceAddedSignal(NODE_TAG), registerModel)
	moduleTrove:Connect(CollectionService:GetInstanceRemovedSignal(NODE_TAG), function(inst: Instance)
		if inst:IsA("Model") then
			registeredNodes[inst] = nil
		end
	end)
end

function NodeRegistry.GetNodeMetadata(model: Model): NodeMetadata?
	return registeredNodes[model]
end

function NodeRegistry.GetSupportedCategories(): { CoreLoopTypes.UpgradeCategory }
	return { "Tool", "Backpack" }
end

-- 8. Retorno congelado inmediato (cero yields antes de llegar aquí)
return table.freeze(NodeRegistry)
~~~

## 4. Errores comunes y antipatrones
- **Interbloqueo por `require()` cruzado entre Servicios:** Si `CombatService` hace `require(DataService)` y `DataService` hace `require(CombatService)` para dar una recompensa al subir de nivel, el motor se congela al arrancar `init.server.luau`. *Solución:* `DataService` es de nivel inferior (núcleo) y nunca requiere servicios de gameplay; expone una `Signal` (`DataService.LevelUp:Connect(...)`) a la que `CombatService` se suscribe en su `:Init()`.
- **Hacer `WaitForChild` bloqueante en la cabecera del módulo sobre instancias creadas por otro módulo:** Si el módulo se requiere antes de que la instancia exista, congela todo el cargador de `init.server.luau`. Mueve la búsqueda de instancias dinámicas dentro de `:Init()` o `:Start()`.

## 5. Cómo comprobar que está bien
1. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/` y `selene src/`.
2. Lanzar una sesión `playtest` mediante Studio MCP y confirmar en consola que `init.server.luau` e `init.client.luau` completan las fases `:Init()` y `:Start()` en menos de $100\text{ ms}$ sin mensajes de `"Infinite yield possible"`.

## 6. Fuentes
- Roblox Creator Docs — `ModuleScript`: `https://create.roblox.com/docs/reference/engine/classes/ModuleScript` (Consultado: 2026-04-09).
- Roblox Creator Docs — Modular Scripting Guide: `https://create.roblox.com/docs/scripting/scripts/module-scripts` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[04_Patrones_y_Estructura_Rojo]] | [[30_01_Modo_Strict_y_Anotaciones]] | [[32_01_Single_Script_Architecture_y_Orden_de_Carga]] | [[32_02_Ciclo_de_Vida_Init_y_Start]]

---
**Ver también (enciclopedia):** [[External Tooling - Rojo Wally Selene StyLua|Module Patterns]]
