# 00. Mapa Central del Vault e Índice de Enrutamiento (Obsidian Graph)

Este documento actúa como el **nodo central (Hub)** de tu bóveda en Obsidian y como mapa de referencia cruzada para **Claude Code** y **Google Gemini**. Al contener enlaces directos en formato Wikilink (`[[ ]]`), conecta automáticamente todos los documentos de ingeniería, diseño y especificaciones en la vista de grafo de Obsidian.

---

## 1. Documentos Maestros de Sistema y Especificaciones (`docs/specs/` y `docs/`)
- [[CLAUDE]] — Reglas operativas maestras para Claude Code, comandos Rokit/Rojo y enrutamiento base.
- [[GEMINI]] — Directrices de sistema para Google Gemini (Arquitectura GDD, economía y arte 3D).
- [[art_bible]] — Biblia de Arte Técnico: paleta hexadecimal, rugosidad PBR, iluminación y tipografía.
- [[GDD_Master]] — Documento de Diseño Maestro: onboarding de $60\text{ s}$, bucles, fórmulas y KPIs.
- [[SPEC_CoreLoop]] — Especificación técnica del sistema principal de recolección, depósito y mejoras.
- [[MANIFEST_Assets.json]] — Diccionario técnico de modelos 3D, pantallas UI y pistas de audio.

---

## 2. Núcleo Fundacional del Vault (Archivos `01` a `10`)
- [[01_API_Roblox_y_Servicios]] — Servicios del DataModel, instancias de red y ciclo del Task Scheduler (`PreRender`, `PreAnimation`, `PreSimulation`, `PostSimulation`, `Heartbeat`).
- [[02_Guia_Luau_Tipado_Estricto]] — Sintaxis Luau moderna, `--!strict`, clases POO con metatables y reemplazo de APIs obsoletas.
- [[03_Seguridad_Cliente_Servidor]] — Autoridad del servidor, protocolo de 5 capas para `RemoteEvents` y control de `SetNetworkOwner`.
- [[04_Patrones_y_Estructura_Rojo]] — Arquitectura modular sin Knit, ciclo `:Init()` y `:Start()`, inyección de dependencias y limpieza con `Trove`.
- [[05_Datos_ProfileStore_Monetizacion]] — Persistencia con `ProfileStore`, bloqueo de sesiones y `MarketplaceService.ProcessReceipt` idempotente.
- [[06_Optimizacion_Movil_Streaming]] — Presupuesto de $60\text{ FPS}$ y $< 1.1\text{ GB}$ RAM, `StreamingEnabled`, `ModelStreamingMode.Atomic` y NPCs sin `Humanoid`.
- [[07_UI_UX_Responsive_Consola]] — Escalado con `UDim2.fromScale`, restricciones táctiles ($\ge 48 \times 48\text{ px}$), soporte de mando (`Gamepad`) y animaciones con `TweenService`.
- [[08_Normas_Moderacion_Politicas]] — Prevención de falsos positivos OCR en texturas IA, `PolicyService` para cajas aleatorias y separación Staging/Producción.
- [[09_Errores_Comunes_IA_Roblox]] — Solución a los 7 antipatrones clásicos de LLMs en Roblox (fugas en tablas `[Player]`, condiciones de carrera, `WaitForChild` infinito). **`[VERIFICAR]`**: Contenido de esta nota presenta anomalía, ver `_revision/ANOMALIA_09_Errores_Comunes.md`.
- [[10_Plantillas_Prompts]] — Biblioteca de prompts parametrizados para código Luau, depuración MCP, arte 3D, materiales PBR y greyboxing.

---

## 3. Expansión de Ingeniería Avanzada AAA (Archivos `11` a `21`)
- [[11_Fisica_Constraints_y_Raycasts_Avanzados]] — Sistema de ensamblajes (`Assemblies`), `LinearVelocity`, `AlignPosition`, `AlignOrientation`, `Shapecasts` (`Blockcast`/`Spherecast`) y `CollisionGroups`.
- [[12_Animaciones_R15_IKControl_y_Motor6D]] — Carga en `Animator`, cinemática inversa nativa (`IKControl`), manipulación procedural de `Motor6D.Transform` y eventos de marcadores.
- [[13_Audio_Modular_AudioPlayer_y_Wires]] — Nuevo sistema de audio de Roblox (`AudioPlayer`, `AudioEmitter`, `AudioListener`, `Wire`, efectos DSP en tiempo real y pooling de SFX).
- [[14_Redes_Avanzadas_Buffers_y_Compresion]] — Serialización binaria con la librería nativa `buffer` de Luau, `UnreliableRemoteEvent` (límite documentado: $1000\text{ bytes}$; presupuesto de diseño: $\le 900\text{ bytes}$; [fuente oficial](https://create.roblox.com/docs/reference/engine/classes/UnreliableRemoteEvent)), interpolación de snapshots y predicción en cliente.
- [[15_Matchmaking_MemoryStore_y_MessagingService]] — Colas multijugador entre servidores con `MemoryStoreService`, `TeleportService:TeleportAsync` a servidores reservados y `OrderedDataStore`.
- [[16_Pipeline_3D_Blender_PBR_y_MCP_Import]] — Automatización en Python para Blender (Decimate, pivotes, normales OpenGL $+Y$), `SurfaceAppearance` y subida por lotes vía Studio MCP.
- [[17_Testing_Automatizado_Lune_y_Playtest_MCP]] — Tests unitarios fuera de Studio con `Lune`, mocks tipados y protocolos de prueba autónoma con subagentes MCP y `@chrrxs/robloxstudio-mcp`.
- [[18_Open_Cloud_v2_CI_CD_y_Automatizacion]] — Uso de `open_cloud_invoke`, APIs REST de Open Cloud v2 para productos/lugares y pipeline de GitHub Actions.
- [[19_Analiticas_Telemetria_Funnel_y_Economia]] — Instrumentación completa de `AnalyticsService` (embudos, fuentes/sumideros de economía, progresión y retención D1/D7).
- [[20_Efectos_Visuales_VFX_Iluminacion_y_Jugabilidad_Juice]] — Presupuesto de `Highlight` (máximo 31 activos por jugador), `ParticleEmitter:Emit()`, sacudida de cámara en `PreRender` y feedback instantáneo.
- [[21_Seguridad_Avanzada_AntiCheat_Heuristico]] — Validación física en `PostSimulation` (anti-speedhack, anti-teleport, anti-fly con compensación de ping) y detección de anomalías económicas.

---

## 4. Serie Avanzada de Luau Estricto (Archivos `30_XX`)
- [[30_01_Modo_Strict_y_Anotaciones]] — Modo `--!strict`, anotaciones de tipo y reglas de inferencia.
- [[30_02_Tipos_Genericos_y_Type_Packs]] — Genéricos y Type Packs en Luau.
- [[30_04_Uniones_Discriminadas_e_Intersecciones]] — Uniones discriminadas e intersecciones de tipos.
- [[30_05_POO_con_Metatables_Tipadas]] — Programación orientada a objetos con metatables tipadas.
- [[30_06_Estructura_de_ModuleScripts_y_Ciclos]] — Estructura de ModuleScripts, ciclos de dependencia y resolución.
- [[30_07_Libreria_Task_y_Scheduler]] — Librería `task`, scheduler y reemplazo de APIs heredadas.
- [[30_08_Iteracion_Generalizada_y_Tablas]] — Iteración generalizada y manipulación de tablas.
- [[30_09_Libreria_Table_Prealojamiento_y_Freeze]] — Librería `table`, prealojamiento y `table.freeze`.
- [[30_10_Libreria_Buffer_Bitpacking_y_Cursores]] — Librería `buffer`, bitpacking y cursores para serialización.

---

## 5. Enciclopedia de Referencia (`encyclopedia/`)
Corpus de ~1.300 notas en inglés procedente de **LihFolk Vault V2** (consulta secundaria; las reglas del proyecto y `docs/vault/` tienen precedencia):
- [[encyclopedia/000 Home|🏠 Home]] — Dashboard de la enciclopedia.
- [[encyclopedia/ROUTER|🔀 Router]] — Navegación por intención ("Quiero construir...", "Necesito arreglar...").
- [[encyclopedia/SYMPTOMS|🩺 Symptoms]] — Índice de síntomas→soluciones.
- [[encyclopedia/INDEX_SLIM|📇 Index]] — Índice compacto de todas las notas.
- [[encyclopedia/Glossary|📖 Glossary]] — Glosario de términos Roblox.
- Secciones `01`-`62`: 59 carpetas temáticas con cobertura exhaustiva de Studio, Luau, Architecture, Data, UI, 3D, VFX, Animation (130 notas), Gameplay, Design, Monetization, Performance, Security, Physics, Math, Pipelines, AI/MCP, Case Studies, Teardowns, y más.
- `57 API and Concept Atlas` (310 notas): una ficha por cada API/concepto del motor.
- `62 Engine Reference Cards` (33 notas): cheat sheets compactos.

---

## 6. Herramientas y Agentes (`_tools/` y `_agents/`)
- `_tools/vault.py` — Buscador BM25F del vault enciclopédico.
- `_agents/` — Skills (22), hooks (desactivados), commands (6) y agents (3) de Claude Code.
- `_agents/AGENTS.md` — Configuración maestra de agentes IA.

---
**Conexiones del Vault:** [[CLAUDE]] | [[GEMINI]] | [[GDD_Master]] | [[art_bible]] | [[SPEC_CoreLoop]]
