---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: critica
ultima_verificacion: 2026-04-09
aliases: [30_03_Refinamiento_de_Tipos_y_Typeof]
---

# 30.03 Refinamiento de Tipos (`Type Narrowing`), `typeof()` e `IsA()`

> **Cuándo leer este nodo:** Al validar argumentos `unknown` recibidos por `RemoteEvent`, al buscar hijos en el DataModel (`FindFirstChild`) o al convertir tipos opcionales (`T?`) en tipos seguros (`T`) sin usar `:: any`.

## 1. Reglas duras
- **HACER:** Usar cláusulas de guarda tempranas (`if not x then return end`) para estrechar tipos opcionales `T?` a `T` en el resto de la función.
- **HACER:** Usar `typeof(val) == "..."` para tipos de Roblox (`"Instance"`, `"Vector3"`, `"CFrame"`, `"Color3"`, `"EnumItem"`, `"buffer"`) y primitivos (`"number"`, `"string"`, `"boolean"`, `"table"`).
- **HACER:** Combinar `typeof(val) == "Instance" and val:IsA("Clase")` para estrechar un `unknown` o `Instance` a una subclase exacta del motor (`Model`, `BasePart`, `Humanoid`, `Attachment`).
- **NO HACER:** Llamar a `:IsA()` sobre un valor `unknown` sin comprobar primero `typeof(val) == "Instance"` (provoca error en tiempo de ejecución si el atacante envía una tabla o número).
- **NO HACER:** Releer una propiedad mutable de una instancia o tabla global después de comprobarla (`if model.PrimaryPart then model.PrimaryPart.CFrame = ...`); asígnala primero a una variable local inmutable (`local primary = model.PrimaryPart`).

## 2. API y firmas verificadas

| Mecanismo de Refinamiento | Expresión en Luau | Tipo Antes $\rightarrow$ Tipo Después | Contexto | Fuente |
| :--- | :--- | :--- | :--- | :--- |
| Prueba de veracidad | `if x then` / `assert(x)` | `T?` $\rightarrow$ `T` | Servidor y Cliente | `https://luau.org/typecheck#type-refinements` |
| Prueba de tipo Roblox | `if typeof(x) == "Vector3" then` | `unknown` $\rightarrow$ `Vector3` | Servidor y Cliente | `https://create.roblox.com/docs/luau/type-checking` |
| Prueba de clase `Instance` | `if inst:IsA("BasePart") then` | `Instance` $\rightarrow$ `BasePart` | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/classes/Instance#IsA` |
| Búsqueda tipada por clase | `parent:FindFirstChildOfClass("Humanoid")` | Retorno directo: `Humanoid?` | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/classes/Instance#FindFirstChildOfClass` |
| Búsqueda por herencia | `parent:FindFirstChildWhichIsA("BasePart")` | Retorno directo: `BasePart?` | Servidor y Cliente | `https://create.roblox.com/docs/reference/engine/classes/Instance#FindFirstChildWhichIsA` |

## 3. Patrón recomendado

~~~luau
--!strict
local CollectionService = game:GetService("CollectionService")

local NodeValidator = {}

-- Estrecha un valor unknown de red a un Model validado con su BasePart raíz
function NodeValidator.ValidateResourceNode(
	rawCandidate: unknown,
	requiredTag: string
): (Model?, BasePart?, number?)
	-- 1. Refinar de unknown a Instance
	if typeof(rawCandidate) ~= "Instance" then
		return nil, nil, nil
	end

	-- 2. Refinar de Instance a Model mediante IsA
	if not rawCandidate:IsA("Model") then
		return nil, nil, nil
	end

	if not CollectionService:HasTag(rawCandidate, requiredTag) then
		return nil, nil, nil
	end

	-- 3. Extraer a variable local para que el refinamiento de nil sea estable
	local primaryPart = rawCandidate.PrimaryPart or rawCandidate:FindFirstChildWhichIsA("BasePart")
	if not primaryPart then
		return nil, nil, nil
	end

	-- 4. Refinar atributo unknown a number finito
	local rawTier = rawCandidate:GetAttribute("NodeTier")
	if typeof(rawTier) ~= "number" or rawTier ~= rawTier then
		return nil, nil, nil
	end

	return rawCandidate, primaryPart, rawTier
end

return table.freeze(NodeValidator)
~~~

## 4. Errores comunes y antipatrones
- **Refinar directamente sobre una propiedad de tabla/instancia (`if player.Character then player.Character:PivotTo(...)`):** En modo `--!strict`, `player.Character` puede cambiar a `nil` entre líneas o el analizador puede no confiar en propiedades dinámicas. Guarda siempre `local character = player.Character` antes del `if`.
- **Usar `type(v)` en lugar de `typeof(v)` para tipos de Roblox:** `type(Vector3.zero)` y `type(workspace)` devuelven ambos `"userdata"`. Solo `typeof()` distingue `"Vector3"` de `"Instance"` y es reconocido por el motor de refinamiento de Luau para tipos de Roblox.
- **Usar funciones validadoras booleanas personalizadas esperando que estrechen el tipo:** En Luau, `if isModel(x) then` NO estrecha el tipo de `x` de `unknown` a `Model` fuera de la función. Si encapsulas la validación en una función auxiliar, haz que devuelva `Model?` en lugar de `boolean` (como en el patrón recomendado arriba).

## 5. Cómo comprobar que está bien
1. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/`.
2. Verificar con `grep -rn ":: any" src/` que no se haya usado ningún casteo inseguro para esquivar comprobaciones de `nil` o `Instance`.

## 6. Fuentes
- Luau Official Documentation — Type Refinements: `https://luau.org/typecheck#type-refinements` (Consultado: 2026-04-09).
- Roblox Creator Docs — `Instance:IsA` & `FindFirstChildWhichIsA`: `https://create.roblox.com/docs/reference/engine/classes/Instance` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[03_Seguridad_Cliente_Servidor]] | [[30_01_Modo_Strict_y_Anotaciones]] | [[33_02_Pipeline_de_Validacion_de_5_Capas]]

---
**Ver también (enciclopedia):** [[Events Signals and Connections|Events & Signals]], [[Advanced Types - Generics and Type Functions|Advanced Types]]
