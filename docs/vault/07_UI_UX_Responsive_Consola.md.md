# 07. Arquitectura de UI/UX Responsive (Móvil, PC y Consola)

## 1. Reglas de Escalado y Restricciones (`Scale` vs `Offset`)
Una interfaz profesional en Roblox debe verse nítida y proporcionada tanto en un teléfono de $6\text{ pulgadas}$ ($720\text{p}$) como en un monitor 4K o televisor de consola:
1. **Posicionamiento y Tamaño en `Scale`:** Define los contenedores principales usando `UDim2.fromScale(x, y)` y fija `AnchorPoint = Vector2.new(0.5, 0.5)` para centrar paneles de forma predecible.
2. **`UIAspectRatioConstraint` Obligatorio:** Añade siempre un `UIAspectRatioConstraint` al `Frame` raíz de cada ventana modal y a las cuadrículas de iconos para que nunca se estiren ni deformen en pantallas ultrapanorámicas.
3. **`UISizeConstraint` para Ergonomía Táctil:** Todo botón interactivo (`TextButton` o `ImageButton`) DEBE incluir un `UISizeConstraint` con `MinSize = Vector2.new(48, 48)` píxeles. Es el tamaño mínimo físico para que un pulgar en pantalla táctil no falle la pulsación.
4. **Tipografía Adaptable:** Al usar `TextScaled = true` en `TextLabel` o `TextButton`, añade siempre un hijo `UITextSizeConstraint` (`MinTextSize = 12`, `MaxTextSize = 36`) para evitar textos ilegibles o gigantescos, y envuelve la fuente con un `UIStroke` de alto contraste.
5. **Áreas Seguras (`ScreenInsets`):** En el objeto `ScreenGui`, mantén `ScreenInsets = Enum.ScreenInsets.CoreUISafeInsets` para los botones del HUD de modo que nunca colisionen con el botón del menú de Roblox, el chat o el joystick virtual táctil (`TouchGui`).

## 2. Soporte Unificado Táctil, Ratón y Mando de Consola (`Gamepad`)
1. **Usa `.Activated`, NUNCA `.MouseButton1Click`:** El evento `GuiButton.Activated` se dispara de forma nativa y unificada con clic izquierdo de ratón, toque en pantalla móvil y botón `ButtonA` / `Cross` del mando de consola.
2. **Navegación por Mando (`GuiService.SelectedObject`):**
   - Asegúrate de que todos los botones de un menú tengan `Selectable = true`.
   - Cuando se abra una ventana modal y `UserInputService.GamepadEnabled == true`, asigna `GuiService.SelectedObject = primerBotonDelMenu`. Al cerrarse el menú, limpia `GuiService.SelectedObject = nil`.

## 3. Patrón de Controlador de Botón con Feedback Sensorial (`TweenService`)

```luau
--!strict
local GuiService = game:GetService("GuiService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local Trove = require(ReplicatedStorage.Packages.Trove)

local UIButtonHelper = {}

local PRESS_TWEEN_INFO = TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
local RELEASE_TWEEN_INFO = TweenInfo.new(0.18, Enum.EasingStyle.Back, Enum.EasingDirection.Out)

function UIButtonHelper.BindResponsiveButton(button: GuiButton, onActivated: () -> ()): typeof(Trove.new())
	local trove = Trove.new()
	trove:AttachToInstance(button)

	button.Selectable = true
	button.AutoButtonColor = false

	-- Asegurar tamaño mínimo táctil de 48x48 px
	if not button:FindFirstChildOfClass("UISizeConstraint") then
		local sizeConstraint = Instance.new("UISizeConstraint")
		sizeConstraint.MinSize = Vector2.new(48, 48)
		sizeConstraint.Parent = button
	end

	local uiScale = button:FindFirstChildOfClass("UIScale")
	if not uiScale then
		uiScale = Instance.new("UIScale")
		uiScale.Scale = 1
		uiScale.Parent = button
	end

	trove:Connect(button.InputBegan, function(input: InputObject)
		if
			input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch
			or input.KeyCode == Enum.KeyCode.ButtonA
		then
			TweenService:Create(uiScale, PRESS_TWEEN_INFO, { Scale = 0.92 }):Play()
		end
	end)

	trove:Connect(button.InputEnded, function(input: InputObject)
		if
			input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch
			or input.KeyCode == Enum.KeyCode.ButtonA
		then
			TweenService:Create(uiScale, RELEASE_TWEEN_INFO, { Scale = 1.0 }):Play()
		end
	end)

	trove:Connect(button.Activated, function()
		onActivated()
	end)

	return trove
end

function UIButtonHelper.FocusModalForGamepad(modalFrame: GuiObject, defaultButton: GuiButton)
	modalFrame.Visible = true
	if UserInputService.GamepadEnabled then
		GuiService.SelectedObject = defaultButton
	end
end

return table.freeze(UIButtonHelper)
```