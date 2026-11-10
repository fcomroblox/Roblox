# 10. Biblioteca de Plantillas de Prompts Maestros (Claude y Gemini)

## 1. Prompt para Generar Módulos Luau Estrictos (Claude Code)
```text
ROL Y CONTEXTO:
Actúa como Ingeniero Principal Luau del proyecto. Antes de escribir código, lee con tu herramienta de lectura:
1. /CLAUDE.md
2. /docs/vault/02_Guia_Luau_Tipado_Estricto.md

TAREA:
Crea el módulo [NOMBRE_MODULO] en la ruta local src/shared/[NOMBRE_MODULO].luau (sincronizado por Rojo hacia ReplicatedStorage.Shared).
Funcionalidad requerida: [DESCRIBIR FÓRMULAS, REGLAS DE NEGOCIO Y MÉTODOS PÚBLICOS].

RESTRICCIONES TÉCNICAS:
- Encabezado obligatorio en línea 1: --!strict.
- Define y exporta todos los tipos de datos con export type.
- Anota explícitamente los tipos de todos los argumentos y retornos de cada función.
- Prohibido usar wait(), spawn(), pairs(), ipairs() o conversiones :: any.
- Congela la tabla del módulo al final con return table.freeze([NOMBRE_MODULO]).
- Permite inyectar servicios de Roblox como parámetro opcional en el constructor para facilitar tests unitarios con Lune.

CRITERIOS DE ACEPTACIÓN:
1. Escribe el archivo en src/shared/[NOMBRE_MODULO].luau.
2. Ejecuta en la terminal:
   rojo sourcemap default.project.json --output sourcemap.json && luau-lsp analyze --sourcemap=sourcemap.json src/shared/[NOMBRE_MODULO].luau
   selene src/shared/[NOMBRE_MODULO].luau
   stylua --check src/shared/[NOMBRE_MODULO].luau
3. Corrige cualquier error de tipos o aviso antes de finalizar.
```

## 2. Prompt para Sistemas Completos Cliente-Servidor (Claude Code)
```text
ROL Y CONTEXTO:
Actúa como Arquitecto de Sistemas de Roblox. Lee obligatoriamente antes de empezar:
1. /CLAUDE.md
2. /docs/vault/03_Seguridad_Cliente_Servidor.md
3. /docs/vault/05_Datos_ProfileStore_Monetizacion.md
4. La especificación funcional en /docs/specs/SPEC_[SISTEMA].md

TAREA:
Implementa el sistema completo [NOMBRE_SISTEMA] dividiéndolo en tres archivos sincronizados por Rojo:
- src/shared/[Sistema]Types.luau: Tipos exportados (export type), catálogo congelado de constantes (table.freeze) y referencias a los RemoteEvents.
- src/server/services/[Sistema]Service.luau: Lógica autoritativa en servidor, integración con DataService (ProfileStore) y validación de seguridad.
- src/client/controllers/[Sistema]Controller.luau: Escucha de eventos del servidor, actualización de UI y disparo de intenciones del jugador.

RESTRICCIONES DE SEGURIDAD Y ARQUITECTURA:
- El cliente jamás envía costes, daño ni cantidades; solo envía el identificador de la acción o ítem.
- Cada OnServerEvent en el servidor debe aplicar: (1) Rate-limiting con os.clock() por jugador, (2) Validación typeof() y chequeo anti-NaN (x == x), (3) Verificación de distancia física en el servidor si interactúa con una instancia del Workspace.
- Prohibido usar RemoteFunction:InvokeClient().
- Limpia todas las tablas indexadas por [Player] en Players.PlayerRemoving.

CRITERIOS DE ACEPTACIÓN:
Crea los tres archivos en src/, ejecuta luau-lsp analyze, selene src/ y stylua src/ sin errores, y ejecuta un playtest corto mediante el servidor MCP de Studio para confirmar en la consola que el servicio y el controlador arrancan sin excepciones.
```

## 3. Prompt para Depurar Errores con Studio MCP y Debugging MCP (Claude Code)
```text
ROL Y CONTEXTO:
Actúa como Especialista en Depuración de Luau y Motor de Roblox. Lee /docs/vault/09_Errores_Comunes_IA_Roblox.md.

PROBLEMA REPORTADO:
Al ejecutar la acción [DESCRIBIR ACCIÓN DEL JUGADOR], se produce el siguiente error o fallo de comportamiento:
[PEGAR TRAZA DE ERROR O INDICAR A CLAUDE QUE LA CAPTURE DIRECTAMENTE DESDE STUDIO MCP / @chrrxs/robloxstudio-mcp]

PROTOCOLO DE RESOLUCIÓN:
1. Usa las herramientas del servidor MCP de Roblox Studio (get_studio_state, search_game_tree, inspect_instance) y del servidor de debugging @chrrxs/robloxstudio-mcp para inspeccionar el estado real de las instancias implicadas en el DataModel y las trazas de LogService.
2. Lee los archivos .luau correspondientes en src/.
3. Determina la causa raíz exacta (ej. condición de carrera al cargar el Character, instancia fuera de radio por StreamingEnabled, propiedad física modificada fuera de PreSimulation, o clave nil en tabla).
4. Aplica la corrección en el archivo .luau local de src/ respetando --!strict.
5. Ejecuta luau-lsp analyze y selene src/ en terminal y lanza una sesión playtest vía MCP para comprobar en los logs de salida que el error ha quedado resuelto.
```

## 4. Prompt para Refactorizar y Optimizar Rendimiento Móvil (Claude Code)
```text
ROL Y CONTEXTO:
Actúa como Ingeniero de Rendimiento y Optimización de Roblox. Lee obligatoriamente /docs/vault/01_API_Roblox_y_Servicios.md y /docs/vault/06_Optimizacion_Movil_Streaming.md.

TAREA:
Refactoriza el módulo src/[RUTA_DEL_ARCHIVO].luau para eliminar cuellos de botella de CPU, fugas de memoria y tráfico de red innecesario pensando en dispositivos móviles de gama baja.

REGLAS DE REFACTORIZACIÓN:
1. Reemplaza cualquier uso heredado de Stepped o RenderStepped por los eventos exactos del Task Scheduler: RunService.PreSimulation (antes de la física / Motor6D.Transform), RunService.PostSimulation (lectura de posiciones post-física) o RunService.PreRender (cámara en cliente).
2. Elimina la creación de objetos dentro de bucles por fotograma: extrae RaycastParams.new() y OverlapParams.new() fuera del callback de RunService para reutilizar una única instancia en memoria.
3. Si el módulo gestiona NPCs, sustituye objetos Humanoid innecesarios por AnimationController reproducidos en el cliente e implementa agrupación (object pooling) en lugar de usar Instance:Destroy() y Instance:Clone() masivamente.
4. Garantiza que todas las conexiones RBXScriptConnection creadas dinámicamente se almacenen en un contenedor Trove y se limpien al destruir la entidad.

CRITERIOS DE ACEPTACIÓN:
Edita el archivo en src/, pasa luau-lsp analyze, selene y stylua con 0 advertencias, y resume las mejoras de complejidad temporal y espacial logradas.
```

## 5. Prompt para Generar Interfaces de Usuario Responsive (Claude Code)
```text
ROL Y CONTEXTO:
Actúa como Ingeniero de UI/UX en Roblox. Lee obligatoriamente /docs/vault/07_UI_UX_Responsive_Consola.md.

TAREA:
Programa el módulo constructor y controlador de interfaz src/client/ui/[NombrePantalla].luau para mostrar [DESCRIBIR ELEMENTOS DE LA PANTALLA].

RESTRICCIONES TÉCNICAS DE UI:
- Usa exclusivamente UDim2.fromScale(x, y) para posicionar y dimensionar los paneles principales, añadiendo un UIAspectRatioConstraint al contenedor central y UIPadding en escala.
- En todos los botones (ImageButton / TextButton), añade UISizeConstraint con MinSize = Vector2.new(48, 48) para cumplir el estándar táctil en móviles.
- Conecta la acción de los botones al evento Activated (nunca MouseButton1Click) para dar soporte simultáneo a clic de ratón, pantalla táctil y botón A de mando de consola.
- Configura Selectable = true y asigna GuiService.SelectedObject al abrir el menú si UserInputService.GamepadEnabled está activo.
- Añade transiciones fluidas de apertura y pulsación mediante TweenService.
```

## 6. Prompt para Especificación de Modelos 3D y Láminas Ortogonales (Google Gemini)
```text
ROL Y CONTEXTO:
Actúa como Director Técnico de Arte 3D para Roblox (especificaciones 2026) siguiendo las reglas de nuestro documento docs/art_bible.md y /GEMINI.md.

TAREA:
Prepara la especificación completa y el prompt de generación visual para el siguiente activo 3D: [DESCRIPCIÓN DEL MODELO].
Categoría en Roblox: [MeshPart de Entorno / Vehículo Articulado / Accesorio Rígido UGC].

ENTREGA OBLIGATORIA EN 3 BLOQUES:
1. PROMPT PARA GENERADOR IMAGEN-A-3D (Rodin Hyper3D / Nilo / Meshy / Tripo3D):
   - Redacta el prompt en inglés para generar la lámina de referencia 2D (vistas ortogonales frontal, lateral y 3/4) sobre fondo blanco puro (#FFFFFF), sin suelo ni sombras proyectadas, iluminación de estudio plana uniforme (flat diffuse lighting sin brillos quemados), geometría estilizada de bordes gruesos (chunky stylized low-poly), superficie estanca (watertight) con volumen real y cero texto, letras o números en la superficie para superar el filtro de moderación de Roblox.
2. FICHA DE RESTRICCIONES TÉCNICAS PARA RODIN HYPER3D / BLENDER Y ROBLOX STUDIO:
   - Presupuesto de triángulos objetivo: (Entre 500 y 5.000 triángulos según relevancia; límite duro de Roblox = 20.000 por MeshPart).
   - Dimensiones exactas en Studs y Metros: (1 stud = 0.28 m; avatar R15 = 5.2 studs).
   - Separación de sub-mallas y Pivotes: Indica qué piezas se fusionan (Merge Meshes) y cuáles quedan separadas para animarse con Motor6D o Constraints.
   - Textura y Normal Map: Horneado a 1024x1024 px, SurfaceAppearance único y normales OpenGL (+Y).
   - Física en Studio (TriangleMeshPart): CollisionFidelity (Box, Hull o Tunable) y estado de Anchored, CanCollide, CanTouch y CanQuery.
3. ENTRADA JSON PARA docs/specs/MANIFEST_Assets.json:
   - Escribe el objeto JSON estructurado con estos metadatos para que Claude Code lo lea y configure el modelo en Roblox Studio vía MCP.
```

## 7. Prompt para Generar Texturas PBR y Materiales (Google Gemini)
```text
ROL Y CONTEXTO:
Actúa como Artista Técnico de Materiales PBR para Roblox Studio siguiendo nuestra Biblia de Arte (docs/art_bible.md).

TAREA:
Diseña la especificación y los prompts para una textura repetible sin costuras (seamless tileable PBR texture) de: [DESCRIPCIÓN DE LA SUPERFICIE].

RESTRICCIONES TÉCNICAS DE TEXTURAS EN ROBLOX:
- Vista cenital ortogonal perfecta (90-degree top-down orthographic), relación 1:1 cuadrada a 1024x1024 píxeles.
- Textura base de color (Albedo) libre de sombras direccionales (delighted), mapa de normales en formato OpenGL (+Y) y mapas de Roughness y Metalness en escala de grises.
- Prohibido incluir letras, números o logotipos para evitar rechazos de moderación.

SALIDA REQUERIDA:
1. Prompt exacto en inglés para el Material Generator nativo de Roblox Studio o generador PBR externo.
2. Configuración recomendada en Studio para la instancia MaterialVariant: StudsPerTile, MaterialPattern (Regular vs Organic) y CustomPhysicalProperties (Density, Friction, Elasticity).
```

## 8. Prompt para Planificar Niveles y Greyboxing (Google Gemini -> Claude Code)
```text
ROL Y CONTEXTO:
Actúa como Diseñador Principal de Niveles (Level Designer) en Roblox.

TAREA:
Diseña la distribución espacial, las métricas de recorrido y la tabla de greyboxing para el mapa/zona: [NOMBRE DE LA ZONA].
Propósito jugable: [DESCRIBIR QUÉ OCURRE EN ESTA ZONA Y CUÁNTOS JUGADORES ALBERGA].

PARÁMETROS DE ESCALA Y RENDIMIENTO EN ROBLOX:
- Conversión y velocidad: 1 stud = 0.28 m; WalkSpeed base = 16 studs/s (80 studs = 5 segundos).
- En la zona de aparición (SpawnLocation), la orientación inicial debe apuntar directamente al objetivo principal situado a menos de 64 studs (4 segundos a pie).
- Diseña bloqueadores de línea de visión para optimizar el renderizado móvil y especifica los valores recomendados para Workspace.StreamingMinRadius y StreamingTargetRadius.

SALIDA REQUERIDA (PARA TRASPASO A CLAUDE CODE + STUDIO MCP):
1. Tabla de volúmenes de Greyboxing con columnas: NombreInstancia, FormaPrimitiva, Posicion_Vector3_Studs, Tamano_Vector3_Studs, ColorReferencia, CollisionFidelity y CollectionService_Tag.
2. Instrucción directa lista para copiar en Claude Code para que construya automáticamente toda esta maqueta de bloques en el Workspace de Roblox Studio mediante el servidor MCP.
```