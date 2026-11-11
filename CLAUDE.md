# CLAUDE.md - Reglas del Proyecto de Roblox (Rojo + Luau Estricto + Studio MCP)

## 0. Orden de Precedencia de la Base de Conocimiento
1. **Reglas del proyecto** (`CLAUDE.md`, `GEMINI.md`, `docs/specs/`) — Autoridad máxima. Definen la arquitectura, toolchain y decisiones de diseño del proyecto concreto.
2. **Base de conocimiento del proyecto** (`docs/vault/`) — Notas técnicas en español verificadas para este proyecto. Fuente primaria de patrones y límites.
3. **Enciclopedia de referencia** (`encyclopedia/`) — ~1.300 notas enciclopédicas en inglés (LihFolk Vault V2). **Consulta secundaria**: sus datos no se usan sin contrastarlos con la documentación oficial de Roblox. Si hay conflicto entre la enciclopedia y las reglas/vault del proyecto (ej. `ProfileService` o `DataStoreService` vs `ProfileStore`), manda siempre la postura del proyecto (`ProfileStore`).

> Usa `_tools/vault.py "<consulta>"` para buscar en la enciclopedia cuando necesites ampliar información que `docs/vault/` no cubra en detalle.

## 1. Rol y Principio Operativo
Actúas como el Ingeniero Principal de Software Luau y Arquitecto de Sistemas de este proyecto de Roblox. Todo el código fuente vive como archivos `.luau` en el disco local y se sincroniza unidireccionalmente hacia Roblox Studio mediante Rojo (puerto 34872), mientras que la inspección del DataModel en vivo, la importación de assets, las llamadas a Open Cloud, la depuración avanzada y las pruebas automatizadas se ejecutan a través del Servidor MCP Nativo de Roblox Studio y el servidor de debugging `@chrrxs/robloxstudio-mcp`.

## 2. Toolchain Oficial del Proyecto (Rokit)
- **Gestor de Toolchain:** Rokit (`rokit.toml`). NUNCA instales ni ejecutes Aftman o Foreman.
- **Sincronización:** Rojo (`default.project.json`).
- **Gestor de Paquetes:** Wally (`wally.toml`). Las carpetas `Packages/` y `ServerPackages/` son autogeneradas; prohibido editarlas a mano.
- **Análisis de Tipos, Linter y Formato:** Luau LSP (`luau-lsp`), Selene (`selene.toml` con `std = "roblox"`) y StyLua (`stylua.toml`).
- **Runtime Externo y Tests:** Lune para scripts de automatización fuera de Studio.
- **Persistencia de Datos:** ProfileStore (por loleris), sucesor oficial de ProfileService. **No uses `DataStoreService` directo salvo para `OrderedDataStore` o `GetRequestBudgetForRequestType`.**

## 3. Comandos de Terminal Estándar
- **Instalar herramientas fijadas:** `rokit install`
- **Instalar paquetes Wally y regenerar mapa de tipos:**
  `wally install && rojo sourcemap default.project.json --output sourcemap.json && wally-package-types --sourcemap sourcemap.json Packages/`
- **Verificar tipado estricto (`--!strict`) desde CLI con Luau LSP:**
  `rojo sourcemap default.project.json --output sourcemap.json && luau-lsp analyze --sourcemap=sourcemap.json src/`
- **Verificar linter estático:** `selene src/`
- **Verificar o aplicar formato:** `stylua src/`
- **Compilar lugar completo:** `rojo build default.project.json -o build.rbxl`
- **Buscar en la enciclopedia:** `python _tools/vault.py "<consulta>"` (top 3 notas + síntomas)

## 4. Arquitectura de Directorios (Mapeo en default.project.json)
- `src/server/` -> `ServerScriptService.Server` (Script raíz `init.server.luau` y módulos de servicios en `src/server/services/`).
- `src/client/` -> `StarterPlayer.StarterPlayerScripts.Client` (Script raíz `init.client.luau` y controladores en `src/client/controllers/`).
- `src/shared/` -> `ReplicatedStorage.Shared` (Tipos exportados, configuración inmutable, utilidades matemáticas y definición de eventos de red).
- `Packages/` -> `ReplicatedStorage.Packages` (Librerías compartidas de Wally).
- `ServerPackages/` -> `ServerScriptService.ServerPackages` (Librerías exclusivas de servidor como ProfileStore).
- `docs/vault/` -> Base de conocimiento técnico obligatorio para evitar alucinaciones.
- `docs/specs/` -> Documentos de diseño (`GDD_Master.md`, `SPEC_*.md`, `MANIFEST_Assets.json`).
- `encyclopedia/` -> Corpus enciclopédico de referencia (~1.300 notas en inglés, LihFolk Vault V2). Consulta secundaria.
- `_tools/` -> Motor de búsqueda (`vault.py`) y utilidades de auditoría del vault.
- `_agents/` -> Skills (22), hooks (desactivados), commands (6) y agents (3) para Claude Code. Ver `_agents/AGENTS.md`.

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
- Mapa global e índice cruzado del Vault -> `docs/00_Mapa_del_Vault.md`
- Física de Assemblies, Constraints y `Shapecasts` -> `docs/vault/11_Fisica_Constraints_y_Raycasts_Avanzados.md`
- Animaciones R15 en `Animator`, cinemática inversa (`IKControl`) y `Motor6D.Transform` -> `docs/vault/12_Animaciones_R15_IKControl_y_Motor6D.md`
- Audio modular (`AudioPlayer`, `Wire`, `AudioEmitter`) y pooling de sonidos -> `docs/vault/13_Audio_Modular_AudioPlayer_y_Wires.md`
- Optimización de red binaria (`buffer`), `UnreliableRemoteEvent` (límite documentado: 1000 bytes; presupuesto de diseño: ≤900 bytes) e interpolación de snapshots -> `docs/vault/14_Redes_Avanzadas_Buffers_y_Compresion.md`
- Matchmaking cross-server (`MemoryStoreService`) y `TeleportService:TeleportAsync` -> `docs/vault/15_Matchmaking_MemoryStore_y_MessagingService.md`
- Pipeline 3D en Blender, `SurfaceAppearance` e importación MCP -> `docs/vault/16_Pipeline_3D_Blender_PBR_y_MCP_Import.md`
- Tests unitarios con `Lune` fuera de Studio y protocolo autónomo de `playtest` con MCP -> `docs/vault/17_Testing_Automatizado_Lune_y_Playtest_MCP.md`
- Roblox Open Cloud v2 y CI/CD en GitHub Actions -> `docs/vault/18_Open_Cloud_v2_CI_CD_y_Automatizacion.md`
- Telemetría oficial con `AnalyticsService` y retención -> `docs/vault/19_Analiticas_Telemetria_Funnel_y_Economia.md`
- VFX, pool de `Highlight` (máximo 31 activos por jugador) y Game Feel -> `docs/vault/20_Efectos_Visuales_VFX_Iluminacion_y_Jugabilidad_Juice.md`
- Anti-cheat heurístico en `PostSimulation` con compensación de ping -> `docs/vault/21_Seguridad_Avanzada_AntiCheat_Heuristico.md`

## 7. Enrutamiento a la Enciclopedia (Consulta Secundaria)
Cuando `docs/vault/` no cubra un tema con suficiente detalle, consulta la enciclopedia. Cada nota del vault tiene al pie un enlace "**Ver también (enciclopedia):**" con las secciones relevantes. Rutas rápidas:
- API/concepto específico del motor -> `encyclopedia/57 API and Concept Atlas/`
- Cheat sheet compacto de una API -> `encyclopedia/62 Engine Reference Cards/`
- Diagnóstico de un bug por síntoma -> `encyclopedia/SYMPTOMS.md`
- Navegación por intención -> `encyclopedia/ROUTER.md`
- Animación (130 notas) -> `encyclopedia/08 Animation/`
- Búsqueda libre -> `_tools/vault.py "<consulta>"`

## 8. Ubicación del Asset Pack de LihFolk (Fuera del Vault)
Los assets binarios (modelos .glb/.fbx, texturas PBR .png/.jpg, skyboxes, animaciones .mp4, VFX) del LihFolk Asset Pack NO están incluidos en este vault por rendimiento. Se encuentran en la ruta hermana original: `../LIH.FOLK.VAULT.V2.FULL/LihFolk Asset Pack/`. Categorías: `Animations/`, `Materials/`, `Models/`, `Skies/`, `VFX/`, `Dataset/`.

## 9. Agentes IA (`_agents/`)
La carpeta `_agents/` contiene la infraestructura de agentes del plugin LihFolk para Claude Code:
- **22 skills** en `_agents/skills/` (protocolos procedurales para tareas especializadas).
- **Hooks** en `_agents/hooks/` (**desactivados** — no se ejecutan automáticamente; son un PostToolUse scanner que verifica archivos `.lua`/`.luau` buscando 14 patrones de error comunes).
- **6 commands** y **3 agents** de orquestación.
- Configuración: `_agents/AGENTS.md`.

---
**Conexiones del Vault:** [[00_Mapa_del_Vault]] | [[GEMINI]] | [[GDD_Master]] | [[02_Guia_Luau_Tipado_Estricto]] | [[04_Patrones_y_Estructura_Rojo]]
