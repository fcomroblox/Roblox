---
tags: [matchmaking-memorystore-y-messagingservice]
ultima_verificacion: pendiente
---
# 15. Arquitectura Multi-Servidor: Matchmaking (`MemoryStoreService`), `TeleportService` y `OrderedDataStore`

Relacionado con: [[00_Mapa_del_Vault]] | [[01_API_Roblox_y_Servicios]] | [[05_Datos_ProfileStore_Monetizacion]] | [[18_Open_Cloud_v2_CI_CD_y_Automatizacion]]

---

## 1. Teletransporte Seguro a Servidores Reservados (`TeleportAsync`)
En juegos cooperativos por partidas o mazmorras donde un grupo de $1\text{ a }4$ jugadores viaja desde el Lobby a una instancia privada:
- **API Única Permitida:** Usa siempre `TeleportService:TeleportAsync(placeId, playersArray, teleportOptions)`. Las funciones antiguas `Teleport()`, `TeleportToPrivateServer()` y `TeleportPartyAsync()` están **deprecadas**.
- **Reserva de Servidor y Datos de Entrada:** Configura `teleportOptions.ShouldReserveServer = true` y envuelve siempre la llamada en `pcall` con reintentos exponenciales en caso de fallo transitorio de los servidores de Roblox.

~~~luau
--!strict
local TeleportService = game:GetService("TeleportService")

local PartyTeleporter = {}

function PartyTeleporter.SendPartyToReservedMatch(
	missionPlaceId: number,
	partyPlayers: { Player },
	difficultyTier: number
): (boolean, string?)
	local options = Instance.new("TeleportOptions")
	options.ShouldReserveServer = true
	options:SetTeleportData({
		difficultyTier = difficultyTier,
		launchedAtUnix = os.time(),
	})

	local maxAttempts = 3
	for attempt = 1, maxAttempts do
		local success, result = pcall(function()
			return TeleportService:TeleportAsync(missionPlaceId, partyPlayers, options)
		end)

		if success then
			return true, nil
		end

		warn(`[PartyTeleporter] Intento {attempt} fallido: {result}`)
		task.wait(1.5 * attempt)
	end

	return false, "No se pudo reservar el servidor de misión. Inténtalo de nuevo."
end

return table.freeze(PartyTeleporter)
~~~
- **Nota de Seguridad sobre `GetLocalPlayerTeleportData`:** Los datos pasados en `SetTeleportData` viajan a través del cliente y **pueden ser falsificados por un exploiter**. Úsalos solo para ajustes visuales de pantalla de carga o metadatos no críticos; para datos críticos de partida entre servidores, guárdalos en `MemoryStoreService` indexados por el `ReservedServerAccessCode` o en el perfil de `ProfileStore`.

---

## 2. Colas de Emparejamiento Globales con `MemoryStoreService`
`MemoryStoreService` ofrece estructuras en memoria RAM compartidas entre todos los servidores activos con expiración automática (TTL):
- **`GetSortedMap("GlobalMatchQueue")`:** Ideal para crear salas públicas cross-server donde cada sala es una clave con TTL de $120\text{ segundos}$ y su valor contiene el número de jugadores actuales.
- **Límites de Cuota de MemoryStore:**
  - Unidades de petición por minuto: $1.000 + 120 \times \text{JugadoresConectados}$.
  - Tamaño máximo de valor por clave: $32\text{ KB}$.
  - Usa siempre `sortedMap:UpdateAsync(key, transformFunction, expirationSeconds)` para modificar contadores de sala de forma atómica sin condiciones de carrera entre dos servidores distintos.

---

## 3. Eventos Globales en Vivo con `MessagingService`
Para anunciar un evento global en todos los servidores (por ejemplo, cuando un jugador compra un Developer Product de *"Lluvia de Suerte Global"* o se lanza una actualización en caliente):
- **Cuota Oficial:**
  - Mensajes publicados por minuto en el universo: $150 + 60 \times \text{NúmeroDeServidores}$.
  - Tamaño máximo del mensaje: $1\text{ KB}$.
- Envuelve siempre `MessagingService:PublishAsync(topic, data)` y `MessagingService:SubscribeAsync(topic, callback)` dentro de `pcall`.

---

## 4. Clasificaciones Globales (`OrderedDataStore`) sin Agotar Cuotas
`DataStoreService:GetOrderedDataStore(name)` permite mostrar los 50 mejores jugadores del mundo en un panel del Lobby:
- **Regla Anti-Throttling:** Está **prohibido** actualizar ni leer el `OrderedDataStore` cada vez que un jugador gana una moneda o cuando un cliente hace clic en la tabla.
- **Patrón de Producción:** Un único bucle en el servidor actualiza la puntuación de los jugadores conectados y descarga las primeras 50 entradas (`GetSortedAsync(false, 50)`) **una sola vez cada $120\text{ a }180\text{ segundos}$**, replicando la tabla cacheada en `ReplicatedStorage` para que todos los clientes la lean gratis.

---
**Ver también (enciclopedia):** [[Multiplayer Services Start Here|Multiplayer Services]]
