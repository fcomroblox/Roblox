---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: critica
ultima_verificacion: 2026-04-09
---

# 30.05 Programación Orientada a Objetos (POO) con Metatables Tipadas en `--!strict`

> **Cuándo leer este nodo:** Al implementar clases con estado por instancia (nodos de recursos activos, sesiones de submarinos, controladores de componentes o máquinas de estado) compatibles con `luau-lsp`.

## 1. Reglas duras
- **HACER:** Separar el tipo de los datos de estado (`type ClassData = { ... }`) de la tabla de métodos (`local Class = {}; Class.__index = Class`) y exportar el tipo combinado con `export type Class = typeof(setmetatable({} :: ClassData, Class))`.
- **HACER:** Anotar explícitamente `self: Class` como primer parámetro en todos los métodos o definir los métodos con sintaxis de punto `function Class.Method(self: Class, ...)` para evitar ambigüedades de inferencia en `--!strict`.
- **HACER:** Incluir un método `Destroy(self: Class): ()` que limpie su `Trove` interno en cualquier clase que conecte señales o cree instancias.
- **NO HACER:** Usar herencia profunda con cadenas de metatables de 3 o más niveles (rompe la inferencia de tipos de Luau y degrada el caché de búsqueda de campos en la VM). Prefiere composición sobre herencia.
- **NO HACER:** Definir closures (funciones anónimas nuevas) dentro de `.new()` por cada instancia creada; todos los métodos deben vivir en la tabla compartida `Class`.

## 2. API y firmas verificadas

| Elemento del Patrón POO | Construcción Verificada en Luau | Contexto | Fuente |
| :--- | :--- | :--- | :--- |
| Tabla de clase y redirección | `local Class = {}; Class.__index = Class` | Servidor y Cliente | `https://luau.org/typecheck#type-inferencing-for-oop-programs` |
| Tipo de instancia exportado | `export type Class = typeof(setmetatable({} :: ClassData, Class))` | Servidor y Cliente | `https://create.roblox.com/docs/luau/type-checking` |
| Constructor tipado | `function Class.new(...): Class` | Servidor y Cliente | `https://luau.org/typecheck` |
| Método de instancia tipado | `function Class.TakeDamage(self: Class, amount: number): boolean` | Servidor y Cliente | `https://luau.org/typecheck` |
| Congelación de tabla de clase | `return table.freeze(Class)` (se congela la clase, NO la instancia `self`) | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/libraries/table#freeze` |

## 3. Patrón recomendado

~~~luau
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Trove = require(ReplicatedStorage.Packages.Trove)

local OxygenTank = {}
OxygenTank.__index = OxygenTank

type OxygenTankData = {
	_trove: typeof(Trove.new()),
	_maxOxygen: number,
	_currentOxygen: number,
	_depletionRatePerSec: number,
	_isDepleted: boolean,
}

export type OxygenTank = typeof(setmetatable({} :: OxygenTankData, OxygenTank))

function OxygenTank.new(maxOxygen: number, depletionRatePerSec: number): OxygenTank
	assert(maxOxygen > 0, "maxOxygen debe ser mayor que 0")
	assert(depletionRatePerSec >= 0, "depletionRatePerSec no puede ser negativo")

	local self: OxygenTankData = {
		_trove = Trove.new(),
		_maxOxygen = maxOxygen,
		_currentOxygen = maxOxygen,
		_depletionRatePerSec = depletionRatePerSec,
		_isDepleted = false,
	}

	return setmetatable(self, OxygenTank)
end

function OxygenTank.Step(self: OxygenTank, deltaTime: number): number
	if self._isDepleted or deltaTime <= 0 then
		return self._currentOxygen
	end

	self._currentOxygen = math.max(0, self._currentOxygen - (self._depletionRatePerSec * deltaTime))
	if self._currentOxygen == 0 then
		self._isDepleted = true
	end
	return self._currentOxygen
end

function OxygenTank.Refill(self: OxygenTank, amount: number): ()
	if amount <= 0 then
		return
	end
	self._currentOxygen = math.min(self._maxOxygen, self._currentOxygen + amount)
	self._isDepleted = self._currentOxygen == 0
end

function OxygenTank.GetRatio(self: OxygenTank): number
	return self._currentOxygen / self._maxOxygen
end

function OxygenTank.Destroy(self: OxygenTank): ()
	self._trove:Clean()
end

return table.freeze(OxygenTank)
~~~

## 4. Errores comunes y antipatrones
- **Llamar a un método con punto en lugar de dos puntos (`tank.Step(0.1)` en vez de `tank:Step(0.1)`):** En las llamadas del consumidor se debe usar `:` para pasar `self` implícitamente. Gracias a `self: OxygenTank`, `luau-lsp` detectará en tiempo de compilación si olvidas los dos puntos `:`.
- **Llamar a `table.freeze(self)` en el constructor de una clase con estado mutable:** Si congelas `self`, cualquier intento posterior de hacer `self._currentOxygen = ...` lanzará un error fatal en tiempo de ejecución. Solo se congela la tabla estática de la clase al final del módulo (`return table.freeze(OxygenTank)`).
- **Añadir propiedades nuevas a `self` fuera de `ClassData` en métodos posteriores:** En `--!strict`, todas las propiedades que una instancia pueda tener durante su vida deben estar declaradas en `ClassData` (como `T?` si empiezan en `nil`).

## 5. Cómo comprobar que está bien
1. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/`.
2. Ejecutar un test unitario con Lune (`lune run tests/...`) que instancie `OxygenTank.new(100, 5)`, llame a `tank:Step(10)` y verifique que `tank:GetRatio() == 0.5`.

## 6. Fuentes
- Luau Official Documentation — Type Inferencing for OOP Programs: `https://luau.org/typecheck` (Consultado: 2026-04-09).
- Roblox Creator Docs — Metatables & Luau Classes: `https://create.roblox.com/docs/luau/metatables` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[04_Patrones_y_Estructura_Rojo]] | [[30_01_Modo_Strict_y_Anotaciones]] | [[32_04_Patron_Trove_Limpieza_Determinista]]

---
**Ver también (enciclopedia):** [[Luau Language Essentials|Luau POO]]
