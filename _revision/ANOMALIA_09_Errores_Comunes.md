# Anomalía detectada: `09_Errores_Comunes_IA_Roblox.md`

## Hallazgo
- **Archivo**: `docs/vault/09_Errores_Comunes_IA_Roblox.md`
- **Problema**: El H1 del archivo dice `# 08. Normas de Moderación, PolicyService y Políticas del Marketplace`, pero el nombre del archivo indica que debería ser la nota 09 ("Errores Comunes IA Roblox").
- **Archivo 08 real**: `08_Normas_Moderacion_Politicas.md` tiene H1 `# 08. Normas de Moderación, PolicyService y Políticas de Roblox` — contenido casi idéntico al del archivo 09 pero con título ligeramente diferente ("de Roblox" vs "del Marketplace").

## Diagnóstico probable
El archivo `09_Errores_Comunes_IA_Roblox.md` fue sobreescrito accidentalmente con una variante del contenido de la nota 08. El contenido real de "Errores Comunes de IA en Roblox" se perdió en el vault original.

## Acción tomada
- Archivo copiado tal cual a `Roblox_Vault_Definitivo/docs/vault/09_Errores_Comunes_IA_Roblox.md`.
- NO se ha modificado, corregido ni generado contenido.
- Se ha registrado este hallazgo para revisión manual.

## Referencia cruzada
- `CLAUDE.md` referencia la nota 09 como: "Depuración de bugs frecuentes, fugas de memoria y física"
- `00_Mapa_del_Vault.md` la describe como: "Solución a los 7 antipatrones clásicos de LLMs en Roblox"
- La enciclopedia LihFolk tiene contenido equivalente en: `encyclopedia/15 AI Tools/` y `encyclopedia/16 MCP and Agents/`

## Estado: `[VERIFICAR]` — Requiere restauración del contenido original de la nota 09
