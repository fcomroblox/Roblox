---
tags: [seguridad-avanzada-anticheat-heuristico]
ultima_verificacion: pendiente
---
# 21. Seguridad Avanzada: Anti-Cheat Heurístico de Movimiento en Servidor y Auditoría Económica

Relacionado con: [[00_Mapa_del_Vault]] | [[03_Seguridad_Cliente_Servidor]] | [[11_Fisica_Constraints_y_Raycasts_Avanzados]] | [[19_Analiticas_Telemetria_Funnel_y_Economia]]

---

## 1. Por Qué el Anti-Cheat de Movimiento Debe Vivir en `RunService.PostSimulation`
Como el cliente tiene autoridad física sobre su propio personaje (`Character`) para que el movimiento se sienta instantáneo sin input-lag, un usuario con exploits puede alterar su `WalkSpeed`, teletransportarse a los nodos de recursos o volar (`NoClip` / `Fly`).
- **Regla Fundamental:** Nunca pongas scripts anti-cheat en el cliente (`LocalScript`), ya que un atacante puede desactivarlos en 1 segundo. Toda verificación de movimiento se ejecuta en el **servidor** dentro de `RunService.PostSimulation` (justo después de que el paso físico ha actualizado las posiciones replicadas).
- **Evitar Falsos Positivos por Lag (`GetNetworkPing`):** En lugar de expulsar (`player:Kick()`) al primer pico de velocidad (que podría ser un tirón de conexión Wi-Fi móvil o el empuje de una explosión legítima), el servidor acumula **puntos de infracción (*Violation Score*)** que decaen con el tiempo y devuelve silenciosamente al personaje a su última posición válida (*Rubberbanding / Rollback*).

---

## 2. Implementación de Referencia: `MovementValidatorService.luau`

~~~luau
--!strict
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local MovementValidatorService = {}

type PlayerMovementRecord = {
	lastValidCFrame: CFrame,
	lastCheckClock: number,
	violationScore: number,
	graceUntilClock: number,
}

local records: { [Player]: PlayerMovementRecord } = {}
local CHECK_INTERVAL = 0.25 -- Evaluar 4 veces por segundo para ahorrar CPU en servidor
local MAX_VERTICAL_ASCENT_PER_SEC = 58.0 -- Tolerancia sobre JumpPower estándar

-- Llamar desde otros servicios del servidor cuando el juego teletransporta legítimamente al jugador
function MovementValidatorService.GrantTeleportGrace(player: Player, destinationCFrame: CFrame, durationSec: number?)
	local rec = records[player]
	if rec then
		rec.lastValidCFrame = destinationCFrame
		rec.graceUntilClock = os.clock() + (durationSec or 1.5)
	end
end

function MovementValidatorService.Init()
	Players.PlayerRemoving:Connect(function(player: Player)
		records[player] = nil
	end)
end

function MovementValidatorService.Start()
	RunService.PostSimulation:Connect(function()
		local now = os.clock()

		for _, player in Players:GetPlayers() do
			local character = player.Character
			local hrp = if character then character:FindFirstChild("HumanoidRootPart") :: BasePart? else nil
			local humanoid = if character then character:FindFirstChildOfClass("Humanoid") else nil

			if not hrp or not humanoid or humanoid.Health <= 0 or humanoid.Sit then
				records[player] = nil
				continue
			end

			local rec = records[player]
			if not rec then
				records[player] = {
					lastValidCFrame = hrp.CFrame,
					lastCheckClock = now,
					violationScore = 0,
					graceUntilClock = now + 1.0,
				}
				continue
			end

			local dt = now - rec.lastCheckClock
			if dt < CHECK_INTERVAL then
				continue
			end
			rec.lastCheckClock = now

			if now < rec.graceUntilClock then
				rec.lastValidCFrame = hrp.CFrame
				continue
			end

			local currentPos = hrp.Position
			local lastPos = rec.lastValidCFrame.Position
			local deltaVector = currentPos - lastPos

			-- Distancia horizontal (XZ) y ascenso vertical (Y)
			local horizontalDist = Vector3.new(deltaVector.X, 0, deltaVector.Z).Magnitude
			local verticalGain = deltaVector.Y

			-- Compensación dinámica por ping del jugador (máx +300ms de margen)
			local pingSeconds = math.clamp(player:GetNetworkPing(), 0.02, 0.30)
			local allowedSpeed = humanoid.WalkSpeed * 1.35 + 8.0
			local maxHorizontalAllowed = allowedSpeed * (dt + pingSeconds * 0.5)
			local maxVerticalAllowed = MAX_VERTICAL_ASCENT_PER_SEC * (dt + pingSeconds * 0.5)

			if horizontalDist > maxHorizontalAllowed or verticalGain > maxVerticalAllowed then
				rec.violationScore += 1.0
				-- Rollback inmediato a la última posición válida si acumula 2 infracciones seguidas
				if rec.violationScore >= 2.0 then
					hrp.AssemblyLinearVelocity = Vector3.zero
					hrp.CFrame = rec.lastValidCFrame
				end
			else
				rec.lastValidCFrame = hrp.CFrame
				rec.violationScore = math.max(0, rec.violationScore - dt * 0.5)
			end
		end
	end)
end

return table.freeze(MovementValidatorService)
~~~

---

## 3. Trampas de Red (*Honeypot Remotes*) y Límites de Ganancia Económica
1. **Límite Superior de Ganancia por Transacción (*Sanity Cap*):** En el `DataService`, cualquier función que sume monedas (`AddCoins(player, amount)`) debe tener un techo máximo configurado según la fórmula matemática del nivel actual del jugador. Si por un bug lógico una función intenta sumar $+1.000.000\text{ monedas}$ en un solo segundo a un jugador de Nivel 1, se bloquea la operación y se registra una alerta en `AnalyticsService`.
2. **Validación de NaN y Números Negativos en Compras:** Nunca restes `coins -= precioEnviado` si el precio pudiera ser negativo (ya que restar un número negativo **suma** dinero infinito al jugador). El precio se lee siempre de una tabla congelada (`table.freeze`) en el servidor.

---
**Ver también (enciclopedia):** [[Anti-Exploit Start Here|Anti-Exploit Architecture]]
