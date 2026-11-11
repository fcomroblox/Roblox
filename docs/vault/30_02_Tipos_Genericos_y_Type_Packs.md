---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: alta
ultima_verificacion: 2026-04-09
---

# 30.02 Tipos Genéricos (`<T>`) y Type Packs (`<T...>`) en Luau

> **Cuándo leer este nodo:** Al escribir contenedores reutilizables (Object Pools, colas, caches, Signals, Promises o envoltorios de llamadas `pcall`) que deban preservar el tipo exacto de sus elementos sin usar `any`.

## 1. Reglas duras
- **HACER:** Declarar parámetros de tipo genérico `<T>` en tipos exportados y funciones de utilidad que operen sobre colecciones o callbacks arbitrarios.
- **HACER:** Usar *Variadic Type Packs* (`T...`, `U...`) con tres puntos al tipar funciones que reenvían argumentos variables (`...`) o retornos múltiples (como `Signal:Fire(T...)` o wrappers de `task.spawn`).
- **HACER:** Colocar los Type Packs (`T...`) siempre después de los tipos individuales (`T`, `K`, `V`) en la lista de parámetros genéricos: `export type Wrapper<T, Args..., Ret...> = ...`.
- **NO HACER:** Usar `{ any }` para representar arrays genéricos ni `(...any) -> ...any` cuando el contrato de entrada/salida pueda parametrizarse con `<Args..., Ret...>`.

## 2. API y firmas verificadas

| Construcción Genérica | Sintaxis Verificada en Luau | Contexto | Fuente |
| :--- | :--- | :--- | :--- |
| Tipo de datos genérico | `export type Result<T, E> = ...` | Servidor y Cliente | `https://luau.org/typecheck#generics` |
| Función genérica | `function head<T>(list: { T }): T?` | Servidor y Cliente | `https://luau.org/typecheck#generics` |
| Type Pack (paquete variádico) | `export type Callback<T...> = (T...) -> ()` | Servidor y Cliente | `https://luau.org/typecheck#type-packs` |
| Valor por defecto en genérico | `export type Response<T = string, P... = ...number> = ...` | Servidor y Cliente | `https://luau.org/typecheck#type-packs` |
| Anotación de `...` en función | `function call<A..., R...>(fn: (A...) -> R..., ...: A...): R...` | Servidor y Cliente | `https://luau.org/typecheck#type-packs` |

## 3. Patrón recomendado

~~~luau
--!strict
-- 1. Contenedor de resultado tipado sin excepciones
export type Result<T, E = string> = {
	ok: true,
	value: T,
} | {
	ok: false,
	err: E,
}

-- 2. Tipo Signal fuertemente tipado mediante Type Pack (T...)
export type Connection = {
	Disconnect: (self: Connection) -> (),
}

export type TypedSignal<T...> = {
	Connect: (self: TypedSignal<T...>, callback: (T...) -> ()) -> Connection,
	Fire: (self: TypedSignal<T...>, T...) -> (),
}

local GenericUtils = {}

-- 3. Envoltorio genérico para medir tiempo de ejecución preservando argumentos (A...) y retorno (R...)
function GenericUtils.MeasureExecution<A..., R...>(
	label: string,
	fn: (A...) -> R...,
	...: A...
): (number, R...)
	debug.profilebegin(label)
	local startClock = os.clock()
	local result: R... = fn(...)
	local elapsedSeconds = os.clock() - startClock
	debug.profileend()
	return elapsedSeconds, result
end

-- 4. Filtro genérico para arrays tipados
function GenericUtils.FilterArray<T>(source: { T }, predicate: (item: T, index: number) -> boolean): { T }
	local output: { T } = {}
	for index, item in source do
		if predicate(item, index) then
			table.insert(output, item)
		end
	end
	return output
end

return table.freeze(GenericUtils)
~~~

## 4. Errores comunes y antipatrones
- **Escribir `...: T` en vez de `...: T...`:** En la firma de una función variádica, si declaras un Type Pack `<T...>`, el parámetro variádico debe anotarse como `...: T...`. Si declaras un tipo simple `<T>` y escribes `...: T`, todos los argumentos extra deberán ser del mismo tipo `T`.
- **Omitir los paréntesis alrededor de un Type Pack cuando es retorno único:** Una función que devuelve un Type Pack debe escribirse `(A...) -> (R...)` o `(A...) -> R...`, pero al anidarla dentro de una unión o tipo opcional debe ir entre paréntesis: `((A...) -> (R...))?`.
- **Intentar usar restricciones de herencia en genéricos (`<T: Instance>`):** Luau no soporta sintaxis de acotación (`bounds` tipo TypeScript `T extends Instance`). Para operar sobre métodos de `Instance`, pasa `Instance` o el tipo concreto directamente, o aplica refinamiento con `typeof`.

## 5. Cómo comprobar que está bien
1. Crear un caso de uso llamando a `GenericUtils.FilterArray({ 10, 20, 30 }, function(n) return n > 15 end)` y asignar el resultado a `local res: { string }` — `luau-lsp analyze` debe reportar error de incompatibilidad (`{ number }` no convertible a `{ string }`).
2. Ejecutar `luau-lsp analyze --sourcemap=sourcemap.json src/` y confirmar $0$ advertencias.

## 6. Fuentes
- Luau Official Documentation — Generics and Type Packs: `https://luau.org/typecheck#generics` (Consultado: 2026-04-09).
- Roblox Creator Docs — Luau Typechecking Generics: `https://create.roblox.com/docs/luau/type-checking#generics` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[02_Guia_Luau_Tipado_Estricto]] | [[30_01_Modo_Strict_y_Anotaciones]] | [[32_05_Patron_Signal_Eventos_Tipados]] | [[32_10_Object_Pooling_Instancias_y_Tablas]]

---
**Ver también (enciclopedia):** [[Advanced Types - Generics and Type Functions|Advanced Types]]
