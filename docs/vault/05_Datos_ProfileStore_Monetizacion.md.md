# 05. Persistencia con ProfileStore y Monetización Idempotente

## 1. Uso Obligatorio de ProfileStore (Sucesor de ProfileService)
En este proyecto no se utiliza `DataStoreService:GetAsync/SetAsync` en crudo ni el módulo antiguo `ProfileService`. Se emplea exclusivamente `ProfileStore` de loleris (`ServerScriptService.ServerPackages.ProfileStore`), que incorpora tipado Luau nativo, auto-guardado optimizado cada 300 segundos (10 veces menos llamadas a DataStore) y resolución acelerada de bloqueo de sesiones entre servidores mediante `MessagingService`.
- **Iniciar sesión:** `PlayerStore:StartSessionAsync(profileKey, { Cancel = function() return player.Parent ~= Players end })`.
- **Finalizar sesión:** `profile:EndSession()` al desconectarse el jugador, y escuchar `profile.OnSessionEnd:Connect(...)` para expulsar al jugador si otro servidor reclama la sesión.
- **Verificación de datos guardados:** `MetaTags` ha sido reemplazado por `profile.LastSavedData` y la señal `profile.OnAfterSave`.
- **Mensajería offline/cross-server:** `GlobalUpdates` ha sido reemplazado por `PlayerStore:MessageAsync(profileKey, payload)` y `profile:MessageHandler(fn)`.

## 2. Patrón Completo de DataService.luau + MarketplaceService.ProcessReceipt
Para evitar pérdida de compras con Robux cuando un servidor se cierra o falla la red, `ProcessReceipt` NUNCA devuelve `PurchaseGranted` hasta que el recibo consta confirmado dentro de `profile.LastSavedData`:

```luau
--!strict
local Players = game:GetService("Players")
local MarketplaceService = game:GetService("MarketplaceService")
local ServerScriptService = game:GetService("ServerScriptService")

local ProfileStore = require(ServerScriptService.ServerPackages.ProfileStore)

export type PlayerData = {
	coins: number,
	gems: number,
	processedReceipts: { [string]: boolean },
}

local DATA_TEMPLATE: PlayerData = {
	coins = 0,
	gems = 0,
	processedReceipts = {},
}

local PlayerStore = ProfileStore.New("LivePlayerData_v1", DATA_TEMPLATE)
local Profiles: { [Player]: typeof(PlayerStore:StartSessionAsync("", {} :: any)) } = {}

local DEV_PRODUCTS: { [number]: (data: PlayerData) -> () } = {
	[10000001] = function(data: PlayerData)
		data.gems += 250
	end,
}

local function loadPlayerProfile(player: Player)
	local profile = PlayerStore:StartSessionAsync(`User_{player.UserId}`, {
		Cancel = function()
			return player.Parent ~= Players
		end,
	})

	if profile ~= nil then
		profile:AddUserId(player.UserId) -- Obligatorio para cumplimiento GDPR en Roblox
		profile:Reconcile()

		profile.OnSessionEnd:Connect(function()
			Profiles[player] = nil
			player:Kick("Tu sesión se ha abierto en otro servidor. Vuelve a unirte.")
		end)

		if player.Parent == Players then
			Profiles[player] = profile
		else
			profile:EndSession()
		end
	else
		player:Kick("Error al bloquear la sesión de datos. Reintenta en unos segundos.")
	end
end

-- Implementación idempotente de ProcessReceipt garantizando guardado en DataStore
MarketplaceService.ProcessReceipt = function(receiptInfo: {
	PlayerId: number,
	PurchaseId: string,
	ProductId: number,
}): Enum.ProductPurchaseDecision
	local player = Players:GetPlayerByUserId(receiptInfo.PlayerId)
	if not player then
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end

	local profile = Profiles[player]
	while not profile and player.Parent == Players do
		task.wait(0.1)
		profile = Profiles[player]
	end

	if not profile or not profile:IsActive() then
		return Enum.ProductPurchaseDecision.NotProcessedYet
	end

	-- Si el recibo ya consta en el último guardado en la nube, confirmar compra
	if profile.LastSavedData.processedReceipts[receiptInfo.PurchaseId] then
		return Enum.ProductPurchaseDecision.PurchaseGranted
	end

	-- Si aún no se otorgó en memoria durante esta sesión, aplicar recompensa
	if not profile.Data.processedReceipts[receiptInfo.PurchaseId] then
		local grantProduct = DEV_PRODUCTS[receiptInfo.ProductId]
		if not grantProduct then
			warn(`[ProcessReceipt] ProductId no registrado: {receiptInfo.ProductId}`)
			return Enum.ProductPurchaseDecision.NotProcessedYet
		end

		grantProduct(profile.Data)
		profile.Data.processedReceipts[receiptInfo.PurchaseId] = true
	end

	-- Forzar guardado inmediato y esperar confirmación en LastSavedData
	profile:Save()
	local savedConfirmed = false
	local connection = profile.OnAfterSave:Connect(function(lastSavedData: PlayerData)
		if lastSavedData.processedReceipts[receiptInfo.PurchaseId] then
			savedConfirmed = true
		end
	end)

	local timeout = os.clock() + 10
	while not savedConfirmed and profile:IsActive() and os.clock() < timeout do
		task.wait(0.1)
		if profile.LastSavedData.processedReceipts[receiptInfo.PurchaseId] then
			savedConfirmed = true
		end
	end
	connection:Disconnect()

	if savedConfirmed then
		return Enum.ProductPurchaseDecision.PurchaseGranted
	end

	return Enum.ProductPurchaseDecision.NotProcessedYet
end

Players.PlayerAdded:Connect(loadPlayerProfile)
for _, player in Players:GetPlayers() do
	task.spawn(loadPlayerProfile, player)
end

Players.PlayerRemoving:Connect(function(player: Player)
	local profile = Profiles[player]
	if profile ~= nil then
		profile:EndSession()
	end
end)
```

## 3. Gestión de Game Passes (`UserOwnsGamePassAsync`)
1. Comprueba `MarketplaceService:UserOwnsGamePassAsync(player.UserId, gamePassId)` envuelto en `pcall` al cargar el perfil del jugador y cachea el resultado booleano en una tabla de sesión en servidor.
2. Escucha `MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, passId, wasPurchased) ...)` en el servidor para activar los beneficios inmediatamente en la misma partida cuando `wasPurchased == true`.