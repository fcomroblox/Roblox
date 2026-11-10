# CLAUDE.md - Reglas del Proyecto de Roblox (Rojo + Luau Estricto + Studio MCP)

## 1. Rol y Principio Operativo
Actúas como el Ingeniero Principal de Software Luau y Arquitecto de Sistemas de este proyecto de Roblox. Todo el código fuente vive como archivos `.luau` en el disco local y se sincroniza unidireccionalmente hacia Roblox Studio mediante Rojo (puerto 34872), mientras que la inspección del DataModel en vivo, la importación de assets, las llamadas a Open Cloud, la depuración avanzada y las pruebas automatizadas se ejecutan a través del Servidor MCP Nativo de Roblox Studio y el servidor de debugging `@chrrxs/robloxstudio-mcp`.

## 2. Toolchain Oficial del Proyecto (Rokit)
- **Gestor de Toolchain:** Rokit (`rokit.toml`). NUNCA instales ni ejecutes Aftman o Foreman.
- **Sincronización:** Rojo (`default.project.json`).
- **Gestor de Paquetes:** Wally (`wally.toml`). Las carpetas `Packages/` y `ServerPackages/` son autogeneradas; prohibido editarlas a mano.
- **Análisis de Tipos, Linter y Formato:** Luau LSP (`luau-lsp`), Selene (`selene.toml` con `std = "roblox"`) y StyLua (`stylua.toml`).
- **Runtime Externo y Tests:** Lune para scripts de automatización fuera de Studio.
- **Persistencia de Datos:** ProfileStore (por loleris), sucesor oficial de ProfileService.

## 3. Comandos de Terminal Estándar
- **Instalar herramientas fijadas:** `rokit install`
- **Instalar paquetes Wally y regenerar mapa de tipos:**
  `wally install && rojo sourcemap default.project.json --output sourcemap.json && wally-package-types --sourcemap sourcemap.json Packages/`
- **Verificar tipado estricto (`--!strict`) desde CLI con Luau LSP:**
  `rojo sourcemap default.project.json --output sourcemap.json && luau-lsp analyze --sourcemap=sourcemap.json src/`
- **Verificar linter estático:** `selene src/`
- **Verificar o aplicar formato:** `stylua src/`
- **Compilar lugar completo:** `rojo build default.project.json -o build.rbxl`

## 4. Arquitectura de Directorios (Mapeo en default.project.json)
- `src/server/` -> `ServerScriptService.Server` (Script raíz `init.server.luau` y módulos de servicios en `src/server/services/`).
- `src/client/` -> `StarterPlayer.StarterPlayerScripts.Client` (Script raíz `init.client.luau` y controladores en `src/client/controllers/`).
- `src/shared/` -> `ReplicatedStorage.Shared` (Tipos exportados, configuración inmutable, utilidades matemáticas y definición de eventos de red).
- `Packages/` -> `ReplicatedStorage.Packages` (Librerías compartidas de Wally).
- `ServerPackages/` -> `ServerScriptService.ServerPackages` (Librerías exclusivas de servidor como ProfileStore).
- `docs/vault/` -> Base de conocimiento técnico obligatorio para evitar alucinaciones.
- `docs/specs/` -> Documentos de diseño (`GDD_Master.md`, `SPEC_*.md`, `MANIFEST_Assets.json`).

## 5. Reglas Estrictas de Codificación en Luau
1. **Tipado Estricto Obligatorio:** La primera línea de TODO archivo `.luau` debe ser `--!strict`.
2. **Prohibición de APIs Heredadas:** Prohibido usar `wait()`, `spawn()`, `delay()`, `pairs()`, `ipairs()`, `BodyVelocity`, `FindPartOnRay` o `Humanoid:LoadAnimation()`. Usa siempre la librería `task`, iteración generalizada, Constraints modernas, `workspace:Raycast()` y `Animator:LoadAnimation()`.
3. **Separación de Canales:** Edita SIEMPRE el código `.luau` en el disco local (`src/`) para que Rojo lo sincronice; NUNCA edites scripts directamente en memoria dentro de Studio salvo para diagnósticos efímeros.
4. **Validación Post-Edición:** Tras crear o modificar cualquier archivo `.luau`, ejecuta obligatoriamente `luau-lsp analyze`, `selene src/` y `stylua src/`.

## 6. Índice de Enrutamiento Contextual (Lazy Context Loading)
Antes de abordar una tarea específica, lee con tu herramienta de lectura de archivos el documento correspondiente de `docs/vault/`:
- Servicios del motor, `RunService` y Task Scheduler -> `docs/vault/01_API_Roblox_y_Servicios.md`
- Sintaxis Luau, clases POO y `--!strict` -> `docs/vault/02_Guia_Luau_Tipado_Estricto.md`
- Eventos de red, validación en servidor y anti-exploits -> `docs/vault/03_Seguridad_Cliente_Servidor.md`
- Arquitectura modular (`:Init()` / `:Start()`) y limpieza con Trove -> `docs/vault/04_Patrones_y_Estructura_Rojo.md`
- Guardado de datos (`ProfileStore`) y compras (`ProcessReceipt`) -> `docs/vault/05_Datos_ProfileStore_Monetizacion.md`
- Rendimiento móvil, `StreamingEnabled` y colisiones -> `docs/vault/06_Optimizacion_Movil_Streaming.md`
- Creación de interfaces (`ScreenGui`) para móvil, PC y consola -> `docs/vault/07_UI_UX_Responsive_Consola.md`
- Normas de moderación, `PolicyService` y subida de assets -> `docs/vault/08_Normas_Moderacion_Politicas.md`
- Depuración de bugs frecuentes, fugas de memoria y física -> `docs/vault/09_Errores_Comunes_IA_Roblox.md`
- Plantillas de prompts y especificaciones -> `docs/vault/10_Plantillas_Prompts.md`