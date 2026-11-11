# GDD Maestro: [TÍTULO DEL PROYECTO, ej. Deep Submarines: Co-op Extraction]

## 1. Visión Ejecutiva y Parámetros del Motor
- **Concepto en 1 línea:** [Acción principal + Entorno + Factor social/diferenciador, ej. Exploración y extracción cooperativa en submarinos donde de 1 a 4 jugadores recolectan reliquias abisales, mejoran su nave y escapan antes de quedarse sin oxígeno].
- **Género Oficial en Roblox:** Survival Co-op / Extraction / Action Tycoon
- **Dispositivos Objetivo:** Móvil Táctil (Prioridad 1 - $60\text{ FPS}$ y $< 1.1\text{ GB}$ RAM), PC Teclado/Ratón (Prioridad 2), Consola Gamepad (Prioridad 3).
- **Jugadores por Servidor (`MaxPlayers`):** $16$ en Lobby Social / $1\text{ a }4$ en Servidor Reservado de Partida (`TeleportService:ReserveServer`).
- **Configuración de Mundo:** `Workspace.StreamingEnabled = true` (`StreamingMinRadius = 64`, `StreamingTargetRadius = 256`), `Lighting.Technology = Enum.Technology.ShadowMap`.

## 2. Flujo de los Primeros 60 Segundos (Onboarding Funnel)
Se instrumenta cada paso con `AnalyticsService:LogOnboardingFunnelStepEvent(player, stepNumber, stepName)`:
- **Paso 1 (Segundos 0-10 - `SpawnedInLobby`):** Aparición en `SpawnLocation` mirando directamente hacia la estación de recolección inicial a menos de $48\text{ studs}$ ($3\text{ segundos}$ a pie), guiado por un `Beam` luminoso animado.
- **Paso 2 (Segundos 10-20 - `FirstResourceCollected`):** El jugador interactúa con un único toque/clic (`ProximityPrompt` o área de contacto) y recolecta su primer recurso con feedback inmediato en cliente (`TweenService` + partículas + sonido con `PitchShiftSoundEffect`).
- **Paso 3 (Segundos 20-35 - `FirstResourceSold`):** El `Beam` redirige a la terminal de entrega situada a $24\text{ studs}$, convirtiendo el recurso en las primeras $100\text{ monedas}$ (`coins`).
- **Paso 4 (Segundos 35-50 - `FirstUpgradePurchased`):** Se abre el panel modal responsive y el jugador compra la Mejora Nivel 1 de su herramienta/vehículo.
- **Paso 5 (Segundos 50-60 - `EnteredFirstZone`):** Se desbloquea la compuerta hacia la Zona 1 y se muestra el objetivo de los próximos $10\text{ minutos}$ (desbloquear el siguiente rango de profundidad).

## 3. El Bucle Principal y Fórmulas de Economía
- **Micro-Bucle ($30\text{ s}$):** Explorar nodo cercano $\rightarrow$ Extraer recurso validado en servidor $\rightarrow$ Recibir feedback sensorial instantáneo y monedas al depositar.
- **Meso-Bucle ($5\text{-}10\text{ min}$):** Invertir monedas en mejorar capacidad de carga, velocidad o linterna $\rightarrow$ Acceder a zonas de mayor profundidad y multiplicador de valor.
- **Macro-Bucle (Sesiones / Largo Plazo):** Alcanzar el nivel máximo de nave $\rightarrow$ Ejecutar Prestigio (*Rebirth*) para obtener multiplicador permanente y desbloquear cosméticos de colección y clasificación en `OrderedDataStore`.
- **Fórmula de Coste de Mejora de Nivel $n$:**
  $$\text{Coste}(n) = \lfloor \text{CosteBase} \times (1.16^{(n - 1)}) \rfloor$$
- **Fórmula de Ganancia por Extracción de Nivel $n$:**
  $$\text{Valor}(n) = \lfloor \text{ValorBase} \times (1 + 0.25 \times (n - 1)) \times \text{MultiplicadorPrestigio} \rfloor$$

## 4. Esquema de Datos en ProfileStore (`PlayerDataSchema`)
Este esquema define la estructura obligatoria de `DATA_TEMPLATE` en `src/server/services/DataService.luau`:

```luau
--!strict
export type PlayerDataSchema = {
	schemaVersion: number,
	coins: number,
	gems: number,
	prestigeCount: number,
	toolTier: number,
	backpackTier: number,
	unlockedItems: { [string]: boolean },
	stats: {
		totalExtractions: number,
		highestDepthReached: number,
		totalPlayTimeSeconds: number,
	},
	processedReceipts: { [string]: boolean },
}
```

## 5. Lista Cerrada de Módulos para el MVP (Versión 1.0)
1. **`DataService.luau`:** Gestión de sesión con `ProfileStore`, auto-guardado cada $300\text{ s}$, reconciliación de esquema y `MarketplaceService.ProcessReceipt` idempotente.
2. **`CoreLoopService.luau` / `CoreLoopController.luau`:** Spawning de nodos de recursos con `CollectionService`, validación de extracción por distancia/cooldown en servidor y efectos visuales en cliente.
3. **`ProgressionShopService.luau` / `ProgressionShopController.luau`:** Catálogo inmutable de niveles de herramientas/mochila y validación de compra autoritativa.
4. **`UIController.luau`:** HUD principal adaptable (`UDim2.fromScale`) y ventana de tienda con soporte táctil ($\ge 48 \times 48\text{ px}$) y mando de consola (`GuiService.SelectedObject`).

## 6. Catálogo Inicial de Monetización Ética y Social
- **Game Passes (Compras Únicas):**
  1. **Pase VIP Explorador ($199\text{ R\$}$):** $+25\%$ de velocidad de movimiento en base, etiqueta dorada en chat y cosmético de linterna exclusivo.
  2. **Almacén Magnético Automático ($149\text{ R\$}$):** Recolección automática en radio corto y $+50\%$ de capacidad de mochila base.
- **Developer Products (Consumibles Repetibles en `ProcessReceipt`):**
  1. **Evento de Servidor: Lluvia de Reliquias Abisales ($99\text{ R\$}$):** Duplica la aparición de recursos raros para **todos los jugadores del servidor** durante $10\text{ minutos}$ y anuncia el agradecimiento en el chat global.
  2. **Paquete de $250\text{ Gemas}$ ($49\text{ R\$}$):** Moneda secundaria para cosméticos directos.
  3. **Paquete de $1.000\text{ Gemas}$ ($169\text{ R\$}$):** Moneda secundaria con bonificación del $20\%$.

## 7. Benchmarks de Lanzamiento (Go/No-Go KPIs en Roblox Analytics)
- **Embudo de Onboarding (Paso 5 completado):** $> 80\%$
- **Duración Media de Sesión (Average Session Time):** $> 15\text{ minutos}$
- **Retención Día 1 (D1 Retention):** $> 14\%$ (Objetivo de escalado: $> 20\%$)
- **Qualified Play Through Rate (qPTR):** $> 2.5\%$
- **Tasa de Sesiones Libres de Fallos (Crash-Free Rate):** $> 99.2\%$

---
**Conexiones del Vault:** [[00_Mapa_del_Vault]] | [[SPEC_CoreLoop]] | [[art_bible]] | [[05_Datos_ProfileStore_Monetizacion]] | [[19_Analiticas_Telemetria_Funnel_y_Economia]]
