# Biblia de Arte Técnico (`art_bible.md`)

## 1. Dirección Estética y Estilo Geométrico
- **Estilo Visual:** *Stylized Low-Poly Hand-Painted* con siluetas limpias y biseles gruesos (*chunky beveled edges*). Se prohíbe mezclar activos fotorrealistas sucios con activos cartoon.
- **Regla de Prompts para Generadores 3D (Rodin Hyper3D / Nilo / Meshy / Tripo3D):**
  Todos los prompts de referencia 2D generados por Gemini deben incluir el sufijo estándar:
  `"stylized low-poly 3D game asset, chunky proportions, clean beveled edges, flat diffuse studio lighting, delighted albedo texture, pure solid white background #FFFFFF, watertight geometry, no text, no letters, no numbers, no logos"`.

## 2. Paleta Cromática Cerrada (Hexadecimal)
Todos los materiales, luces e interfaces (`ScreenGui`) deben utilizar exclusivamente variaciones de esta paleta:
- **Color Primario (Estructura / Base):** Azul Abisal Profundo `#1B263B`
- **Color Secundario (Paneles / Suelos):** Gris Acero Industrial `#415A77`
- **Acento Principal (Objetivos / Botones CTA / Guías):** Cian Bioluminiscente `#00F5D4`
- **Acento de Recompensa / Éxito / Monedas:** Ámbar Dorado `#FFB703`
- **Acento de Peligro / Alerta / Salud:** Coral Intenso `#FB5607`
- **Blanco / Texto Principal:** Blanco Hielo `#F8F9FA` (con `UIStroke` oscuro `#0D1B2A`)

## 3. Reglas de Materiales PBR y Texturas
- **Rugosidad (`Roughness`):** Mantener entre $0.65\text{ y }0.85$ en superficies generales para evitar el aspecto plástico brillante típico de modelos IA sin calibrar. Metales pulidos entre $0.25\text{ y }0.40$.
- **Resolución de Horneado:** Todas las texturas de `SurfaceAppearance` y `MaterialVariant` se exportan a **$1024 \times 1024\text{ px}$ (1K)** en formato `.png`, con mapas de normales en convención **OpenGL ($+Y$)** y canal Albedo totalmente libre de sombras proyectadas (*delighted*).
- **Seguridad OCR:** Ninguna textura 2D o 3D puede contener letras, números ni pseudotexto generado por IA. Todo cartel dentro del mundo se rotula en Studio mediante `SurfaceGui` + `TextLabel`.

## 4. Configuración Exacta del Servicio `Lighting`
- **`Lighting.Technology`:** `Enum.Technology.ShadowMap` (equilibrio óptimo entre calidad de sombras y $60\text{ FPS}$ en móviles).
- **Propiedades Base de `Lighting`:**
  - `Ambient = Color3.fromHex("#1B263B")`
  - `OutdoorAmbient = Color3.fromHex("#415A77")`
  - `Brightness = 2.2`
  - `EnvironmentDiffuseScale = 0.6`
  - `EnvironmentSpecularScale = 0.5`
- **Instancias Hijas Obligatorias en `Lighting`:**
  - `Atmosphere`: `Density = 0.32`, `Offset = 0.15`, `Color = Color3.fromHex("#1B263B")`, `Decay = Color3.fromHex("#0D1B2A")`, `Glare = 0`, `Haze = 1.5`.
  - `ColorCorrectionEffect`: `Brightness = 0.03`, `Contrast = 0.12`, `Saturation = 0.15`.

## 5. Estándares de Interfaz de Usuario (UI) y Tipografía
- **Máximo 2 Fuentes Tipográficas (`Enum.Font`) en Todo el Juego:**
  1. **Títulos, Cabeceras y Contadores Numéricos:** `Enum.Font.FredokaOne` (con `UIStroke` de grosor $2\text{ px}$).
  2. **Descripciones, Estadísticas y Cuerpo de Texto:** `Enum.Font.GothamMedium` (o `BuilderSansMedium`).
- **Esquinas y Bordes:** `UICorner` con `CornerRadius = UDim.new(0.15, 0)` en botones y `UDim.new(0.04, 0)` en ventanas modales.