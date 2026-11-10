# 08. Normas de Moderación, PolicyService y Políticas del Marketplace

## 1. Prevención de Rechazos y Sanciones al Subir Assets Generados por IA
El sistema de moderación automatizada de Roblox escanea cada textura 2D, mapa UV de modelo 3D y pista de audio antes de asignarle un `rbxassetid://`. Como el titular de la cuenta o Grupo responde legalmente por todo lo que suba a través del servidor MCP o Open Cloud, se aplican estas cuatro reglas innegociables:
1. **Cero Texto o Garabatos en Texturas 3D/2D (Filtro OCR):** Los generadores de imágenes y texturas 3D suelen dibujar letras falsas o símbolos parecidos a palabras en carteles, pantallas o ropa. El escáner OCR de Roblox interpreta esos garabatos como intentos de ocultar enlaces fuera de la plataforma o información personal, rechazando el activo y aplicando advertencias a la cuenta. **Solución:** Añade siempre `no text, no letters, no numbers, no logos, clean surface` en los prompts de generación visual. Si necesitas texto en un cartel dentro del juego, usa un `SurfaceGui` + `TextLabel` nativo de Roblox Studio.
2. **Prohibición de Marcas Reales y Derechos de Autor:** Nunca incluyas nombres ni logotipos de marcas comerciales reales (coches, refrescos, ropa, armas con denominación comercial registrada).
3. **Revisión de Mapas UV Desplegados:** Verifica que ningún mapa de textura Albedo contenga zonas de color carne que el clasificador visual automático pueda confundir fuera de contexto con desnudez.
4. **Audio Instrumental y Cuotas Verificadas:** Las pistas musicales generadas por IA (Suno / Udio) no deben superar los $7\text{ minutos}$ y deben ser **100% instrumentales** (las voces sintéticas disparan con frecuencia filtros de moderación vocal o reclamaciones). Recuerda que una cuenta con identidad verificada (ID Verified) dispone de $2.000\text{ subidas de audio gratuitas cada }30\text{ días}$.

## 2. Cumplimiento Legal por Jugador con `PolicyService`
Las leyes internacionales sobre cajas de botín aleatorias de pago (*lootboxes* / *gacha*), comercio de ítems y enlaces a redes sociales varían según el país y la edad del jugador. Está **prohibido** mostrar una ruleta o cofre aleatorio pagado con Robux (o con monedas compradas con Robux) sin consultar antes `PolicyService:GetPolicyInfoForPlayerAsync(player)`:

```luau
--!strict
local PolicyService = game:GetService("PolicyService")

export type PlayerCompliancePolicy = {
	ArePaidRandomItemsRestricted: boolean,
	IsPaidItemTradingAllowed: boolean,
	AllowedExternalLinkReferences: { string },
}

local PolicyHelper = {}

function PolicyHelper.CanPlayerBuyRandomCrate(player: Player): boolean
	local success, result = pcall(function()
		return PolicyService:GetPolicyInfoForPlayerAsync(player)
	end)

	if not success or not result then
		-- Por seguridad legal, si la llamada falla se asume que está restringido
		warn(`[PolicyService] No se pudo verificar la política para {player.Name}. Bloqueando compra aleatoria.`)
		return false
	end

	return result.ArePaidRandomItemsRestricted == false
end

return table.freeze(PolicyHelper)
```
- **Divulgación de Probabilidades Obligatoria:** Siempre que exista un cofre o recompensa aleatoria permitida, la interfaz de usuario (`ScreenGui`) debe mostrar explícitamente en pantalla el porcentaje exacto de probabilidad de cada categoría (ej. *Común: 70%, Raro: 25%, Legendario: 5%*).

## 3. Aislamiento de Entornos para el Servidor MCP (Staging vs. Producción)
- **Regla Operativa:** El servidor MCP de Roblox Studio y el comando `open_cloud_invoke` deben usarse **únicamente en el universo privado de `[DEV] Staging`**.
- En el universo público de **Producción**, mantén desactivados en el panel `MCP Permissions` los permisos de escritura en DataStores en vivo, monetización y sanciones de jugadores para impedir que una instrucción mal interpretada altere la economía real.