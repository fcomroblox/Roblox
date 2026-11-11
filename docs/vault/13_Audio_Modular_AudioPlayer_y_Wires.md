---
tags: [audio-modular-audioplayer-y-wires]
ultima_verificacion: pendiente
---
# 13. Sistema de Audio Modular (`AudioPlayer`, `Wire`, `AudioEmitter`) y Diseño Sonoro

Relacionado con: [[00_Mapa_del_Vault]] | [[01_API_Roblox_y_Servicios]] | [[08_Normas_Moderacion_Politicas]] | [[20_Efectos_Visuales_VFX_Iluminacion_y_Jugabilidad_Juice]]

---

## 1. Cuotas y Especificaciones de Archivos de Audio en Roblox
- **Cuota de Subida Verificada:** Una cuenta con identidad verificada (`ID Verified` + 2FA) dispone de **$2.000\text{ subidas de audio gratuitas cada }30\text{ días}$** (frente a solo $100$ en cuentas sin verificar).
- **Límites de Archivo:** Duración máxima de **$7\text{ minutos}$** ($420\text{ s}$) y tamaño máximo de $20\text{ MB}$ por archivo en formato `.ogg` (recomendado por menor latencia de decodificación y bucles sin costuras) o `.mp3`.
- **Seguridad de Moderación:** Toda música de fondo generada con IA (Suno / Udio) debe ser **100% instrumental** y normalizada en Audacity/Reaper a $-14\text{ LUFS}$ (música) y $-6\text{ dB}$ de pico (SFX).

---

## 2. Arquitectura del Nuevo Sistema de Audio Modular de Roblox
Roblox introdujo un grafo de audio basado en nodos conectados mediante instancias **`Wire`**, permitiendo enrutar múltiples reproductores hacia filtros DSP en tiempo real sin duplicar activos:

```text
[AudioPlayer] --(Wire)--> [AudioPitchShifter / AudioFilter / AudioReverb] --(Wire)--> [AudioEmitter (3D) o AudioDeviceOutput (2D)]
```

| Clase del Motor de Audio | Función en el Grafo | Propiedades Clave |
| :--- | :--- | :--- |
| `AudioPlayer` | Fuente que carga y reproduce el `AssetId` (`rbxassetid://...`). | `AssetId`, `Looping`, `Volume`, `PlaybackSpeed`, `TimePosition`, método `:Play()` y `:Stop()`. |
| `Wire` | Cable lógico que transporta el flujo de audio de un nodo a otro. | `SourceInstance` (nodo emisor), `TargetInstance` (nodo receptor). |
| `AudioEmitter` | Posiciona el sonido espacialmente en el mundo 3D cuando es hijo de un `BasePart` o `Attachment`. | `AudioInteractionGroup`, curva de atenuación por distancia (`SetDistanceAttenuation`). |
| `AudioListener` | Oído receptor (normalmente emparentado a `workspace.CurrentCamera`). | Captura los `AudioEmitter` del mismo grupo y los envía a `AudioDeviceOutput`. |
| `AudioFader` / `AudioCompressor` | Control de ganancia por canal (Música, SFX, UI, Ambiente) y limitación de picos cuando suenan muchas explosiones a la vez. | `Volume` ($0\text{ a }1$), `Threshold`, `Ratio`, `Attack`, `Release`. |

---

## 3. Variación de Tono (`Pitch Variation`) y Agrupación (`Sound Pooling`)
Repetir un mismo efecto de sonido (como golpear un nodo de cristal o recoger monedas) con exactamente la misma frecuencia produce fatiga auditiva inmediata en sesiones largas:
1. **Regla del Micro-Bucle:** Todo efecto de sonido repetitivo debe variar su velocidad/tono aleatoriamente entre **$0.94\text{ y }1.06$** en cada reproducción (o subir progresivamente en escala $+0.04$ por cada combo consecutivo).
2. **Pool de Sonidos 2D/3D en Cliente:** Nunca crees y destruyas instancias `Sound` o `AudioPlayer` masivamente en cada fotograma. Usa un gestor con pool prealojado:

~~~luau
--!strict
local SoundService = game:GetService("SoundService")
local Workspace = game:GetService("Workspace")

local AudioManager = {}

local sfxGroup = Instance.new("SoundGroup")
sfxGroup.Name = "MasterSFXGroup"
sfxGroup.Volume = 1.0
sfxGroup.Parent = SoundService

local soundPool2D: { [string]: { Sound } } = {}
local rng = Random.new()

function AudioManager.PlaySFX2D(assetId: string, baseVolume: number?, minPitch: number?, maxPitch: number?)
	local pool = soundPool2D[assetId]
	if not pool then
		pool = {}
		soundPool2D[assetId] = pool
	end

	local chosenSound: Sound? = nil
	for _, sound in pool do
		if not sound.IsPlaying then
			chosenSound = sound
			break
		end
	end

	if not chosenSound then
		local newSound = Instance.new("Sound")
		newSound.SoundId = assetId
		newSound.SoundGroup = sfxGroup
		newSound.Parent = sfxGroup
		table.insert(pool, newSound)
		chosenSound = newSound
	end

	if chosenSound then
		chosenSound.Volume = baseVolume or 0.7
		local low = minPitch or 0.94
		local high = maxPitch or 1.06
		chosenSound.PlaybackSpeed = rng:NextNumber(low, high)
		chosenSound:Play()
	end
end

function AudioManager.PlaySFX3DAtPosition(assetId: string, worldPosition: Vector3, maxDistance: number?)
	local attachment = Instance.new("Attachment")
	attachment.WorldPosition = worldPosition
	attachment.Parent = Workspace.Terrain

	local sound = Instance.new("Sound")
	sound.SoundId = assetId
	sound.SoundGroup = sfxGroup
	sound.RollOffMode = Enum.RollOffMode.InverseTapered
	sound.RollOffMinDistance = 8
	sound.RollOffMaxDistance = maxDistance or 80
	sound.PlaybackSpeed = rng:NextNumber(0.95, 1.05)
	sound.Parent = attachment

	sound.Ended:Once(function()
		attachment:Destroy()
	end)
	sound:Play()
end

return table.freeze(AudioManager)
~~~

---
**Ver también (enciclopedia):** [[Ambient Soundscapes - Layer Stacks, Overlapping Zones and Voice Pools|Audio Design]]
