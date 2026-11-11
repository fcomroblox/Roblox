# Auditoría de Enlaces Preexistentes Huérfanos: Serie 30_XX del Vault Original

## Resumen del Hallazgo
Durante la auditoría matemática de enlaces del vault fusionado, se detectaron 16 enlaces internos en formato wikilink (`[[ ... ]]`) dentro de las notas de la serie `30_XX` y `0_03` que **nunca existieron físicamente** en el repositorio original `./Roblox`. 

Corresponden a notas planificadas o esquematizadas por el autor original pero nunca creadas en disco.

## Lista de Enlaces Huérfanos Preexistentes (Originales)

| Enlace Huérfano | Notas que lo referencian | Causa / Diagnóstico |
|---|---|---|
| `[[MOC_Luau_Lenguaje_y_Tipado]]` | Referenciado en las 9 notas `30_XX` y en `0_03` | MOC planificado de Luau nunca redactado en `./Roblox` |
| `[[30_03_Refinamiento_de_Tipos_y_Typeof]]` | `30_01`, `30_04` | El archivo real fue nombrado `0_03_Refinamiento_de_Tipos_y_Typeof.md`. Se añadió alias en frontmatter para resolverlo en Obsidian |
| `[[31_04_RunService_Fases_del_Frame_y_Contexto]]` | `30_07` | Nota planificada no escrita |
| `[[32_01_Single_Script_Architecture_y_Orden_de_Carga]]` | `30_06` | Nota planificada no escrita |
| `[[32_02_Ciclo_de_Vida_Init_y_Start]]` | `30_06` | Nota planificada no escrita |
| `[[32_04_Patron_Trove_Limpieza_Determinista]]` | `30_05`, `30_07` | Nota planificada no escrita |
| `[[32_05_Patron_Signal_Eventos_Tipados]]` | `30_02` | Nota planificada no escrita |
| `[[32_07_Maquinas_de_Estado_Finito_FSM]]` | `30_04` | Nota planificada no escrita |
| `[[32_09_Modulos_de_Configuracion_y_Catalogos_Inmutables]]` | `30_09` | Nota planificada no escrita |
| `[[32_10_Object_Pooling_Instancias_y_Tablas]]` | `30_02`, `30_08` | Nota planificada no escrita |
| `[[33_02_Pipeline_de_Validacion_de_5_Capas]]` | `0_03` | Nota planificada no escrita |
| `[[33_03_Saneamiento_de_Tipos_NaN_Inf_y_UTF8]]` | `30_10` | Nota planificada no escrita |
| `[[33_06_Empaquetado_Binario_de_Paquetes_con_Buffer]]` | `30_10` | Nota planificada no escrita |
| `[[38_07_NPCs_Sin_Humanoid_y_Workspace_BulkMoveTo]]` | `30_09` | Nota planificada no escrita |
| `[[39_04_Luau_LSP_Sourcemap_y_Analisis_CLI]]` | `30_01` | Nota planificada no escrita |
| `[[40_02_Fugas_de_Memoria_en_Tablas_Indexadas_por_Player]]` | `30_08` | Nota planificada no escrita |

## Acción Tomada
1. Se preservaron intactos los textos de las notas originales (cumpliendo la regla no destructiva).
2. Para `30_03_Refinamiento_de_Tipos_y_Typeof`, se agregó `aliases: [30_03_Refinamiento_de_Tipos_y_Typeof]` al frontmatter de `0_03_Refinamiento_de_Tipos_y_Typeof.md` para que Obsidian lo resuelva automáticamente.
3. Los temas de las notas no escritas (Signal, Trove, RunService, FSM, Object Pooling, Buffers, etc.) se encuentran cubiertos exhaustivamente en `encyclopedia/` y pueden consultarse allí mediante `_tools/vault.py`.
