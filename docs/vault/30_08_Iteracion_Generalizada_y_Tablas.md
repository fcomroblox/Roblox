---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: alta
ultima_verificacion: 2026-04-09
---

# 30.08 Iteración Generalizada (`for k, v in t do`) y Mutación Segura de Tablas

> **Cuándo leer este nodo:** Al recorrer arrays o diccionarios en Luau, al eliminar elementos durante la iteración o al optimizar bucles calientes que se ejecutan en cada fotograma.

## 1. Reglas duras
- **HACER:** Usar siempre iteración generalizada directa `for index, value in array do` o `for key, value in dictionary do`.
- **HACER:** Usar la palabra clave nativa `continue` para saltar a la siguiente iteración en lugar de anidar bloques `if` profundos.
- **HACER:** Recorrer arrays en orden inverso (`for i = #array, 1, -1 do`) cuando necesites eliminar elementos con `table.remove(array, i)` o Swap-Pop durante el recorrido.
- **NO HACER:** Usar `pairs(t)` ni `ipairs(t)` (prohibidos por convención en `CLAUDE.md`; la iteración generalizada de Luau está optimizada a nivel de bytecode en la VM).
- **NO HACER:** Insertar claves nuevas en un diccionario mientras lo estás recorriendo con `for k, v in dict do` (comportamiento indefinido de rehash en Lua/Luau que puede saltar elementos o entrar en bucle).

## 2. API y firmas verificadas

| Construcción de Bucle | Sintaxis en Luau | Complejidad / Garantía | Fuente |
| :--- | :--- | :--- | :--- |
| Iteración generalizada en array | `for i, item in array do` | Orden garantizado $1 \dots \#\text{array}$ para la parte de array contigua. | `https://luau.org/syntax#generalized-iteration` |
| Iteración generalizada en mapa | `for key, val in dict do` | Recorre todas las claves no `nil`; el orden de claves hash no está garantizado. | `https://luau.org/syntax#generalized-iteration` |
| Salto de iteración | `continue` | Salta inmediatamente al siguiente paso del bucle `for`, `while` o `repeat`. | `https://luau.org/syntax#continue-statement` |
| Eliminación en diccionario in-place | `dict[key] = nil` (dentro de `for key, val in dict`) | Seguro en Luau: borrar (`= nil`) la clave actual durante la iteración es válido; añadir claves nuevas NO lo es. | `https://luau.org/performance` |
| Eliminación $O(1)$ en array sin orden | `array[i] = array[#array]; array[#array] = nil` | Evita el desplazamiento de memoria $O(N)$ de `table.remove(array, i)`. | `https://luau.org/performance` |

## 3. Patrón recomendado

~~~luau
--!strict
export type ActiveProjectile = {
	id: number,
	position: Vector3,
	velocity: Vector3,
	remainingLifeSeconds: number,
}

local ProjectileUpdater = {}

-- Actualiza proyectiles activos y elimina los expirados en O(1) por baja usando bucle inverso + Swap-Pop
function ProjectileUpdater.StepProjectiles(projectiles: { ActiveProjectile }, dt: number): ()
	for i = #projectiles, 1, -1 do
		local proj = projectiles[i]
		proj.remainingLifeSeconds -= dt

		if proj.remainingLifeSeconds <= 0 then
			-- Swap-Pop O(1): mover el último elemento a la posición actual y truncar el final
			local lastIndex = #projectiles
			projectiles[i] = projectiles[lastIndex]
			projectiles[lastIndex] = nil
			continue
		end

		proj.position += proj.velocity * dt
	end
end

-- Limpieza segura de entradas expiradas en un diccionario de cooldowns
function ProjectileUpdater.PurgeExpiredCooldowns(cooldownsByUserId: { [number]: number }, nowClock: number): ()
	for userId, expiryClock in cooldownsByUserId do
		if nowClock >= expiryClock then
			-- Asignar nil a una clave existente durante la iteración es seguro
			cooldownsByUserId[userId] = nil
		end
	end
end

return table.freeze(ProjectileUpdater)
~~~

## 4. Errores comunes y antipatrones
- **Llamar a `table.remove(list, i)` dentro de un bucle ascendente `for i, v in list do`:** Al borrar el índice `i`, todos los elementos posteriores se desplazan a `i - 1`, por lo que el elemento que estaba en `i + 1` nunca es evaluado en la siguiente iteración.
- **Usar `#dictionary` para contar elementos de una tabla con claves string o `[Player]`:** El operador de longitud `#t` solo cuenta la parte de array entero contiguo ($1 \dots N$). En un diccionario `{ [Player]: Profile }`, `#t` devuelve siempre $0$. Si necesitas el conteo, mantén un contador entero incremental o cuenta en un bucle.
- **Depender del orden de `for k, v in dict do` para lógica determinista o UI:** El orden de iteración sobre claves string es arbitrario. Si muestras ítems de una tienda en un `ScrollingFrame`, ordena primero las claves en un array con `table.sort` o usa `LayoutOrder` numérico explícito.

## 5. Cómo comprobar que está bien
1. Buscar usos prohibidos con `grep -rnE "\b(pairs|ipairs)\s*\(" src/` (debe devolver $0$ coincidencias).
2. Ejecutar `selene src/` y `luau-lsp analyze --sourcemap=sourcemap.json src/`.

## 6. Fuentes
- Luau Official Documentation — Generalized Iteration & `continue`: `https://luau.org/syntax#generalized-iteration` (Consultado: 2026-04-09).
- Luau Official Documentation — Performance Guide (Table Iteration): `https://luau.org/performance` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[30_09_Libreria_Table_Prealojamiento_y_Freeze]] | [[32_10_Object_Pooling_Instancias_y_Tablas]] | [[40_02_Fugas_de_Memoria_en_Tablas_Indexadas_por_Player]]

---
**Ver también (enciclopedia):** [[Cookbook - Tables Strings and Time|Tables Cookbook]]
