---
tags: [guia-luau-tipado-estricto]
ultima_verificacion: pendiente
---
# 02. Guía de Luau Moderno y Tipado Estricto (--!strict)

## 1. Construcciones Nativas de Luau Obligatorias
Luau incorpora mejoras sintácticas y de rendimiento sobre Lua 5.1 que deben emplearse siempre:
- **Interpolación de cadenas:** Usa comillas invertidas `` `Jugador {player.Name} alcanzó el nivel {level}` `` en lugar de concatenaciones manuales o `string.format` simples.
- **Asignación compuesta:** Usa `+=`, `-=`, `*=`, `/=`, `%=`, `^=`, `..=`. Recuerda que los operadores `++` y `--` NO existen en Luau.
- **Expresiones ternarias seguras:** Usa `local resultado = if condicion then valorVerdadero else valorFalso`. Nunca uses `condicion and valorVerdadero or valorFalso` cuando `valorVerdadero` pueda evaluarse como `false` o `nil`.
- **Control de bucles e iteración:** `continue` es nativo en todos los bucles. Itera tablas directamente con `for clave, valor in coleccion do` sin llamar a `pairs()` ni `ipairs()`.
- **Librería `table` extendida:** Usa `table.freeze(t)` para hacer inmutables las tablas de configuración, `table.clone(t)` para copias superficiales rápidas, `table.clear(t)` para vaciar tablas prealojadas reutilizando su memoria y `table.create(n, valor)` cuando conozcas el tamaño de un array de antemano.

## 2. Reglas de Tipado Estricto (--!strict)
Todo archivo `.luau` comienza con `--!strict`. Se deben exportar los tipos de dominio y anotar todas las firmas de funciones. Prohibido usar `:: any` para silenciar errores de tipo salvo en mocks de tests; utiliza refinamiento de tipos (`typeof`, `IsA`, `assert`) o tipos genéricos.

```luau
--!strict
export type Rarity = "Common" | "Rare" | "Epic" | "Legendary"

export type WeaponConfig = {
	id: string,
	displayName: string,
	baseDamage: number,
	cooldownSeconds: number,
	rarity: Rarity,
	elementalTag: string?, -- Equivalente a string | nil
}

-- Patrón oficial para clases orientadas a objetos con metatables en modo --!strict
local WeaponInstance = {}
WeaponInstance.__index = WeaponInstance

type WeaponInternalData = {
	config: WeaponConfig,
	durability: number,
	lastSwingTime: number,
}

export type WeaponInstance = typeof(setmetatable({} :: WeaponInternalData, WeaponInstance))

function WeaponInstance.new(config: WeaponConfig, initialDurability: number): WeaponInstance
	local self: WeaponInternalData = {
		config = config,
		durability = initialDurability,
		lastSwingTime = 0,
	}
	return setmetatable(self, WeaponInstance)
end

function WeaponInstance.CanSwing(self: WeaponInstance, currentTime: number): boolean
	return self.durability > 0 and (currentTime - self.lastSwingTime) >= self.config.cooldownSeconds
end

return table.freeze(WeaponInstance)
```

## 3. Tabla de Reemplazo de APIs Obsoletas (Errores Frecuentes de Entrenamiento IA)

| API Deprecada / Obsoleta (PROHIBIDA) | API Moderna Oficial en Roblox / Luau | Justificación Técnica en el Motor |
| :--- | :--- | :--- |
| `wait(n)` | `task.wait(n)` | `wait()` opera sobre el planificador heredado con estrangulamiento artificial; `task.wait()` reanuda en el primer fotograma exacto del Task Scheduler. |
| `spawn(f)` / `delay(n, f)` | `task.spawn(f)` / `task.defer(f)` / `task.delay(n, f)` | `task.spawn` ejecuta el hilo inmediatamente sin latencia de un fotograma; `task.defer` programa la ejecución de forma segura en el siguiente punto de reanudación. |
| `Part.Velocity` / `Part.RotVelocity` | `Part.AssemblyLinearVelocity` / `Part.AssemblyAngularVelocity` | Las propiedades antiguas fueron deprecadas en favor del sistema unificado de ensamblajes rígidos (Assemblies). |
| `BodyVelocity`, `BodyPosition`, `BodyGyro` | `LinearVelocity`, `AlignPosition`, `AlignOrientation` | Los antiguos BodyMovers están deprecados; el motor físico moderno utiliza exclusivamente Constraints basadas en Attachments. |
| `workspace:FindPartOnRay()` | `workspace:Raycast(origin, dir, params)` | `Raycast` con un objeto `RaycastParams` prealojado es significativamente más rápido y limpio en memoria. |
| `Humanoid:LoadAnimation(anim)` | `Animator:LoadAnimation(anim)` | Cargar animaciones sobre el `Humanoid` está deprecado; se debe cargar sobre la instancia hija `Animator` (o dentro de un `AnimationController` para NPCs). |
| `RunService.Stepped` / `RenderStepped` | `RunService.PreSimulation` / `PostSimulation` / `PreRender` | El Task Scheduler moderno define explícitamente qué ocurre antes de la física (`PreSimulation`), después de la física (`PostSimulation`) y antes del renderizado (`PreRender`). |

---
**Conexiones del Vault:** [[00_Mapa_del_Vault]] | [[CLAUDE]] | [[01_API_Roblox_y_Servicios]] | [[04_Patrones_y_Estructura_Rojo]] | [[09_Errores_Comunes_IA_Roblox]]

---
**Ver también (enciclopedia):** [[Luau Language Essentials|Luau Essentials]], [[Advanced Types - Generics and Type Functions|Advanced Types]], [[Cookbook - Tables Strings and Time|Luau Cookbook]]
