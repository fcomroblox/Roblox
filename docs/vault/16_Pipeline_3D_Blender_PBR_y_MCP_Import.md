---
tags: [pipeline-3d-blender-pbr-y-mcp-import]
ultima_verificacion: pendiente
---
# 16. Ingeniería de Assets 3D: Automatización en Blender, `SurfaceAppearance` y Subida vía Studio MCP

Relacionado con: [[00_Mapa_del_Vault]] | [[art_bible]] | [[06_Optimizacion_Movil_Streaming]] | [[08_Normas_Moderacion_Politicas]] | [[MANIFEST_Assets.json]]

---

## 1. Reglas de Geometría para Generadores IA (Rodin Hyper3D, Nilo, Meshy, Tripo3D)
1. **Presupuesto de Triángulos por Categoría:**
   - **Props pequeños / recolectables / decoración repetida:** $150\text{ a }800\text{ triángulos}$.
   - **Mobiliario / estaciones interactivas (`DepositTerminal`, `UpgradeStation`):** $1.000\text{ a }2.800\text{ triángulos}$.
   - **Vehículos protagonistas / Jefes articulados:** $3.000\text{ a }6.000\text{ triángulos}$ (divididos solo en las sub-mallas que requieran rotación independiente con `Motor6D` o `HingeConstraint`).
   - **Límite duro absoluto de Roblox Studio:** $20.000\text{ triángulos}$ por `MeshPart` ($4.000$ en accesorios UGC).
2. **Fusión de Sub-Mallas Estáticas (`Merge Meshes`):**
   - Si un generador IA exporta un edificio o estación con 40 piezas estáticas sueltas, deben unirse en Blender (`Ctrl + J`) en **una única malla** con un solo atlas UV antes de exportar. Importar 40 piezas separadas multiplica por 40 las llamadas de dibujado (*Draw Calls*) y arruina los FPS en móvil.

---

## 2. Script en Python para Limpieza y Calibración Automática en Blender 4.2+
Claude Code puede generar y ejecutar este script en modo headless (`blender -b archivo.glb -P clean_for_roblox.py`) o tú puedes ejecutarlo en la pestaña *Scripting* de Blender para dejar cualquier modelo IA listo para Roblox en 1 segundo (une mallas, suelda vértices duplicados, aplica el modificador *Decimate* al presupuesto exacto de triángulos, coloca el pivote en el centro de la base inferior y aplica transformaciones a escala métrica):

~~~python
# clean_for_roblox.py - Automatización de limpieza 3D para Roblox Studio
import bpy
import bmesh
from mathutils import Vector

TARGET_MAX_TRIANGLES = 2000  # Ajustar según MANIFEST_Assets.json

def clean_and_prepare_mesh_for_roblox(target_tris: int):
    # 1. Seleccionar todos los objetos de tipo MESH y unirlos en uno solo
    bpy.ops.object.select_all(action='DESELECT')
    mesh_objects = [obj for obj in bpy.context.scene.objects if obj.type == 'MESH']
    if not mesh_objects:
        print("[RobloxPipeline] No se encontraron mallas en la escena.")
        return

    for obj in mesh_objects:
        obj.select_set(True)
    bpy.context.view_layer.objects.active = mesh_objects[0]

    if len(mesh_objects) > 1:
        bpy.ops.object.join()

    active_obj = bpy.context.active_object

    # 2. Aplicar todas las rotaciones y escalas previas
    bpy.ops.object.transform_apply(location=True, rotation=True, scale=True)

    # 3. Soldar vértices por distancia y recalcular normales hacia afuera
    bpy.ops.object.mode_set(mode='EDIT')
    bpy.ops.mesh.select_all(action='SELECT')
    bpy.ops.mesh.remove_doubles(threshold=0.0005)
    bpy.ops.mesh.normals_make_consistent(inside=False)
    bpy.ops.mesh.quads_convert_to_tris()
    bpy.ops.object.mode_set(mode='OBJECT')

    # 4. Evaluar conteo de triángulos y aplicar Decimate si supera el presupuesto
    current_tris = len(active_obj.data.polygons)
    if current_tris > target_tris:
        ratio = target_tris / float(current_tris)
        decimate = active_obj.modifiers.new(name="RobloxDecimate", type='DECIMATE')
        decimate.decimate_type = 'COLLAPSE'
        decimate.ratio = max(0.05, min(1.0, ratio))
        decimate.use_collapse_triangulate = True
        bpy.ops.object.modifier_apply(modifier=decimate.name)
        print(f"[RobloxPipeline] Diezmado de {current_tris} a {len(active_obj.data.polygons)} triángulos.")

    # 5. Colocar el pivote (Origin) exactamente en el centro inferior (BottomCenter)
    bbox_corners = [active_obj.matrix_world @ Vector(corner) for corner in active_obj.bound_box]
    min_z = min(corner.z for corner in bbox_corners)
    center_x = sum(corner.x for corner in bbox_corners) / 8.0
    center_y = sum(corner.y for corner in bbox_corners) / 8.0

    bottom_center = Vector((center_x, center_y, min_z))
    active_obj.data.transform(bpy.mathutils.Matrix.Translation(-bottom_center))
    active_obj.matrix_world.translation = Vector((0.0, 0.0, 0.0))

    print(f"[RobloxPipeline] Malla '{active_obj.name}' lista con pivote en base inferior.")

if __name__ == "__main__":
    clean_and_prepare_mesh_for_roblox(TARGET_MAX_TRIANGLES)
~~~

---

## 3. Configuración de `SurfaceAppearance` y `MaterialVariant` en Studio
- **`SurfaceAppearance` (Texturas UV Únicas por Modelo):**
  - Hijo directo del `MeshPart`.
  - `AlphaMode = Enum.AlphaMode.Overlay` (opaco estándar; usa `Transparency` solo si la textura tiene cristal o rejillas reales, ya que el blending transparente es más costoso en GPU móvil).
  - Asegúrate de que el mapa Normal exportado por Rodin Hyper3D / Meshy esté en formato **OpenGL ($+Y$)**; si los relieves se ven invertidos hacia adentro, era un mapa DirectX ($-Y$) y debes invertir el canal Verde (Green) de la imagen.
- **Protocolo de Importación con Claude Code vía Studio MCP:**
  1. Coloca el archivo `.fbx` o `.glb` limpio en la carpeta local aprobada (`Approved Upload Folder`).
  2. Ordena a Claude Code leer `docs/specs/MANIFEST_Assets.json` y ejecutar la habilidad nativa **`rbx-import-asset`** e **`insert_asset`**.
  3. Claude verificará automáticamente las propiedades `Anchored`, `CanCollide`, `CanTouch`, `CanQuery`, `CollisionFidelity`, `ModelStreamingMode` y asignará los tags de `CollectionService`.

---
**Ver también (enciclopedia):** [[Blender Workflow for Roblox Creators|3D Modeling]], [[Art Pipeline - Concept to In-Game Asset|Art Pipeline]], [[Low-Poly Creature From a Flat Reference Sheet|Procedural Blender]]
