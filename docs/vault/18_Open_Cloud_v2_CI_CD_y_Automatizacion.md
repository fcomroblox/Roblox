---
tags: [open-cloud-v2-ci-cd-y-automatizacion]
ultima_verificacion: pendiente
---
# 18. Roblox Open Cloud v2, `open_cloud_invoke` y Pipeline CI/CD en GitHub Actions

Relacionado con: [[00_Mapa_del_Vault]] | [[CLAUDE]] | [[05_Datos_ProfileStore_Monetizacion]] | [[08_Normas_Moderacion_Politicas]]

---

## 1. Arquitectura de Roblox Open Cloud v2 (`https://apis.roblox.com`)
Roblox Open Cloud v2 unifica el acceso REST a los recursos de tu universo bajo esquemas OpenAPI 3.2.0. Existen dos vías de ejecución en este proyecto:
1. **Desde Roblox Studio vía MCP (`open_cloud_invoke` sin API Key):**
   - Claude Code puede invocar endpoints de Open Cloud directamente usando la herramienta `open_cloud_invoke` del servidor MCP nativo de Studio.
   - Studio gestiona la autenticación internamente según los interruptores activados en `Assistant` $\rightarrow$ `...` $\rightarrow$ `Settings` $\rightarrow$ `MCP Permissions`.
   - Utiliza siempre primero las habilidades integradas (`rbx-open-cloud-usage`, `rbx-permissions`, `rbx-import-asset`) para consultar esquemas y permisos antes de lanzar peticiones REST.
2. **Desde GitHub Actions (CI/CD con `x-api-key`):**
   - Para compilar, verificar tipos y publicar automáticamente al hacer `git push`, se utiliza una clave creada en `Creator Dashboard` $\rightarrow$ `Open Cloud` $\rightarrow$ `API Keys` guardada en los secretos de GitHub (`ROBLOX_OPEN_CLOUD_API_KEY`).

---

## 2. Endpoints Clave y Límites de Frecuencia (Rate Limits)
- **Developer Products v2:**
  - Ruta: `/developer-products/v2/universes/{universeId}/developer-products`
  - Permite listar, crear y actualizar precios o iconos de productos consumibles por código.
- **Game Passes v2:**
  - Ruta: `/game-passes/v2/universes/{universeId}/game-passes`
- **Límite de Cuota Verificado:** Tanto Developer Products como Game Passes tienen un límite estricto de **$3\text{ peticiones por segundo}$** en total para todas las API Keys de un usuario o Grupo. Si un script de sincronización crea 15 productos desde un archivo JSON, debe insertar un `task.wait(0.4)` entre cada petición HTTP.
- **Mensajería Global desde la Nube (`publishMessage`):**
  - Ruta: `POST https://apis.roblox.com/cloud/v2/universes/{universeId}:publishMessage`
  - Cuerpo JSON: `{"topic": "GlobalServerEvent", "message": "{\"type\":\"DoubleCoins\"}"}`.

---

## 3. Plantilla de Integración Continua para GitHub Actions (`.github/workflows/ci.yml`)
Si deseas que GitHub verifique automáticamente que ningún commit rompe `--!strict`, `selene` o `stylua` y compile el archivo `build.rbxl`:

~~~yaml
name: Roblox Luau CI & Quality Gate

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  check-and-build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Install Rokit Toolchain
        run: |
          curl -sSf https://raw.githubusercontent.com/rojo-rbx/rokit/main/scripts/install.sh | bash
          echo "$HOME/.rokit/bin" >> $GITHUB_PATH
          $HOME/.rokit/bin/rokit install

      - name: Install Wally Packages & Generate Types
        run: |
          wally install
          rojo sourcemap default.project.json --output sourcemap.json
          wally-package-types --sourcemap sourcemap.json Packages/

      - name: Check Formatting (StyLua)
        run: stylua --check src/

      - name: Run Static Linter (Selene)
        run: selene src/

      - name: Run Strict Typecheck (Luau LSP)
        run: |
          curl -sL https://raw.githubusercontent.com/MaximumADHD/Roblox-Client-Tracker/roblox/api-docs/en-us.json -o globalTypes.d.luau || true
          luau-lsp analyze --sourcemap=sourcemap.json src/

      - name: Build Roblox Place Binary (.rbxl)
        run: rojo build default.project.json --output build.rbxl
~~~

---
**Ver también (enciclopedia):** [[Git and Source Control for Roblox|CI/CD]], [[Open Cloud APIs|Open Cloud]]
