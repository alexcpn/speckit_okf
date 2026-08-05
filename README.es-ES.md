

# speckit-okf — Generador de Paquetes de Conocimiento OKF para Spec Kit

[![CodeQL](https://github.com/alexcpn/speckit_ofk/actions/workflows/codeql.yml/badge.svg)](https://github.com/alexcpn/speckit_ofk/actions/workflows/codeql.yml)
[![ShellCheck](https://github.com/alexcpn/speckit_ofk/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/alexcpn/speckit_ofk/actions/workflows/shellcheck.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Una extensión de [Spec Kit](https://github.com/github/spec-kit) que convierte a tu agente de código IA en un **agente de enriquecimiento OKF**: analiza un repositorio de código fuente y genera un paquete de conocimiento conforme al [Formato Abierto de Conocimiento (OKF v0.1)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md) — un directorio de conceptos de markdown interconectados con frontmatter YAML que describe tus servicios, módulos, APIs, modelos de datos y operaciones.

Dado que los paquetes OKF son markdown plano en git, el paquete generado es legible por humanos, se puede comparar en diffs de PRs y es consumible por otros agentes sin necesidad de herramientas personalizadas.

## Comandos

| Comando | Qué hace |
| ------- | ------------ |
| `/speckit.okf.generate` | Inicializa un paquete completo desde el repositorio: exploración de inventario, minería del historial de git (churn + justificación), plan de conceptos, documentos de conceptos, archivos `index.md`, `log.md`, validación. |
| `/speckit.okf.update` | Actualización incremental: compara el historial de git desde el último commit registrado, actualiza de forma precisa solo los conceptos obsoletos, deprecita los huérfanos, crea conceptos para código nuevo y preserva la edición humana. |
| `/speckit.okf.clarify` | Resuelve las `open_questions` que generan/actualizan aparcadas (en lugar de adivinar) preguntando al usuario, y luego incorpora las respuestas en los conceptos como conocimiento citado y protegido por curación. |
| `/speckit.okf.validate` | Ejecuta el verificador de conformidad OKF §9 y una verificación puntual de calidad; informa sobre ERRORs/WARNINGs con correcciones. |

## Instalación

```bash
# Opción 1: instalar desde un archivo publicado (no se necesita catálogo)
specify extension add okf --from https://github.com/alexcpn/speckit_ofk/archive/refs/tags/v0.3.0.zip

# Opción 2: instalar desde un clon local (modo dev)
git clone https://github.com/alexcpn/speckit_ofk.git
specify extension add --dev speckit_ofk/

# Opción 3: una vez que esté en el catálogo comunitario
specify extension add okf
```

## Uso

Desde tu agente de código (Claude Code, Copilot, etc.) en el proyecto:

```bash
# 1. Generar el paquete de conocimiento inicial
/speckit.okf.generate

# 2. Resolver lo que el agente no pudo inferir del código + historial de git
/speckit.okf.clarify

# 3. Después de hacer cambios en el código, actualizar de forma incremental
/speckit.okf.update

# 4. Validar la conformidad antes de hacer commit
/speckit.okf.validate
```

La salida se genera en `knowledge/` (configurable) como un conjunto de archivos de conceptos en markdown interconectados, listos para commit junto con tu código.

## Configuración

Copia `okf-config.template.yml` a `.specify/extensions/okf/okf-config.yml` para controlar el directorio del paquete (por defecto `knowledge/`), la base de URI de recursos, exclusiones, mapeos de tipos, diseño y granularidad (`coarse` / `medium` / `fine`). Los valores predeterminados funcionan sin configuración adicional.

## Cómo funciona

1. `scripts/bash/okf-inventory.sh` explora el repositorio de manera determinista (puntos de entrada, manifiestos de dependencias, definiciones de API, migraciones/modelos, CI/CD, documentación, ADRs) a JSON — incluyendo **señales del historial de git** (`churn` = conteo de commits por archivo para relevancia, `recent_commits`) — para que el agente planifique con hechos, no con suposiciones.
2. `scripts/bash/okf-history.sh <path>` le proporciona al agente un historial de git acotado y por concepto — commit de creación, temas recientes y commits de revert/hotfix/riesgo — para que los conceptos capturen el **"por qué"** (invariantes, advertencias) con citas de commits, no solo el "qué".
3. El prompt del comando instruye al agente para que redacte un plan de conceptos y luego escriba documentos conformes a OKF: frontmatter `type` obligatorio, `title`/`description`/`resource`/`tags`/`timestamp` recomendados, campos de extensión del productor `source_files` (mapea cada concepto de vuelta al código — lo que hace posible las actualizaciones incrementales) y `open_questions` (aparca la incertidumbre para `/speckit.okf.clarify` en lugar de adivinar), enlaces relativos al paquete, archivos de revelación progresiva `index.md` y un `log.md` con fecha ISO.
4. `scripts/python/validate_okf.py` aplica OKF §9: frontmatter parseable en todas partes, `type` no vacío, estructura de archivos reservados — y advierte sobre enlaces rotos, índices faltantes, cuerpos vacíos, cadenas que parecen secretos, `source_files` huérfanos, conceptos duplicados y `open_questions` sin resolver.

## Estructura del paquete generado (ejemplo)

```
knowledge/
├── index.md            # okf_version: "0.1" + directorio de todo
├── log.md              # historial con fecha, registra el SHA del commit fuente
├── architecture/
│   ├── index.md
│   └── overview.md     # type: Reference — concepto "comienza aquí"
├── services/…          # type: Service
├── modules/…           # type: Module
├── apis/…              # type: API Endpoint / API Resource
├── data/…              # type: Data Model / Database Table
└── operations/…        # type: Pipeline / Configuration / Playbook
```

## Notas

- El actualizador nunca elimina conceptos o texto escrito por humanos; el código eliminado genera `status: deprecated`, no una eliminación.
- El agente aparca lo que no puede verificar en `open_questions` en lugar de adivinar; `/speckit.okf.clarify` convierte esos elementos en hechos confirmados por humanos y protegidos por curación (marcados con centinelas `<!-- clarified -->` que el actualizador no sobrescribirá).
- Los secretos encontrados en configuraciones — o revelados por la minería del historial de git — se describen por su estructura, nunca por su valor; el validador marca cualquier cosa que se escape.
- El modelo de consumo permisivo de OKF (tipos desconocidos OK, enlaces rotos OK) se utiliza deliberadamente — la generación es segura para ejecutarla temprano y con frecuencia.
- Ambos scripts se verifican en cada push mediante CodeQL y ShellCheck; consulta [SECURITY.md](SECURITY.md) para informar una vulnerabilidad.

## Licencia

MIT
