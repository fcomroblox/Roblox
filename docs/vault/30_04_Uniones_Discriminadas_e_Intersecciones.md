---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: alta
ultima_verificacion: 2026-04-09
---

# 30.04 Uniones Discriminadas (`Tagged Unions`) e Intersecciones (`&`)

> **Cuándo leer este nodo:** Al modelar payloads de red con múltiples variantes, estados de una máquina de estados (FSM), acciones de UI o sobrecargas de funciones en `--!strict`.

## 1. Reglas duras
- **HACER:** Incluir un campo discriminante con un tipo literal de cadena o booleano (ej. `kind: "RewardCoins"` vs `kind: "RewardItem"`) en cada variante de una unión de tablas (`|`).
- **HACER:** Comprobar el campo discriminante (`if action.kind == "RewardCoins" then`) para que Luau estreche automáticamente la tabla a la variante exacta y permita acceder a sus campos exclusivos.
- **HACER:** Usar intersecciones (`&`) para combinar traits/interfaces de tablas o para declarar funciones sobrecargadas (`((id: number) -> string) & ((id: string) -> number)`).
- **NO HACER:** Definir una sola tabla con decenas de campos opcionales (`coins: number?`, `itemId: string?`, `errorCode: string?`) cuando los campos sean mutuamente excluyentes según el estado.

## 2. API y firmas verificadas

| Concepto de Tipo | Sintaxis en Luau | Comportamiento en `--!strict` | Fuente |
| :--- | :--- | :--- | :--- |
| Tipo literal (Singleton) | `"Idle" \| "Extracting" \| "Depositing"` | Solo acepta exactamente una de esas cadenas literales. | `https://luau.org/typecheck#singleton-types` |
| Unión discriminada (`Tagged Union`) | `{ kind: "A", a: number } \| { kind: "B", b: string }` | Al filtrar `if x.kind == "A"`, `x.a` queda disponible como `number`. | `https://luau.org/typecheck#tagged-unions` |
| Intersección de tablas (`&`) | `BaseEntity & { shield: number }` | Exige que la tabla cumpla simultáneamente ambas estructuras. | `https://luau.org/typecheck#intersection-types` |
| Sobrecarga de función con `&` | `((x: number) -> number) & ((x: Vector3) -> Vector3)` | Permite llamar a la función con cualquiera de las firmas listadas. | `https://luau.org/typecheck#intersection-types` |

## 3. Patrón recomendado

~~~luau
--!strict
-- Unión discriminada para recompensas de expedición o cofres
export type CrateReward =
	{
		rewardType: "Currency",
		currencyKey: "coins" | "gems",
		amount: number,
	}
	| {
		rewardType: "UnlockItem",
		itemId: string,
		isDuplicate: boolean,
		fallbackCoins: number,
	}
	| {
		rewardType: "ServerBoost",
		boostMultiplier: number,
		durationSeconds: number,
	}

-- Intersección para añadir metadatos de auditoría a cualquier recompensa
export type AuditedSummary = {
	timestampUnix: number,
	sourceSku: string,
} & {
	reward: CrateReward,
}

local RewardFormatter = {}

function RewardFormatter.FormatNotification(reward: CrateReward): string
	if reward.rewardType == "Currency" then
		-- Aquí Luau sabe que reward.amount y reward.currencyKey existen y NO son nil
		return `+{reward.amount} {reward.currencyKey}`
	elseif reward.rewardType == "UnlockItem" then
		-- Aquí Luau sabe que reward.itemId y reward.isDuplicate existen
		if reward.isDuplicate then
			return `Ítem repetido ({reward.itemId}): +{reward.fallbackCoins} coins`
		end
		return `¡Nuevo desbloqueo: {reward.itemId}!`
	else
		-- Por eliminación exhaustiva, Luau refina a la variante "ServerBoost"
		return `Boost x{reward.boostMultiplier} activo durante {reward.durationSeconds}s`
	end
end

return table.freeze(RewardFormatter)
~~~

## 4. Errores comunes y antipatrones
- **Intentar acceder a un campo no compartido antes de comprobar el discriminante:** Si intentas leer `reward.amount` fuera del `if reward.rewardType == "Currency"`, `luau-lsp` lanzará error porque las variantes `"UnlockItem"` y `"ServerBoost"` no poseen la propiedad `amount`.
- **Usar variables dinámicas como discriminante:** El campo discriminante en la definición del `export type` debe ser un literal exacto (`"Currency"`), no el tipo genérico `string`.
- **Construir una variante asignando campos línea por línea después de crear `{}`:** Construye el literal de tabla completo en una sola expresión (`local r: CrateReward = { rewardType = "Currency", currencyKey = "coins", amount = 50 }`) para que Luau valide la variante completa de inmediato.

## 5. Cómo comprobar que está bien
1. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/`.
2. Añadir temporalmente un acceso inválido como `print(reward.itemId)` dentro de la rama `reward.rewardType == "Currency"` y verificar que `luau-lsp` falla inmediatamente.

## 6. Fuentes
- Luau Official Documentation — Singleton Types and Tagged Unions: `https://luau.org/typecheck#tagged-unions` (Consultado: 2026-04-09).
- Luau Official Documentation — Union and Intersection Types: `https://luau.org/typecheck#union-types` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[30_01_Modo_Strict_y_Anotaciones]] | [[30_03_Refinamiento_de_Tipos_y_Typeof]] | [[32_07_Maquinas_de_Estado_Finito_FSM]]

---
**Ver también (enciclopedia):** [[Advanced Types - Generics and Type Functions|Advanced Types]]
