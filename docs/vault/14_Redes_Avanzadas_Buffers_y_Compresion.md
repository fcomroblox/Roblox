---
tags: [redes-avanzadas-buffers-y-compresion]
ultima_verificacion: pendiente
---
# 14. Redes de Alto Rendimiento: Librería `buffer` de Luau, `UnreliableRemoteEvent` e Interpolación

Relacionado con: [[00_Mapa_del_Vault]] | [[01_API_Roblox_y_Servicios]] | [[03_Seguridad_Cliente_Servidor]] | [[06_Optimizacion_Movil_Streaming]]

---

## 1. Presupuesto de Ancho de Banda en Roblox
Para mantener una latencia baja en conexiones móviles 4G/5G y evitar el estrangulamiento (*throttling*) de paquetes de red:
- **Límite de Recepción por Cliente:** Mantener el tráfico entrante (`Incoming KB/s` en `Shift + F3` / Consola `F9`) por debajo de **$50\text{ KB/s}$** en juegos estándar y nunca superar $100\text{ KB/s}$.
- **El Coste de Enviar Diccionarios con Claves de Texto:** Enviar `{ playerId = 123456, position = Vector3.new(10, 5, 20), state = "Running" }` por un `RemoteEvent` serializa los nombres de las claves (`"playerId"`, `"position"`, `"state"`) en cada paquete. En sistemas de alta frecuencia (sincronizar 50 enemigos, proyectiles o dirección de apuntado), esto multiplica por $5\times$ el consumo de red.

---

## 2. Serialización Binaria con la Librería Nativa `buffer` de Luau
Luau incluye el tipo primitivo **`buffer`** para empaquetar números enteros, flotantes y booleanos directamente en bytes contiguos antes de enviarlos por `RemoteEvent` o `UnreliableRemoteEvent`:

| Función de la Librería `buffer` | Bytes Consumidos | Rango de Valores | Caso de Uso Ideal en Red |
| :--- | :--- | :--- | :--- |
| `buffer.writeu8(b, offset, val)` | $1\text{ byte}$ | $0\text{ a }255$ | IDs de estado, nivel de mejora (`toolTier`), porcentaje de salud ($0\text{-}100$), banderas booleanas. |
| `buffer.writeu16(b, offset, val)` | $2\text{ bytes}$ | $0\text{ a }65.535$ | Identificador local de entidad/enemigo en el servidor, ángulos cuantizados, cantidades de inventario. |
| `buffer.writei16(b, offset, val)` | $2\text{ bytes}$ | $-32.768\text{ a }32.767$ | Coordenadas de posición multiplicadas por $10$ (precisión de $0.1\text{ studs}$ en mapas de $\pm 3.200\text{ studs}$). |
| `buffer.writef32(b, offset, val)` | $4\text{ bytes}$ | Flotante IEEE 754 | Coordenadas exactas, marcas de tiempo de alta precisión. |

~~~luau
--!strict
-- Empaquetado binario de una actualización de entidad en solo 10 bytes (frente a >65 bytes en tabla)
local PacketSerializer = {}

export type EntitySnapshot = {
	entityId: number, -- u16 (2 bytes)
	posX: number,     -- i16 cuantizado x10 (2 bytes)
	posY: number,     -- i16 cuantizado x10 (2 bytes)
	posZ: number,     -- i16 cuantizado x10 (2 bytes)
	yawAngleDeg: number, -- u8 mapeado 0..255 (1 byte)
	healthPercent: number, -- u8 0..100 (1 byte)
}

local BYTES_PER_ENTITY = 10

function PacketSerializer.EncodeBatch(snapshots: { EntitySnapshot }): buffer
	local count = #snapshots
	local buf = buffer.create(count * BYTES_PER_ENTITY)

	for i = 1, count do
		local snap = snapshots[i]
		local offset = (i - 1) * BYTES_PER_ENTITY

		buffer.writeu16(buf, offset, math.clamp(snap.entityId, 0, 65535))
		buffer.writei16(buf, offset + 2, math.clamp(math.round(snap.posX * 10), -32768, 32767))
		buffer.writei16(buf, offset + 4, math.clamp(math.round(snap.posY * 10), -32768, 32767))
		buffer.writei16(buf, offset + 6, math.clamp(math.round(snap.posZ * 10), -32768, 32767))

		local normalizedYaw = math.floor(((snap.yawAngleDeg % 360) / 360) * 255)
		buffer.writeu8(buf, offset + 8, math.clamp(normalizedYaw, 0, 255))
		buffer.writeu8(buf, offset + 9, math.clamp(math.round(snap.healthPercent), 0, 100))
	end

	return buf
end

function PacketSerializer.DecodeBatch(buf: buffer): { EntitySnapshot }
	local totalBytes = buffer.len(buf)
	if totalBytes % BYTES_PER_ENTITY ~= 0 then
		return {}
	end

	local count = totalBytes // BYTES_PER_ENTITY
	local result = table.create(count)

	for i = 1, count do
		local offset = (i - 1) * BYTES_PER_ENTITY
		local entityId = buffer.readu16(buf, offset)
		local posX = buffer.readi16(buf, offset + 2) / 10
		local posY = buffer.readi16(buf, offset + 4) / 10
		local posZ = buffer.readi16(buf, offset + 6) / 10
		local yawByte = buffer.readu8(buf, offset + 8)
		local healthPercent = buffer.readu8(buf, offset + 9)

		result[i] = {
			entityId = entityId,
			posX = posX,
			posY = posY,
			posZ = posZ,
			yawAngleDeg = (yawByte / 255) * 360,
			healthPercent = healthPercent,
		}
	end

	return result
end

return table.freeze(PacketSerializer)
~~~

---

## 3. Reglas de `UnreliableRemoteEvent` e Interpolación en Cliente
1. **Límite Duro de Tamaño:** Todo paquete enviado por un `UnreliableRemoteEvent` debe pesar **menos de $900\text{ bytes}$** (si supera el MTU de red de Roblox, se descarta silenciosamente).
2. **Frecuencia de Envío de Snapshots (Tickrate de Red):** El servidor corre a $60\text{ Hz}$ en `Heartbeat`, pero **no debes enviar actualizaciones de posición en cada fotograma**. Agrupa y envía los snapshots a **$15\text{ Hz}$ o $20\text{ Hz}$** (cada $0.05\text{-}0.066\text{ s}$) y en el cliente interpola suavemente las posiciones visuales a $60\text{ FPS}$ dentro de `RunService.PreRender` o `PreSimulation` usando `vectorA:Lerp(vectorB, alpha)`.

---
**Ver también (enciclopedia):** [[Client Prediction and Reconciliation for Combat|Networking]], [[Buffers and Binary Data|Buffers]]
