---
tags: [normas-moderacion-politicas]
ultima_verificacion: pendiente
---
# 08. Normas de Moderación, PolicyService y Políticas de Roblox

## 1. Prevención de Rechazos y Baneos Automáticos por OCR en Texturas 3D/2D
El escáner automático de moderación de imágenes y texturas de Roblox aplica reconocimiento óptico de caracteres (OCR) sobre cada mapa UV y activo 2D subido:
- **Cero Texto en Modelos y Texturas IA:** Los generadores de imágenes y texturas 3D suelen inventar pseudotexto o símbolos parecidos a letras en superficies, lo que dispara falsos positivos en el filtro OCR de Roblox por *"información personal o enlaces externos"*.
- **Regla de Prompts 3D/2D:** Incluye siempre en los prompts negativos y de especificación: `no text, no letters, no numbers, no logos, no watermarks, no gibberish typography`.
- **Revisión Pre-Subida:** Revisa visualmente cada textura antes de moverla a la carpeta aprobada (`Approved Upload Folder`) del servidor MCP, y realiza las subidas de prueba utilizando una cuenta secundaria verificada con permisos de desarrollador en el Grupo en lugar de arriesgar la cuenta propietaria principal.
- **Prohibición de Marcas del Mundo Real:** Nunca utilices nombres, logotipos ni siluetas registradas de marcas reales (coches, ropa, refrescos, armas con marcas comerciales).

---

## 2. Cumplimiento Legal con `PolicyService` (Cajas de Botín y Enlaces)
En jurisdicciones con leyes estrictas para menores (como Reino Unido, Países Bajos, Bélgica o Australia), Roblox restringe los artículos aleatorios de pago (Lootboxes / Gacha con Robux) y las referencias a redes sociales externas. Es obligatorio consultar `PolicyService:GetPolicyInfoForPlayerAsync(player)` en el servidor antes de mostrar o permitir compras aleatorias:

```luau
--!strict
local PolicyService = game:GetService("PolicyService")
local Players = game:GetService("Players")

export type PlayerCompliancePolicy = {
	canBuyPaidRandomItems: boolean,
	allowedExternalLinks: { string },
}

local policyCache: { [Player]: PlayerCompliancePolicy } = {}

local function fetchPlayerPolicy(player: Player): PlayerCompliancePolicy
	if policyCache[player] then
		return policyCache[player]
	end

	local success, result = pcall(function()
		return PolicyService:GetPolicyInfoForPlayerAsync(player)
	end)

	local policy: PlayerCompliancePolicy = {
		-- Por seguridad, si falla la llamada asumimos restricción activa (false)
		canBuyPaidRandomItems = if success and result then not result.ArePaidRandomItemsRestricted else false,
		allowedExternalLinks = if success and result then result.AllowedExternalLinkReferences else {},
	}

	if player.Parent == Players then
		policyCache[player] = policy
	end

	return policy
end

Players.PlayerRemoving:Connect(function(player: Player)
	policyCache[player] = nil
end)

return table.freeze({
	FetchPlayerPolicy = fetchPlayerPolicy,
})
```

---

## 3. Separación Estricta de Entornos: Staging vs. Producción para MCP
El servidor MCP de Roblox Studio se conecta a los DataStores reales de la experiencia abierta:
1. Mantén siempre **dos universos separados** dentro de tu Grupo de Roblox: **`[DEV] Proyecto - Staging`** (privado) y **`Proyecto - Producción`**.
2. El desarrollo diario con Claude Code y MCP se realiza exclusivamente sobre **Staging**.
3. En el lugar de **Producción**, mantén desactivados los permisos MCP de escritura en DataStores, monetización y modificación del universo (`universe:write`) para evitar alteraciones accidentales de datos de jugadores en vivo.

---
**Conexiones del Vault:** [[00_Mapa_del_Vault]] | [[03_Seguridad_Cliente_Servidor]] | [[05_Datos_ProfileStore_Monetizacion]] | [[16_Pipeline_3D_Blender_PBR_y_MCP_Import]] | [[18_Open_Cloud_v2_CI_CD_y_Automatizacion]]

---
**Ver también (enciclopedia):** [[Community and Moderation Start Here|Moderation]], [[Age Checks and Safety Features|Safety Features]]
