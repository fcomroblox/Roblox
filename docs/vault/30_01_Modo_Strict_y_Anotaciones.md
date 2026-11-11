---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: critica
ultima_verificacion: 2026-04-09
---

# 30.01 Modo Strict (`--!strict`) y Anotaciones de Tipos en Luau

> **Cuándo leer este nodo:** Al crear cualquier archivo `.luau` nuevo o al corregir errores reportados por `luau-lsp analyze` en la línea de comandos.

## 1. Reglas duras
- **HACER:** Colocar `--!strict` exactamente en la línea 1 de todo archivo `.luau` en `src/` y `tests/`.
- **HACER:** Anotar explícitamente los tipos de todos los parámetros de funciones, el tipo de retorno de funciones públicas y las tablas vacías (`local cache: { [Player]: number } = {}`).
- **HACER:** Exportar con `export type` los tipos de dominio que otros módulos necesiten consumir al hacer `require()`.
- **NO HACER:** Usar `--!nonstrict` o `--!nocheck` en código del proyecto.
- **NO HACER:** Usar el casteo `:: any` para silenciar advertencias del analizador de tipos en `src/` (solo permitido de forma aislada en mocks de `tests/`).

## 2. API y firmas verificadas

| Elemento Sintáctico | Firma / Patrón en Luau | Contexto | Fuente |
| :--- | :--- | :--- | :--- |
| Directiva de modo estricto | `--!strict` (primera línea del archivo) | Servidor y Cliente | `https://luau.org/typecheck` |
| Tipo opcional (Nullable) | `T?` (equivalente exacto a `T \| nil`) | Servidor y Cliente | `https://luau.org/typecheck` |
| Tipo de diccionario (Mapa) | `{ [K]: V }` (ej. `{ [Player]: number }`) | Servidor y Cliente | `https://luau.org/typecheck` |
| Tipo de array secuencial | `{ T }` (ej. `{ BasePart }`) | Servidor y Cliente | `https://luau.org/typecheck` |
| Tipo de función con nombres | `(player: Player, amount: number) -> boolean` | Servidor y Cliente | `https://luau.org/typecheck` |
| Retorno múltiple / vacío | `() -> (boolean, string?)` / `() -> ()` | Servidor y Cliente | `https://luau.org/typecheck` |
| Tipo cumbre seguro | `unknown` (requiere refinamiento antes de operar) | Servidor y Cliente | `https://luau.org/typecheck` |
| Tipo fondo (sin retorno) | `never` (funciones que lanzan `error()` siempre) | Servidor y Cliente | `https://luau.org/typecheck` |
| Singleton literal | `"Tool" \| "Backpack"` o `true` | Servidor y Cliente | `https://luau.org/typecheck` |

## 3. Patrón recomendado

~~~luau
--!strict
export type UpgradeCategory = "Tool" | "Backpack"

export type TransactionResult = {
	success: boolean,
	newBalance: number,
	errorMessage: string?,
}

export type PriceCalculator = (basePrice: number, currentTier: number) -> number

local ShopPricing = {}

-- Tabla vacía con tipo de clave y valor explícito (evita inferencia never)
local lastPurchaseTimeByUserId: { [number]: number } = {}

local calculateExponentialPrice: PriceCalculator = function(basePrice: number, currentTier: number): number
	return math.floor(basePrice * (1.55 ^ (currentTier - 1)))
end

function ShopPricing.GetUpgradePrice(category: UpgradeCategory, currentTier: number): number
	local base = if category == "Tool" then 100 else 150
	return calculateExponentialPrice(base, currentTier)
end

function ShopPricing.RecordPurchaseTimestamp(userId: number, timestamp: number): ()
	lastPurchaseTimeByUserId[userId] = timestamp
end

function ShopPricing.GetLastPurchaseTimestamp(userId: number): number?
	return lastPurchaseTimeByUserId[userId]
end

return table.freeze(ShopPricing)
~~~

## 4. Errores comunes y antipatrones
- **Inicializar tablas vacías sin anotación (`local items = {}`):** En modo `--!strict`, si una tabla nace vacía y no tiene anotación, Luau infiere `{ [never]: never }` o la sella con la primera inserción, provocando `TypeError` al insertar elementos opcionales o de subtipos distintos después.
- **Usar `any` en argumentos de eventos de red:** Desactiva por completo el comprobador de tipos en cascada. En `RemoteEvent.OnServerEvent`, anota siempre los parámetros entrantes como `unknown` (ver [[30_03_Refinamiento_de_Tipos_y_Typeof]]).
- **Omitir el retorno `()` o tipo de retorno en funciones recursivas:** Sin tipo de retorno explícito, `luau-lsp` puede fallar al inferir llamadas recursivas mutuas.
- **Olvidar paréntesis al combinar tipos función con `?`:** Escribir `() -> boolean?` significa función que devuelve `boolean | nil`. Para que la función en sí sea opcional, escribe `(() -> boolean)?`.

## 5. Cómo comprobar que está bien
Ejecutar desde la raíz del proyecto en terminal:
1. `rojo sourcemap default.project.json --output sourcemap.json`
2. `luau-lsp analyze --sourcemap=sourcemap.json src/` (debe devolver $0$ errores de tipo).
3. `selene src/` y `stylua --check src/`.

## 6. Fuentes
- Luau Official Documentation — Typechecking: `https://luau.org/typecheck` (Consultado: 2026-04-09).
- Roblox Creator Docs — Luau Typechecking (`llms.txt`): `https://create.roblox.com/docs/luau/type-checking` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[30_02_Tipos_Genericos_y_Type_Packs]] | [[30_03_Refinamiento_de_Tipos_y_Typeof]] | [[39_04_Luau_LSP_Sourcemap_y_Analisis_CLI]]

---
**Ver también (enciclopedia):** [[Luau Language Essentials|Luau Essentials]], [[Advanced Types - Generics and Type Functions|Advanced Types]]
