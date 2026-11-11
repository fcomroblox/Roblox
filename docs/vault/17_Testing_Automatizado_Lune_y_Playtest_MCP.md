---
tags: [testing-automatizado-lune-y-playtest-mcp]
ultima_verificacion: pendiente
---
# 17. Testing Automatizado: Tests Unitarios con `Lune` y Simulación `Playtest` vía Studio MCP

Relacionado con: [[00_Mapa_del_Vault]] | [[CLAUDE]] | [[04_Patrones_y_Estructura_Rojo]] | [[09_Errores_Comunes_IA_Roblox]]

---

## 1. Pirámide de Verificación de Calidad en el Estudio IA
Ningún módulo `.luau` generado por Claude Code se considera terminado hasta que supera secuencialmente estos **4 niveles de verificación automatizada**:

| Nivel | Herramienta | Qué Verifica Exactamente | Comando / Mecanismo |
| :--- | :--- | :--- | :--- |
| **Nivel 1: Tipos y Sintaxis** | `luau-lsp analyze` + `selene` + `stylua` | Cumplimiento estricto de `--!strict`, inexistencia de variables `nil` no controladas y cero uso de APIs deprecadas. | `rojo sourcemap default.project.json -o sourcemap.json && luau-lsp analyze --sourcemap=sourcemap.json src/ && selene src/` |
| **Nivel 2: Lógica Matemática Pura** | `Lune` (`lune run`) | Fórmulas de economía, curvas de coste/XP, serializadores `buffer`, tablas de probabilidad y validadores anti-NaN en menos de $50\text{ ms}$ fuera de Studio. | `lune run tests/run_unit_tests.luau` |
| **Nivel 3: Integración en Motor** | Servidor MCP Nativo de Roblox Studio (`playtest`) | Arranque real de `init.server.luau` e `init.client.luau`, simulación de clic/ratón (`user_mouse_input`) y ausencia de errores en consola. | Herramientas MCP `playtest`, `get_studio_state`, `user_mouse_input`. |
| **Nivel 4: Telemetría Profunda** | `@chrrxs/robloxstudio-mcp` | Inspección en tiempo de ejecución de `LogService`, atributos en memoria del `DataModel` y verificación de que no queden instancias huérfanas. | Servidor MCP `@chrrxs/robloxstudio-mcp`. |

---

## 2. Estructura de Tests Unitarios con `Lune` (`tests/`)
Para ejecutar tests ultrarrápidos desde la terminal con `Lune` (instalado en `rokit.toml`), crea una carpeta `tests/` en la raíz de tu proyecto (fuera de `src/` para no enviarla al cliente de producción).

~~~luau
--!strict
-- Ejemplo: tests/test_economy_math.luau (Ejecutable con: lune run tests/test_economy_math.luau)
local process = require("@lune/process")
local stdio = require("@lune/stdio")

type TestSuite = {
	passed: number,
	failed: number,
}

local suite: TestSuite = { passed = 0, failed = 0 }

local function assertEqual(actual: unknown, expected: unknown, label: string)
	if actual == expected then
		suite.passed += 1
		stdio.write(`  [PASS] {label}\n`)
	else
		suite.failed += 1
		stdio.ewrite(`  [FAIL] {label} -> Esperado: {tostring(expected)}, Recibido: {tostring(actual)}\n`)
	end
end

-- Función bajo prueba (Fórmula de coste del GDD: floor(100 * 1.55^(n-1)))
local function calculateUpgradeCost(tier: number): number
	return math.floor(100 * (1.55 ^ (tier - 1)))
end

stdio.write("=== Ejecutando Batería de Tests Unitarios con Lune ===\n")
assertEqual(calculateUpgradeCost(1), 100, "Coste Nivel 1 debe ser 100")
assertEqual(calculateUpgradeCost(2), 155, "Coste Nivel 2 debe ser 155")
assertEqual(calculateUpgradeCost(3), 240, "Coste Nivel 3 debe ser 240")

stdio.write(`\nResultado Final: {suite.passed} pasados, {suite.failed} fallidos.\n`)
if suite.failed > 0 then
	process.exit(1)
end
~~~

---

## 3. Protocolo de Bucle Cerrado de Pruebas con el Subagente `playtest` de Studio MCP
Cuando Claude Code termina de escribir o refactorizar un servicio en `src/server/services/` y su controlador en `src/client/controllers/`, debe ejecutar este protocolo autónomo sin pedir al director humano que copie errores a mano:
1. Verificar que `rojo serve` ha inyectado los cambios en el DataModel usando `search_game_tree` o `inspect_instance`.
2. Lanzar la simulación con la herramienta MCP `playtest` y confirmar con `get_studio_state` que el motor está en ejecución.
3. Si la tarea involucra interfaz (`ScreenGui`) o interacción en el mundo, enviar pulsaciones simuladas con `user_mouse_input` y teclado.
4. Capturar todas las líneas de salida, advertencias (`warn`) y trazas de error (`error`) del servidor y del cliente mediante el MCP nativo y `@chrrxs/robloxstudio-mcp`.
5. Detener la simulación, corregir en los archivos `.luau` locales cualquier anomalía detectada y repetir la verificación hasta obtener **0 errores y 0 advertencias**.

---
**Ver también (enciclopedia):** [[Automated Testing with TestEZ and Jest-Lua|Testing]], [[Automated Luau Checks - LihFolk Hook and Audit|Automated Checks]]
