# MongoDB CSFLE Demo con Mongoose

Demo interactivo de **Client-Side Field Level Encryption (CSFLE)** usando Mongoose y el driver oficial de MongoDB. El cifrado ocurre en el cliente: los datos sensibles nunca viajan ni se almacenan en texto plano.

## Arquitectura

```
App (Mongoose)
  └─ ClientEncryption (mongodb driver)
       ├─ KMS Provider  → clave maestra (local / AWS / Azure / GCP)
       └─ Key Vault     → DEKs cifradas almacenadas en MongoDB
```

El demo corre como un Jupyter Notebook (`.ipynb`) usando el kernel **tslab** (TypeScript sobre Node.js).

## Requisitos

| Herramienta | Versión mínima | Notas |
|---|---|---|
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | cualquiera | Para levantar MongoDB |
| [uv](https://docs.astral.sh/uv/) | 0.5+ | Gestor del entorno Python/Jupyter |
| [Node.js](https://nodejs.org) | 18+ | Runtime para tslab |
| [tslab](https://github.com/yunabe/tslab) | 1.0+ | Kernel TypeScript para Jupyter |
| [pnpm](https://pnpm.io) | 11.x | Gestor de paquetes Node |
| [VS Code](https://code.visualstudio.com) | cualquiera | Editor recomendado |
| Extensión [Jupyter](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.jupyter) | — | Para abrir el `.ipynb` |

## Instalación

### 1. Clonar el repositorio

```bash
git clone <url-del-repo>
cd mongoose-csfle-demo
```

### 2. Configurar el entorno Python con uv

El proyecto usa `uv` para gestionar Python y JupyterLab. `uv` lee `pyproject.toml` y `.python-version` automáticamente:

```bash
uv sync
```

Esto crea el virtualenv en `.venv` e instala JupyterLab con Python 3.14.

### 3. Instalar dependencias Node

```bash
pnpm install
```

### 4. Instalar el kernel tslab (solo la primera vez)

tslab registra el kernel de TypeScript en Jupyter. Debe instalarse en el entorno `uv`:

```bash
npm install -g tslab
uv run tslab install
```

### 5. Levantar MongoDB

```bash
docker compose up -d
```

Esto inicia MongoDB Enterprise en `localhost:47017` con usuario `admin` / contraseña `secret`.

### 3. Descargar `crypt_shared`

La auto-encryption requiere la librería `crypt_shared` de MongoDB. Descárgala desde [mongodb.com/try/download/enterprise](https://www.mongodb.com/try/download/enterprise) seleccionando:

- **Version**: 7.0 o superior (current recomendado) 
- **Platform**: tu sistema operativo  
- **Package**: `crypt_shared`

Extrae el archivo y toma nota de la ruta al binario:
- macOS/Linux: `lib/mongo_crypt_v1.dylib` / `lib/mongo_crypt_v1.so`
- Windows: `bin/mongo_crypt_v1.dll`

### 4. Generar la master key local

```bash
openssl rand -base64 96
```

Guarda el output: es tu `CSFLE_LOCAL_MASTER_KEY_B64` (96 bytes en base64).

### 5. Configurar variables de entorno

Copia el archivo de ejemplo y completa los valores:

```bash
cp .env.example .env
```

Edita `.env`:

```env
CSFLE_LOCAL_MASTER_KEY_B64=<resultado del openssl rand>
MONGODB_URI=mongodb://admin:secret@127.0.0.1:47017/?authSource=admin&directConnection=true
MONGO_CRYPT_SHARED_LIB_PATH=/ruta/a/mongo_crypt_v1.dylib
```

## Ejecutar en VS Code

1. Abre la carpeta del proyecto en VS Code:
   ```bash
   code .
   ```
2. Instala la extensión **Jupyter** si aun no la tienes.
3. Abre el archivo `csfle-demo.ipynb`.
4. En la esquina superior derecha del notebook, haz clic en **Select Kernel** → **Select Another Kernel...** → **Jupyter Kernel...**.
5. Antes de elegir el kernel de TypeScript, asegurate de que VS Code esté usando el entorno Python de `uv`. Si la lista muestra varios kernels de tslab, selecciona el que proviene del `.venv` del proyecto (la ruta debe contener `.venv` dentro del directorio del repo). Puedes verificar el entorno activo abriendo la paleta de comandos (`Cmd+Shift+P`) → **Python: Select Interpreter** → elige `./.venv/bin/python`.
6. Con el intérprete correcto activo, selecciona el kernel **TypeScript (tslab)**.
7. Ejecuta las celdas en orden con **Run All** o `Shift+Enter` celda por celda.

> **Por qué importa:** `tslab install` se ejecutó dentro del `.venv` de `uv` (via `uv run tslab install`). Si VS Code resuelve el servidor Jupyter desde otro entorno Python, no encontrará ese kernel.

### Lanzar JupyterLab en el navegador (alternativa)

```bash
uv run jupyter lab
```

## Estructura del proyecto

```
.
├── csfle-demo.ipynb          # Notebook principal
├── docker-compose.yml        # MongoDB Enterprise en Docker
├── pyproject.toml            # Dependencias Python (JupyterLab via uv)
├── uv.lock                   # Lock file de uv
├── .python-version           # Version de Python requerida (3.14)
├── .env.example              # Plantilla de variables de entorno
├── .env                      # Variables locales (no se commitea)
├── package.json              # Dependencias Node (mongoose, mongodb, etc.)
└── tsconfig.json             # Configuracion TypeScript
```

## Conceptos demostrados

| Celda | Concepto |
|---|---|
| `kms-setup` | Inicializar `ClientEncryption` con KMS provider local |
| `dek` | Crear o recuperar una Data Encryption Key (DEK) |
| `mongoose-setup` | Schema de Mongoose con campos `encrypt` + `autoEncryption` |
| `encrypt-insert` | Insertar documento con cifrado automatico |
| `verify-raw` | Verificar que los campos son `Binary subtype=6` en la base de datos |
| `search-deterministic` | Buscar por campo cifrado determinístico |

## Proximos pasos

- **Queryable Encryption (FLE2)**: MongoDB 7.0+ permite queries de rango sobre campos cifrados.
- **KMS externo**: reemplaza `local` por `aws`, `azure` o `gcp` para produccion.
- **Rotacion de claves**: usa `encryption.rewrapManyDataKey()` para re-cifrar DEKs.

## Referencias

- [MongoDB CSFLE Docs](https://www.mongodb.com/docs/manual/core/csfle/)
- [Mongoose Encryption](https://mongoosejs.com/docs/field-level-encryption.html)
- [mongodb-client-encryption](https://www.npmjs.com/package/mongodb-client-encryption)
