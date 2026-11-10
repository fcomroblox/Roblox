# Especificación Técnica: Sistema de Recolección, Entrega y Mejoras (`SPEC_CoreLoop.md`)

## Bloque 1: Reglas de Negocio y Ecuaciones Matemáticas Exactas
1. **Extracción de Nodos (`ResourceNode`):**
   - El mundo contiene instancias `Model` etiquetadas con `CollectionService` bajo el tag `"ResourceNode"` y el atributo numérico `NodeTier: number` ($1, 2, 3$).
   - Un jugador solo puede extraer un nodo si su `toolTier >= NodeTier`.
   - Tiempo mínimo entre golpes de extracción por jugador (*Cooldown*):
     $$\text{IntervaloMinimo}(\text{toolTier}) = \max(0.25, 0.85 - 0.10 \times (\text{toolTier} - 1))\text{ segundos}$$
   - Cantidad de recursos obtenidos por extracción exitosa:
     $$\text{CantidadExtraida} = \text{NodeTier} \times \text{toolTier}$$
   - La carga actual del jugador (`currentLoad`) no puede superar la capacidad máxima de su mochila:
     $$\text{CapacidadMaxima}(\text{backpackTier}) = 20 \times \text{backpackTier}$$
2. **Venta / Depósito en Terminal (`DepositTerminal`):**
   - Al acercarse a una instancia con el tag `"DepositTerminal"`, el jugador convierte toda su `currentLoad` en monedas (`coins`):
     $$\text{MonedasGanadas} = \lfloor \text{currentLoad} \times 10 \times (1 + 0.5 \times \text{prestigeCount}) \rfloor$$
   - Tras la entrega, `currentLoad` vuelve a $0$.
3. **Compra de Mejora de Herramienta o Mochila (`UpgradeStation`):**
   - El coste en `coins` para subir del nivel actual $n$ al nivel $n + 1$ ($n_{\text{max}} = 10$) es:
     $$\text{CosteMejora}(n) = \lfloor 100 \times (1.55^{(n - 1)}) \rfloor$$

## Bloque 2: Propiedades de Estado y Persistencia (Claves en ProfileStore)
- **Estado Persistente (Guardado en `Profile.Data` vía `DataService`):**
  - `coins: number` (Entero $\ge 0$, inicializado en $0$).
  - `toolTier: number` (Entero de $1\text{ a }10$, inicializado en $1$).
  - `backpackTier: number` (Entero de $1\text{ a }10$, inicializado en $1$).
  - `prestigeCount: number` (Entero $\ge 0$, inicializado en $0$).
  - `stats.totalExtractions: number` (Entero $\ge 0$, incrementado en $+1$ por cada extracción válida).
- **Estado Efímero de Sesión (Tabla en memoria del servidor indexada por `[Player]`, limpiada en `PlayerRemoving`):**
  - `currentLoad: number` (Recursos en la mochila aún no depositados, inicializado en $0$).
  - `lastExtractClock: number` (Marca de tiempo `os.clock()` de la última extracción válida).

## Bloque 3: Contratos de Red (`RemoteEvent` y Argumentos Tipados)
Ubicación en el DataModel: carpeta `ReplicatedStorage.Remotes` (creada en inicialización del servidor o definida en `src/shared/CoreLoopTypes.luau`):

| Nombre del RemoteEvent | Dirección | Firma de Argumentos Tipados | Propósito |
| :--- | :--- | :--- | :--- |
| `RequestExtractNode` | Cliente $\rightarrow$ Servidor | `(nodeModel: unknown)` | El cliente solicita extraer la instancia `Model` del nodo. No envía cantidades ni daño. |
| `RequestDepositLoad` | Cliente $\rightarrow$ Servidor | `(terminalModel: unknown)` | El cliente solicita vaciar su mochila en la terminal indicada. |
| `RequestBuyUpgrade` | Cliente $\rightarrow$ Servidor | `(upgradeCategory: unknown)` | El cliente envía `"Tool"` o `"Backpack"` para subir un nivel. |
| `SyncCoreLoopState` | Servidor $\rightarrow$ Cliente | `(payload: CoreLoopStatePayload)` | Notifica al cliente sus valores actualizados de `coins`, `currentLoad`, `maxCapacity`, `toolTier` y `backpackTier`. |

```luau
--!strict
export type UpgradeCategory = "Tool" | "Backpack"

export type CoreLoopStatePayload = {
	coins: number,
	currentLoad: number,
	maxCapacity: number,
	toolTier: number,
	backpackTier: number,
}
```

## Bloque 4: Casos Límite y Reglas de Seguridad (Validación de 5 Capas)
1. **Rate-Limiting en Servidor:**
   - `RequestExtractNode`: Descartar si `(os.clock() - lastExtractClock) < IntervaloMinimo(toolTier)`.
   - `RequestDepositLoad` y `RequestBuyUpgrade`: Cooldown mínimo estricto de $0.30\text{ segundos}$ por jugador.
2. **Validación `typeof` en Tiempo de Ejecución:**
   - Para `RequestExtractNode`: Comprobar `typeof(nodeModel) == "Instance"` y `(nodeModel :: Instance):IsA("Model")` y `CollectionService:HasTag(nodeModel :: Instance, "ResourceNode")`.
   - Para `RequestBuyUpgrade`: Comprobar `upgradeCategory == "Tool" or upgradeCategory == "Backpack"`.
3. **Validación Espacial de Magnitud en Servidor:**
   - Obtener `HumanoidRootPart` del `player.Character` y verificar que `Humanoid.Health > 0`.
   - Obtener la posición del objetivo (`nodeModel:GetPivot().Position` o `terminalModel:GetPivot().Position`).
   - Rechazar la petición si `(hrp.Position - targetPosition).Magnitude > 18` studs ($14\text{ studs}$ de alcance base $+ 4\text{ studs}$ de tolerancia de latencia móvil).
4. **Limpieza de Memoria:**
   - Al dispararse `Players.PlayerRemoving`, eliminar la entrada del jugador en las tablas de estado efímero y de cooldowns.