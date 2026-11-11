---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: alta
ultima_verificacion: 2026-04-09
---

# 30.09 Librería `table` de Luau: `create`, `clear`, `clone`, `freeze` y `move`

> **Cuándo leer este nodo:** Al definir catálogos de configuración inmutables, al clonar plantillas de estado o al optimizar la memoria en bucles que procesan lotes de datos por fotograma.

## 1. Reglas duras
- **HACER:** Congelar siempre la tabla exportada al final de cada `ModuleScript` y todas las tablas de constantes/catálogos con `table.freeze(t)`.
- **HACER:** Usar `table.create(count, initialValue?)` cuando conozcas el tamaño aproximado de un array antes de llenarlo (ej. decodificar snapshots de red o usar `workspace:BulkMoveTo`).
- **HACER:** Reutilizar tablas de trabajo temporales en bucles de `RunService` vaciándolas con `table.clear(t)` en lugar de crear una tabla nueva `{}` en cada fotograma (`60` veces por segundo).
- **NO HACER:** Asumir que `table.freeze(t)` o `table.clone(t)` son profundos (*deep*): ambos operan **únicamente en el primer nivel** (*shallow*). Si una tabla contiene subtablas, debes congelar o clonar recursivamente.

## 2. API y firmas verificadas

| Función de `table` | Firma Verificada en Luau | Complejidad y Efecto en Memoria | Fuente |
| :--- | :--- | :--- | :--- |
| `table.create` | `table.create<V>(count: number, value: V?): { V }` | Prealoja la parte de array de una sola vez, evitando realojamientos al crecer con `table.insert`. | `https://create.roblox.com/docs/reference/engine/libraries/table#create` |
| `table.clear` | `table.clear(t: { [any]: any }): ()` | Borra todas las claves y valores pero **conserva la capacidad de memoria alocada** para reutilizar la tabla sin presionar al GC. | `https://create.roblox.com/docs/reference/engine/libraries/table#clear` |
| `table.clone` | `table.clone<T>(t: T): T` | Crea una copia superficial (*shallow copy*) ultrarrápida con la misma capacidad y metatable (si no tiene `__metatable` protegido). | `https://create.roblox.com/docs/reference/engine/libraries/table#clone` |
| `table.freeze` | `table.freeze<T>(t: T): T` | Marca la tabla como de solo lectura en primer nivel; cualquier escritura futura lanza `error` en tiempo de ejecución. | `https://create.roblox.com/docs/reference/engine/libraries/table#freeze` |
| `table.isfrozen` | `table.isfrozen(t: { [any]: any }): boolean` | Devuelve `true` si la tabla fue congelada con `table.freeze`. | `https://create.roblox.com/docs/reference/engine/libraries/table#isfrozen` |
| `table.move` | `table.move<V>(a: { V }, f: number, e: number, t: number, dst: { V }?): { V }` | Copia en bloque en C los elementos del rango `[f, e]` de `a` hacia el índice `t` de `dst`. | `https://create.roblox.com/docs/reference/engine/libraries/table#move` |

## 3. Patrón recomendado

~~~luau
--!strict
local TableHelpers = {}

-- Congelación profunda recursiva para catálogos de configuración (armas, mejoras, precios)
function TableHelpers.DeepFreeze<T>(rootTable: T): T
	assert(typeof(rootTable) == "table", "DeepFreeze requiere una tabla")

	local function freezeRecursive(tbl: { [unknown]: unknown })
		for _, value in tbl do
			if typeof(value) == "table" and not table.isfrozen(value) then
				freezeRecursive(value :: { [unknown]: unknown })
			end
		end
		if not table.isfrozen(tbl) then
			table.freeze(tbl)
		end
	end

	freezeRecursive(rootTable :: { [unknown]: unknown })
	return rootTable
end

-- Clonación profunda para plantillas de datos (ej. DATA_TEMPLATE de jugador en tests)
function TableHelpers.DeepClone<T>(source: T): T
	if typeof(source) ~= "table" then
		return source
	end

	local copy = table.clone(source :: { [unknown]: unknown })
	for key, value in copy do
		if typeof(value) == "table" then
			copy[key] = TableHelpers.DeepClone(value)
		end
	end

	return (copy :: unknown) :: T
end

-- Ejemplo de reutilización de buffers de tabla sin alocar memoria por fotograma
local scratchParts: { BasePart } = table.create(64)
local scratchCFrames: { CFrame } = table.create(64)

function TableHelpers.BatchMoveNodes(nodes: { BasePart }, targetCFrames: { CFrame }): ()
	table.clear(scratchParts)
	table.clear(scratchCFrames)

	local count = math.min(#nodes, #targetCFrames)
	table.move(nodes, 1, count, 1, scratchParts)
	table.move(targetCFrames, 1, count, 1, scratchCFrames)

	workspace:BulkMoveTo(scratchParts, scratchCFrames, Enum.BulkMoveMode.FireCFrameChanged)
end

return table.freeze(TableHelpers)
~~~

## 4. Errores comunes y antipatrones
- **Confiar en `table.freeze(CATALOG)` cuando `CATALOG` contiene diccionarios anidados:** Si `CATALOG = table.freeze({ Tool = { BaseCost = 100 } })`, un bug accidental que escriba `CATALOG.Tool.BaseCost = 0` **NO fallará**, porque la subtabla `Tool` nunca fue congelada. Usa siempre `DeepFreeze` en configuraciones anidadas.
- **Usar `table.clone(DATA_TEMPLATE)` para inicializar datos con subtablas (`stats`, `unlockedItems`):** Con `table.clone` superficial, todos los jugadores compartirían la misma referencia a la subtabla `stats` en memoria. Usa `DeepClone` (o `profile:Reconcile()` de `ProfileStore`, que ya clona en profundidad).
- **Llamar a `table.freeze` dos veces sobre la misma tabla sin comprobar `table.isfrozen`:** En Luau, llamar a `table.freeze` sobre una tabla que ya está congelada lanza un error en tiempo de ejecución (`cannot freeze a table more than once`).

## 5. Cómo comprobar que está bien
1. Ejecutar test en Lune verificando que `pcall(function() frozenCatalog.SubItem.Price = 999 end)` devuelve `false` tras aplicar `DeepFreeze`.
2. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/`.

## 6. Fuentes
- Roblox Creator Docs — `table` Library: `https://create.roblox.com/docs/reference/engine/libraries/table` (Consultado: 2026-04-09).
- Luau Official Documentation — Sandboxing & `table.freeze`: `https://luau.org/sandbox` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[30_08_Iteracion_Generalizada_y_Tablas]] | [[32_09_Modulos_de_Configuracion_y_Catalogos_Inmutables]] | [[38_07_NPCs_Sin_Humanoid_y_Workspace_BulkMoveTo]]

---
**Ver también (enciclopedia):** [[Buffers and Binary Data|Data Structures]]
