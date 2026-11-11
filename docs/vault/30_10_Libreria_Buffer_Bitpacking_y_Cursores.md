---
tags: [area/30_Luau_Lenguaje_y_Tipado, consumidor/claude]
importancia: alta
ultima_verificacion: 2026-04-09
---

# 30.10 Librería `buffer` de Luau:Cursores, Bitpacking y Límites de Offset

> **Cuándo leer este nodo:** Al escribir serializadores binarios para red (`RemoteEvent` / `UnreliableRemoteEvent`) o compresión de bloques de datos para `DataStore` y `MemoryStore`.

## 1. Reglas duras
- **HACER:** Recordar que todos los offsets de la librería `buffer` están **indexados desde $0$** (`0` a `buffer.len(b) - 1`), a diferencia de los arrays de Luau que empiezan en `1`, y se almacenan en orden **Little-Endian**.
- **HACER:** Validar siempre la longitud total de un `buffer` recibido por red desde el cliente (`typeof(b) == "buffer" and buffer.len(b) == EXPECTED_SIZE`) antes de llamar a `buffer.read*` para evitar que un paquete truncado lance excepción de fuera de rango (`buffer access out of bounds`).
- **HACER:** Usar la librería nativa `bit32` (`bit32.band`, `bit32.bor`, `bit32.lshift`, `bit32.rshift`, `bit32.extract`, `bit32.replace`) para empaquetar hasta $8$ banderas booleanas en un único byte (`u8`).
- **NO HACER:** Escribir números fuera de rango sin `math.clamp` o comprobación previa, ya que las funciones `buffer.writeu8/u16/i16` aplican desbordamiento modular (wrap-around) de entero en complemento a dos.

## 2. API y firmas verificadas

| Función de `buffer` / `bit32` | Firma Verificada | Bytes / Rango Exacto | Fuente |
| :--- | :--- | :--- | :--- |
| `buffer.create` / `buffer.len` | `buffer.create(size: number): buffer` / `buffer.len(b: buffer): number` | Máximo $1\text{ GiB}$ ($1.073.741.824\text{ bytes}$). | `https://luau.org/library#buffer-library` |
| `buffer.readu8` / `writeu8` | `(b: buffer, offset: number, val: number)` | $1\text{ byte}$: $[0, 255]$ | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readi8` / `writei8` | `(b: buffer, offset: number, val: number)` | $1\text{ byte}$: $[-128, 127]$ | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readu16` / `writeu16` | `(b: buffer, offset: number, val: number)` | $2\text{ bytes}$: $[0, 65.535]$ | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readi16` / `writei16` | `(b: buffer, offset: number, val: number)` | $2\text{ bytes}$: $[-32.768, 32.767]$ | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readu32` / `writeu32` | `(b: buffer, offset: number, val: number)` | $4\text{ bytes}$: $[0, 4.294.967.295]$ | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readf32` / `writef32` | `(b: buffer, offset: number, val: number)` | $4\text{ bytes}$: IEEE 754 precisión simple ($\approx 7$ dígitos). | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readf64` / `writef64` | `(b: buffer, offset: number, val: number)` | $8\text{ bytes}$: IEEE 754 doble precisión (`number` nativo). | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.readstring` / `writestring` | `readstring(b, offset, count)` / `writestring(b, offset, s, count?)` | Lee/escribe `count` bytes crudos de un `string`. | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `buffer.copy` / `buffer.fill` | `copy(dst, dstOff, src, srcOff?, count?)` / `fill(b, off, val, count?)` | Copia o rellena bloques de memoria en C. | `https://create.roblox.com/docs/reference/engine/libraries/buffer` |
| `bit32.extract` / `replace` | `bit32.extract(n, field, width?)` / `bit32.replace(n, v, field, width?)` | Extrae o reemplaza bits `[field, field + width - 1]` ($0\dots 31$). | `https://create.roblox.com/docs/reference/engine/libraries/bit32` |

## 3. Patrón recomendado

~~~luau
--!strict
-- Cursor lector/escritor seguro con empaquetado de banderas de bits (bitpacking)
local BufferCursor = {}
BufferCursor.__index = BufferCursor

type CursorData = {
	rawBuffer: buffer,
	offset: number,
	length: number,
}

export type BufferCursor = typeof(setmetatable({} :: CursorData, BufferCursor))

function BufferCursor.fromBuffer(buf: buffer): BufferCursor
	local self: CursorData = {
		rawBuffer = buf,
		offset = 0,
		length = buffer.len(buf),
	}
	return setmetatable(self, BufferCursor)
end

function BufferCursor.CanRead(self: BufferCursor, byteCount: number): boolean
	return (self.offset + byteCount) <= self.length
end

function BufferCursor.ReadU8(self: BufferCursor): number?
	if not self:CanRead(1) then
		return nil
	end
	local val = buffer.readu8(self.rawBuffer, self.offset)
	self.offset += 1
	return val
end

function BufferCursor.ReadShortString(self: BufferCursor): string?
	local strLen = self:ReadU8()
	if not strLen or not self:CanRead(strLen) then
		return nil
	end
	local str = buffer.readstring(self.rawBuffer, self.offset, strLen)
	self.offset += strLen
	return str
end

-- Empaqueta 3 booleanos y un tier (0..15, 4 bits) en un único byte u8
function BufferCursor.PackPlayerFlags(
	isSprinting: boolean,
	flashlightOn: boolean,
	isExtracting: boolean,
	toolTier: number
): number
	local byte = 0
	byte = bit32.replace(byte, if isSprinting then 1 else 0, 0, 1)
	byte = bit32.replace(byte, if flashlightOn then 1 else 0, 1, 1)
	byte = bit32.replace(byte, if isExtracting then 1 else 0, 2, 1)
	byte = bit32.replace(byte, math.clamp(toolTier, 0, 15), 3, 4)
	return byte
end

function BufferCursor.UnpackPlayerFlags(byte: number): (boolean, boolean, boolean, number)
	local isSprinting = bit32.extract(byte, 0, 1) == 1
	local flashlightOn = bit32.extract(byte, 1, 1) == 1
	local isExtracting = bit32.extract(byte, 2, 1) == 1
	local toolTier = bit32.extract(byte, 3, 4)
	return isSprinting, flashlightOn, isExtracting, toolTier
end

return table.freeze(BufferCursor)
~~~

## 4. Errores comunes y antipatrones
- **Empezar el offset en `1` por costumbre de Lua:** Escribir `buffer.writeu8(buf, 1, val)` en un buffer de $1\text{ byte}$ (`buffer.create(1)`) lanza inmediatamente `buffer access out of bounds`, porque el único índice válido es `0`.
- **Leer `f32`/`f64` de un cliente sin validar `NaN`:** Un cliente malicioso puede enviar los bytes IEEE 754 de `NaN` (`0x7FC00000`) dentro de un `buffer`. Después de llamar a `buffer.readf32` o `readf64` en el servidor, valida siempre `val == val and math.abs(val) < math.huge`.
- **Olvidar que `buffer` en `DataStore` requiere serialización si se guarda dentro de tablas mixtas o versiones antiguas:** Aunque los `RemoteEvents` transmiten el tipo `buffer` nativamente con sobrecarga mínima, para persistencia sigue el esquema tipado de `ProfileStore`.

## 5. Cómo comprobar que está bien
1. Ejecutar test unitario en Lune (`lune run`) verificando que `PackPlayerFlags(true, false, true, 9)` seguido de `UnpackPlayerFlags` recupera exactamente `(true, false, true, 9)`.
2. Probar que `BufferCursor.fromBuffer(buffer.create(2)):ReadShortString()` devuelve `nil` sin lanzar excepción cuando el prefijo de longitud excede los bytes restantes.

## 6. Fuentes
- Roblox Creator Docs — `buffer` Library: `https://create.roblox.com/docs/reference/engine/libraries/buffer` (Consultado: 2026-04-09).
- Roblox Creator Docs — `bit32` Library: `https://create.roblox.com/docs/reference/engine/libraries/bit32` (Consultado: 2026-04-09).

**Conexiones del Vault:** [[MOC_Luau_Lenguaje_y_Tipado]] | [[14_Redes_Avanzadas_Buffers_y_Compresion]] | [[30_01_Modo_Strict_y_Anotaciones]] | [[33_03_Saneamiento_de_Tipos_NaN_Inf_y_UTF8]] | [[33_06_Empaquetado_Binario_de_Paquetes_con_Buffer]]

---
**Ver también (enciclopedia):** [[Buffers and Binary Data|Buffers]], [[Client Prediction and Reconciliation for Combat|Networking]]
