---
tags: [efectos-visuales-vfx-iluminacion-y-jugabilidad-juice]
ultima_verificacion: pendiente
---
# 20. Efectos Visuales (VFX), Límites de `Highlight` y Pulido Sensorial (*Game Feel & Juice*)

Relacionado con: [[00_Mapa_del_Vault]] | [[art_bible]] | [[06_Optimizacion_Movil_Streaming]] | [[13_Audio_Modular_AudioPlayer_y_Wires]]

---

## 1. El Principio de Respuesta en Cero Milisegundos (*Client-Immediate Juice*)
En un juego excelente de Roblox, cuando el jugador hace clic o toca la pantalla para golpear un nodo de cristal, disparar o pulsar un botón:
- **El efecto visual y sonoro ocurre en el cliente en el fotograma $0$** (sin esperar el viaje de ida y vuelta al servidor de $100\text{-}200\text{ ms}$).
- Simultáneamente, el cliente envía el `RemoteEvent` al servidor. Cuando el servidor valida y responde, actualiza el contador oficial de monedas/recursos.

---

## 2. Límites Técnicos del Motor en VFX (`Highlight`, `ParticleEmitter`, `Beam`)
1. **Límite Duro de `Highlight` (Máximo 31 Instancias Activas):**
   - **Hecho Verificado del Motor de Roblox:** Roblox solo puede renderizar **un máximo de $31\text{ instancias }Highlight$ activas simultáneamente** en toda la escena. Si creas un `Highlight` número $32$, simplemente no se dibujará en pantalla.
   - **Regla de Oro:** Jamás dejes un objeto `Highlight` desactivado o permanente dentro de cientos de nodos o enemigos. Usa un **Pool de Highlights en el cliente** de máximo $15\text{-}20$ instancias que se asignan dinámicamente (`highlight.Adornee = targetModel`) solo cuando el jugador pasa el cursor por encima o golpea al objetivo durante $0.15\text{ segundos}$ (*Hit Flash*), y luego devuélvelos a `Adornee = nil`.
2. **Ráfagas de Partículas con `:Emit(count)` en lugar de `.Enabled = true`:**
   - Para impactos, explosiones o recolección de monedas, mantén `particleEmitter.Enabled = false` y dispara una ráfaga exacta por código con `particleEmitter:Emit(12)`. Es mucho más limpio y determinista que encender `Enabled = true`, hacer `task.wait(0.2)` y apagarlo.
3. **Presupuesto de Luces Dinámicas (`PointLight`, `SpotLight`, `SurfaceLight`):**
   - En dispositivos móviles, las luces con `Shadows = true` son muy costosas. Activa `Shadows = true` únicamente en la luz principal de la linterna del jugador o faros de su vehículo; en cristales decorativos o antorchas del escenario, fija siempre `Shadows = false` y `Range <= 24`.

---

## 3. Módulo de Impacto Visual y Sacudida de Cámara (`VFXJuiceController.luau`)

~~~luau
--!strict
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")

local VFXJuiceController = {}

local FLASH_TWEEN = TweenInfo.new(0.16, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local SCALE_PUNCH_TWEEN = TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out)

-- Pool seguro de 12 Highlights para nunca superar el límite de 31 del motor de Roblox
local highlightPool: { Highlight } = {}
for i = 1, 12 do
	local hl = Instance.new("Highlight")
	hl.Name = `PooledHitHighlight_{i}`
	hl.DepthMode = Enum.HighlightDepthMode.Occluded
	hl.FillColor = Color3.fromHex("#FFFFFF")
	hl.OutlineColor = Color3.fromHex("#00F5D4")
	hl.FillTransparency = 1
	hl.OutlineTransparency = 1
	hl.Parent = Workspace
	table.insert(highlightPool, hl)
end

local poolIndex = 1
local shakeIntensity = 0

-- Efecto de destello blanco/cian y rebote elástico al golpear un modelo (ej. ResourceNode)
function VFXJuiceController.PlayHitFlashAndPunch(targetModel: Model)
	local hl = highlightPool[poolIndex]
	poolIndex = (poolIndex % #highlightPool) + 1

	hl.Adornee = targetModel
	hl.FillTransparency = 0.25
	hl.OutlineTransparency = 0.0

	local fadeTween = TweenService:Create(hl, FLASH_TWEEN, {
		FillTransparency = 1,
		OutlineTransparency = 1,
	})
	fadeTween:Play()

	-- Rebote de escala sutil sobre el modelo
	local originalScale = targetModel:GetScale()
	targetModel:ScaleTo(originalScale * 0.90)

	local scaleValue = Instance.new("NumberValue")
	scaleValue.Value = originalScale * 0.90
	local connection = scaleValue.Changed:Connect(function(newScale: number)
		if targetModel.Parent then
			targetModel:ScaleTo(newScale)
		end
	end)

	local punchTween = TweenService:Create(scaleValue, SCALE_PUNCH_TWEEN, { Value = originalScale })
	punchTween.Completed:Once(function()
		connection:Disconnect()
		scaleValue:Destroy()
	end)
	punchTween:Play()
end

-- Sacudida sutil de cámara ejecutada en RunService.PreRender
function VFXJuiceController.AddCameraShake(impulse: number)
	shakeIntensity = math.clamp(shakeIntensity + impulse, 0, 1.5)
end

RunService.PreRender:Connect(function(dt: number)
	if shakeIntensity > 0.01 then
		local camera = Workspace.CurrentCamera
		if camera then
			local offset = Vector3.new(
				(math.random() - 0.5) * 2 * shakeIntensity,
				(math.random() - 0.5) * 2 * shakeIntensity,
				0
			)
			camera.CFrame *= CFrame.new(offset)
		end
		shakeIntensity = math.max(0, shakeIntensity - dt * 6.0)
	end
end)

return table.freeze(VFXJuiceController)
~~~

---
**Ver también (enciclopedia):** [[Beams and Trails|VFX]], [[Game Feel Start Here|Game Feel]]
