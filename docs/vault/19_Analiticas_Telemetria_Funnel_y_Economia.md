---
tags: [analiticas-telemetria-funnel-y-economia]
ultima_verificacion: pendiente
---
# 19. Instrumentación de `AnalyticsService`, Embudos de Retención y Economía

Relacionado con: [[00_Mapa_del_Vault]] | [[GDD_Master]] | [[05_Datos_ProfileStore_Monetizacion]] | [[SPEC_CoreLoop]]

---

## 1. Los 4 Eventos Oficiales de `AnalyticsService` en Roblox
Para que los gráficos del panel **Creator Hub $\rightarrow$ Analytics** (Onboarding, Economy, Progression y Custom Events) se pueblen automáticamente sin pagar herramientas externas, el servidor debe registrar los eventos mediante este módulo envoltorio seguro (siempre dentro de `pcall` para que un fallo de telemetría jamás interrumpa el bucle de juego):

| Método de `AnalyticsService` | Cuándo Dispararlo en el Servidor | Qué Mide en el Creator Dashboard |
| :--- | :--- | :--- |
| `LogOnboardingFunnelStepEvent` | Una sola vez por cada uno de los primeros $5\text{ a }8$ hitos del primer minuto de un jugador nuevo. | Dónde abandonan el juego los usuarios nuevos antes de entender el bucle principal. |
| `LogEconomyEvent` | Cada vez que un jugador **gana** (`Source`) o **gasta** (`Sink`) monedas (`coins`) o gemas (`gems`). | Inflación monetaria del servidor, balance de fuentes vs. sumideros y qué mejoras son las más compradas. |
| `LogProgressionEvent` | Al iniciar (`Start`), completar (`Complete`) o fallar (`Fail`) una zona, oleada o expedición. | Curva de dificultad y tasa de victoria/derrota por nivel o profundidad. |
| `LogCustomEvent` | Acciones específicas de diseño (ej. abrir tienda, usar habilidad cooperativa, equipar mascota). | Frecuencia de uso de características secundarias. |

---

## 2. Implementación de Referencia: `TelemetryService.luau`

~~~luau
--!strict
local AnalyticsService = game:GetService("AnalyticsService")
local RunService = game:GetService("RunService")

local TelemetryService = {}

-- Evitar contaminar las métricas reales cuando probamos en Roblox Studio
local IS_STUDIO = RunService:IsStudio()

function TelemetryService.LogOnboardingStep(player: Player, stepNumber: number, stepName: string)
	if IS_STUDIO then
		return
	end
	task.spawn(function()
		pcall(function()
			AnalyticsService:LogOnboardingFunnelStepEvent(player, stepNumber, stepName)
		end)
	end)
end

function TelemetryService.LogCurrencyEarned(
	player: Player,
	currencyType: string,
	amountEarned: number,
	newEndingBalance: number,
	transactionCategory: string,
	itemSku: string?
)
	if IS_STUDIO or amountEarned <= 0 then
		return
	end
	task.spawn(function()
		pcall(function()
			AnalyticsService:LogEconomyEvent(
				player,
				Enum.AnalyticsEconomyFlowType.Source,
				currencyType,
				amountEarned,
				newEndingBalance,
				transactionCategory,
				itemSku or "DefaultSource"
			)
		end)
	end)
end

function TelemetryService.LogCurrencySpent(
	player: Player,
	currencyType: string,
	amountSpent: number,
	newEndingBalance: number,
	transactionCategory: string,
	itemSku: string
)
	if IS_STUDIO or amountSpent <= 0 then
		return
	end
	task.spawn(function()
		pcall(function()
			AnalyticsService:LogEconomyEvent(
				player,
				Enum.AnalyticsEconomyFlowType.Sink,
				currencyType,
				amountSpent,
				newEndingBalance,
				transactionCategory,
				itemSku
			)
		end)
	end)
end

return table.freeze(TelemetryService)
~~~

---

## 3. Mecánicas Probadas para Elevar Retención D1, D7 y Tiempo de Sesión
1. **Para subir la Retención Día 1 ($> 14\text{-}20\%$):**
   - **Pre-compromiso del Próximo Desbloqueo (*Next Session Goal*):** Nunca dejes al jugador sin saber qué viene después. Muestra en el HUD una barra de progreso hacia un hito muy atractivo (ej. *"Submarino Nivel 3 — 75% completado"*) y guarda ese progreso exacto en `ProfileStore`.
   - **Recompensa de Retorno de 24 Horas (Daily Login Streak):** Almacena `lastClaimUnixTime: number` y `streakDay: number` en `ProfileStore`. Verifica en el servidor que hayan pasado al menos $20\text{ horas}$ (`os.time() - lastClaimUnixTime >= 72000`) antes de otorgar el siguiente día.
2. **Para subir el Tiempo Medio de Sesión ($> 15\text{-}20\text{ min}$):**
   - **Eventos Globales Cíclicos cada $12\text{-}15\text{ minutos}$:** Un contador visible en el servidor anuncia: *"Tormenta de Cristales Épicos en 03:45"*. Los jugadores que pensaban desconectarse a los $10\text{ minutos}$ se quedan para aprovechar el evento.

---
**Ver también (enciclopedia):** [[Analytics and Funnels|Analytics]], [[Economy Design Start Here|Economy Design]]
