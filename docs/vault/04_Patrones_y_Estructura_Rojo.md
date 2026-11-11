---
tags: [patrones-y-estructura-rojo]
ultima_verificacion: pendiente
---
# 04. Patrones de Arquitectura Modular, Rojo y Gestión de Memoria

## 1. Por Qué NO Usamos Frameworks Heredados (Knit / AeroGameFramework)
- **Hecho Técnico:** Knit fue archivado y desaconsejado oficialmente por su propio creador (`sleitnick`) porque oculta el tipado estático de Luau (`--!strict`) detrás de tablas dinámicas globales y añade sobrecarga innecesaria sobre los `RemoteEvents`.
- **Estándar del Proyecto:** Empleamos **Single-Script Architecture** nativa con `ModuleScripts` fuertemente tipados. Un único `Script` en servidor (`src/server/init.server.luau`) y un único `LocalScript` en cliente (`src/client/init.client.luau`) requieren los módulos en orden determinista. Cuando un servicio necesita comunicarse con otro servicio del servidor, lo importa directamente mediante `require()` para preservar el autocompletado y el chequeo de tipos de `luau-lsp`.

## 2. Estructura de `default.project.json` (Rojo)
Todo archivo en `src/` se mapea en el DataModel siguiendo esta configuración base:

```json
{
  "name": "RobloxAIProject",
  "tree": {
    "$className": "DataModel",
    "ReplicatedStorage": {
      "Shared": {
        "$path": "src/shared"
      },
      "Packages": {
        "$path": "Packages"
      }
    },
    "ServerScriptService": {
      "Server": {
        "$path": "src/server"
      },
      "ServerPackages": {
        "$path": "ServerPackages"
      }
    },
    "StarterPlayer": {
      "StarterPlayerScripts": {
        "Client": {
          "$path": "src/client"
        }
      }
    }
  }
}
```

## 3. Ciclo de Vida de dos Fases: `:Init()` y `:Start()`
Para evitar bloqueos circulares (`deadlocks`) al inicializar el juego, todos los Servicios (`src/server/services/`) y Controladores (`src/client/controllers/`) exponen dos métodos opcionales:
1. **`Service:Init()` (Síncrono):** Inicializa variables internas, tablas de estado y registra escuchas de eventos locales. **Prohibido** hacer llamadas que cedan el hilo (`task.wait`, peticiones de red o DataStore) dentro de `:Init()`.
2. **`Service:Start()` (Asíncrono con `task.spawn`):** Se ejecuta únicamente cuando **todos** los módulos han terminado su `:Init()`. Aquí se inician los bucles de juego, conexiones a `Players.PlayerAdded` y comunicación activa con otros servicios.

```luau
--!strict
-- Patrón del cargador en src/server/init.server.luau
local servicesFolder = script:WaitForChild("services")

type LifecycleModule = {
	Init: ((self: LifecycleModule) -> ())?,
	Start: ((self: LifecycleModule) -> ())?,
}

local loadedServices: { LifecycleModule } = {}

for _, moduleScript in servicesFolder:GetChildren() do
	if moduleScript:IsA("ModuleScript") then
		local service = require(moduleScript) :: LifecycleModule
		table.insert(loadedServices, service)
	end
end

-- Fase 1: Inicialización síncrona
for _, service in loadedServices do
	if service.Init then
		service:Init()
	end
end

-- Fase 2: Arranque asíncrono
for _, service in loadedServices do
	if service.Start then
		task.spawn(function()
			service:Start()
		end)
	end
end
```

## 4. Gestión Determinista de Memoria con `Trove`
Toda entidad dinámica (un personaje, un vehículo, una ventana de UI temporal o una zona cargada) que conecte eventos (`RBXScriptConnection`), cree instancias o lance hilos (`task.spawn`) DEBE agrupar esos recursos dentro de un objeto `Trove` (`ReplicatedStorage.Packages.Trove`) y llamar a `trove:Clean()` o `trove:AttachToInstance(instance)` al destruirse:

```luau
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Trove = require(ReplicatedStorage.Packages.Trove)

local HazardController = {}

function HazardController.BindToModel(model: Model)
	local trove = Trove.new()
	trove:AttachToInstance(model) -- Se limpia automáticamente cuando model:Destroy() se ejecuta

	trove:Connect(RunService.PreSimulation, function(dt: number)
		-- Lógica por fotograma que se desconectará sola al destruir el modelo
		local primary = model.PrimaryPart
		if primary then
			primary.CFrame *= CFrame.Angles(0, math.rad(90) * dt, 0)
		end
	end)
end

return table.freeze(HazardController)
```

## 5. Inyección de Dependencias para Tests con Lune
En módulos de lógica pura y matemáticas en `src/shared/`, permite pasar dependencias externas (como el reloj de tiempo o el generador aleatorio `Random.new(seed)`) como argumentos para que Lune pueda ejecutar tests unitarios deterministas desde la terminal sin requerir una sesión abierta de Studio.

---
**Conexiones del Vault:** [[00_Mapa_del_Vault]] | [[CLAUDE]] | [[02_Guia_Luau_Tipado_Estricto]] | [[17_Testing_Automatizado_Lune_y_Playtest_MCP]]

---
**Ver también (enciclopedia):** [[Entity Component Systems|Architecture]], [[External Tooling - Rojo Wally Selene StyLua|External Tooling]]
