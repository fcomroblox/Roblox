---
tags: [animaciones-r15-ikcontrol-y-motor6d]
ultima_verificacion: pendiente
---
# 12. Sistema de Animaciones R15, Cinemática Inversa (`IKControl`) y `Motor6D`

Relacionado con: [[00_Mapa_del_Vault]] | [[01_API_Roblox_y_Servicios]] | [[06_Optimizacion_Movil_Streaming]] | [[09_Errores_Comunes_IA_Roblox]]

---

## 1. Arquitectura de Reproducción (`Animator` y `AnimationPriority`)
- **Regla de Carga:** Está prohibido usar `Humanoid:LoadAnimation()`. Toda instancia `Animation` debe cargarse sobre el objeto `Animator` hijo del `Humanoid` (en avatares de jugadores) o hijo de un `AnimationController` (en NPCs optimizados, vehículos y objetos animados).
- **Dónde Cargar y Reproducir:**
  - Si el `Animator` fue creado en el servidor y replicado al cliente, el cliente puede llamar a `animator:LoadAnimation(anim)` desde un `LocalScript` y el motor de Roblox replicará automáticamente la animación al resto de jugadores sin consumir eventos de red manuales.
  - Pre-carga siempre las animaciones (`ContentProvider:PreloadAsync`) y reutiliza las instancias `AnimationTrack` guardándolas en una tabla de caché por personaje; nunca llames a `LoadAnimation()` repetidamente cada vez que el jugador ataca o salta (existe un límite duro de $256\text{ pistas}$ cargadas sobre un mismo `Animator`).
- **Jerarquía de Prioridades (`Enum.AnimationPriority`):**
  De menor a mayor prioridad de sobrescritura: `Core` $\rightarrow$ `Idle` $\rightarrow$ `Movement` $\rightarrow$ `Action` $\rightarrow$ `Action2` $\rightarrow$ `Action3` $\rightarrow$ `Action4`. Asigna siempre la prioridad por código (`track.Priority = Enum.AnimationPriority.Action2`) antes de llamar a `track:Play(fadeTime)`.

---

## 2. Eventos de Sincronización en Animaciones (`GetMarkerReachedSignal`)
Nunca uses la señal obsoleta `KeyframeReached` ni `task.wait(0.4)` fijos para sincronizar el sonido de un paso o el impacto de una espada (si cambias la velocidad de ataque con `track:AdjustSpeed(1.5)`, un `task.wait` fijo quedará desincronizado).
- Usa siempre **Animation Events (Markers)** nombrados en el Animation Editor de Studio y escúchalos con `track:GetMarkerReachedSignal("HitboxActive"):Connect(...)`.

---

## 3. Cinemática Inversa Nativa (`IKControl`)
Para lograr que las manos de un avatar R15 se agarren exactamente al volante de un vehículo, sostengan una herramienta pesada de distintos tamaños, apunten una linterna o giren la cabeza hacia un punto de interés sin crear docenas de animaciones estáticas, utiliza la instancia nativa **`IKControl`** dentro del `Humanoid` o `AnimationController`:

| Propiedad de `IKControl` | Configuración para Agarre de Manos | Configuración para Mirada de Cabeza (`LookAt`) |
| :--- | :--- | :--- |
| `Type` | `Enum.IKControlType.Transform` (posición y rotación) o `Position` | `Enum.IKControlType.LookAt` |
| `ChainRoot` | `character:FindFirstChild("RightUpperArm")` | `character:FindFirstChild("UpperTorso")` |
| `EndEffector` | `character:FindFirstChild("RightHand")` | `character:FindFirstChild("Head")` |
| `Target` | `Attachment` situado en el mango de la herramienta o volante | `Attachment` o `BasePart` del objeto a mirar |
| `SmoothTime` | $0.05\text{ s}$ (respuesta rápida de agarre) | $0.18\text{ s}$ (giro de cuello orgánico) |
| `Weight` | $1.0$ (activado) / $0.0$ (desactivado con `TweenService`) | $0.75$ |

~~~luau
--!strict
local Trove = require(game:GetService("ReplicatedStorage").Packages.Trove)

local CharacterIKHelper = {}

function CharacterIKHelper.AttachHandToGrip(
	character: Model,
	gripAttachment: Attachment,
	isRightHand: boolean
): typeof(Trove.new())
	local trove = Trove.new()
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local rootName = if isRightHand then "RightUpperArm" else "LeftUpperArm"
	local handName = if isRightHand then "RightHand" else "LeftHand"

	local chainRoot = character:FindFirstChild(rootName) :: BasePart?
	local endEffector = character:FindFirstChild(handName) :: BasePart?

	if not humanoid or not chainRoot or not endEffector then
		return trove
	end

	local ikControl = trove:Construct(Instance, "IKControl") :: IKControl
	ikControl.Name = `{handName}_GripIK`
	ikControl.Type = Enum.IKControlType.Transform
	ikControl.ChainRoot = chainRoot
	ikControl.EndEffector = endEffector
	ikControl.Target = gripAttachment
	ikControl.SmoothTime = 0.05
	ikControl.Weight = 1.0
	ikControl.Parent = humanoid

	return trove
end

return table.freeze(CharacterIKHelper)
~~~

---

## 4. Animación Procedural de Alto Rendimiento con `Motor6D.Transform`
Cuando necesites inclinar el torso del jugador al correr en curva (*character tilting*), girar hélices/ruedas de decenas de vehículos o animar enjambres de NPCs en el cliente sin sobrecargar el solver de física:
1. Conecta el cálculo **exclusivamente** al evento `RunService.PreSimulation` (que se ejecuta justo después de `PreAnimation` y antes de la simulación física).
2. Modifica la propiedad `motor6D.Transform = CFrame.new(...) * CFrame.Angles(...)` (a diferencia de `C0` y `C1`, modificar `Transform` no invalida la geometría caché del ensamblaje rígido y es extremadamente rápido en CPU).

---
**Ver también (enciclopedia):** [[AnimationTrack Reference|AnimationTrack]], [[Animator - Engine Reference Card|Animation Cards]]
