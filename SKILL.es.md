name: hyper-maker
description: Herramienta de creación impulsada por IA para tareas de datos - automatice generación, transformación y análisis de datos
version: "1.0.0"
author: "Hyper Maker Team"
tags: ["data", "ai", "automation"]
maintainer: "SMOUJBOT"
category: "data-processing"
```

# Hyper Maker

Herramienta CLI impulsada por IA para tareas automatizadas de datos, incluyendo generación, transformación, limpieza y análisis.

## Propósito

Hyper Maker automatiza flujos de trabajo complejos de datos mediante IA. Casos de uso reales:

- Generar conjuntos de datos sintéticos que coincidan con esquemas y distribuciones específicas
- Transformar datos entre formatos (CSV → JSON, XML → Parquet, etc.) con mapeo inteligente
- Limpiar datos desordenados: detectar valores atípicos, manejar valores faltantes, normalizar formatos
- Analizar conjuntos de datos: generar estadísticas resumidas, detectar patrones, crear informes de calidad de datos
- Aumentar conjuntos de datos existentes con muestras sintéticas generadas por IA
- Validar datos contra reglas y restricciones personalizadas
- Crear canalizaciones de datos a partir de descripciones en lenguaje natural

## Alcance

Comandos que ejecuta Hyper Maker:

- `hyper-maker generate <schema> --format <csv|json|parquet> --size <N> --seed <int>`
- `hyper-maker transform <input> --to <format> --map <field_mappings> --filter <conditions>`
- `hyper-maker clean <input> --strategy <auto|conservative|aggressive> --output <clean.csv>`
- `hyper-maker analyze <input> --profile --report <report.json> --correlations`
- `hyper-maker augment <input> --samples <N> --preserve-distribution --noise <float>`
- `hyper-maker validate <input> --rules <rules.yaml> --fail-on <errors|warnings>`
- `hyper-maker pipeline <pipeline.yml> --dry-run --parallel --cache-dir <path>`

## Proceso de Trabajo

1. **Validación de Entrada**
   - Verificar existencia y formato de archivo
   - Comprobar espacio en disco (necesita 2× tamaño de entrada para archivos temporales)
   - Validar variables de entorno: `HYPER_MAKER_API_KEY`, `HYPER_MAKER_MODEL` (predeterminado: "claude-3.5-sonnet")

2. **Procesamiento con IA**
   - Construir contexto a partir de datos de entrada (primeras 100 filas de muestra)
   - Construir prompts específicos del dominio según el tipo de tarea
   - Ejecutar inferencia de IA con temperature=0.2 para salidas deterministas
   - Analizar respuesta de IA en operaciones estructuradas

3. **Ejecución de Transformación**
   - Aplicar transformaciones usando pandas/polars (detección automática según tamaño de archivo: <100MB → pandas, >100MB → polars)
   - Procesamiento por streaming para archivos >1GB (tamaño de chunk: 100,000 filas)
   - Registro de progreso en `~/.hyper-maker/logs/`

4. **Validación y Calidad**
   - Comprobaciones de calidad de datos post-transformación
   - Validación de esquema si se proporciona esquema destino
   - Calcular métricas de calidad: puntuaciones de completitud, unicidad, validez

5. **Salida y Metadatos**
   - Escribir salida en formato solicitado
   - Generar `output.metadata.json` con resumen de operación, conteos de filas, puntuaciones de calidad
   - Registrar operación en `~/.hyper-maker/history.jsonl`

## Reglas de Oro

1. **Nunca sobrescribir archivos fuente** - siempre crear nuevas salidas a menos que `--inplace` esté explícitamente configurado (predeterminado: false)
2. **Mantener siempre los tipos de datos** - conservar tipos numéricos, fechas, categóricos; nunca convertir todo a strings
3. **Sembrar todo** - operaciones aleatorias deben usar `--seed` para reproducibilidad; semilla predeterminada de variable `HYPER_MAKER_SEED` o basada en timestamp
4. **Respetar límites de memoria** - para archivos >500MB, forzar modo streaming; fallar gracefulmente si memoria >80% de RAM del sistema
5. **Registrar todos los prompts de IA** - almacenar prompts y respuestas en `~/.hyper-maker/audit/` para depuración
6. **Validar antes de escribir** - comprobaciones de esquema y calidad deben pasar antes de que el archivo de salida se mueva a ubicación final
7. **Cachear transformaciones** - reutilizar transformaciones idénticas dentro de 24h si cache habilitado (`--cache`)

## Ejemplos

### Ejemplo 1: Generar datos sintéticos de clientes
```bash
hyper-maker generate '{"name": "string", "age": "int[18,80]", "email": "email", "signup_date": "date[2020-01-01,2024-12-31]"}' \
  --format csv \
  --size 10000 \
  --seed 42 \
  --output customers.csv
```

**Salida**: `customers.csv` con 10,000 filas, `customers.metadata.json` con parámetros de generación

### Ejemplo 2: Limpiar datos de productos desordenados
```bash
hyper-maker clean products.csv \
  --strategy aggressive \
  --remove-duplicates \
  --normalize-text \
  --output products_clean.csv
```

**Qué sucede**:
- IA analiza primeras 100 filas para detectar inconsistencias de formato
- Elimina IDs de Producto duplicados
- Estandariza: "N/A", "null", "" → nulos apropiados
- Recorta espacios en blanco, corrige mayúsculas/minúsculas en campos categóricos

### Ejemplo 3: Transformar con mapeo de campos con IA
```bash
hyper-maker transform legacy_data.xml \
  --to json \
  --map "customer_id=id, full_name=name, phone_number=contact.phone" \
  --filter "status == 'active' AND age >= 18" \
  --output customers.json
```

**Mapeo de campos**: rutas de destino anidadas admitidas con notación de punto

### Ejemplo 4: Analizar conjunto de datos con correlaciones
```bash
hyper-maker analyze sales.csv \
  --profile \
  --correlations \
  --report analysis_report.md
```

**Informe incluye**:
- Resumen de esquema
- Matriz de valores faltantes
- Datos de mapa de calor de correlaciones
- Top 10 anomalías detectadas
- Puntuación de calidad de datos (0-100)

### Ejemplo 5: Aumentar conjunto de datos desbalanceado
```bash
hyper-maker augment fraud_transactions.csv \
  --samples 5000 \
  --preserve-distribution \
  --noise 0.05 \
  --output fraud_augmented.csv
```

IA genera muestras sintéticas de fraude que mantienen propiedades estadísticas mientras añade variación.

## Dependencias y Requisitos

**Requeridos**:
- Python 3.9+
- `pandas>=2.0.0` (predeterminado) o `polars>=0.19.0` (archivos grandes)
- `numpy>=1.24.0`
- `scikit-learn>=1.3.0` (para aumento)
- `openai>=1.0.0` o `anthropic>=0.8.0` (proveedor de IA)
- 4GB RAM mínimo, 16GB recomendado para conjuntos >1GB

**Opcionales**:
- `pyarrow>=12.0.0` (soporte Parquet/Feather)
- `orjson>=3.9.0` (JSON rápido)
- `pandera>=0.16.0` (validación de esquema)

**Variables de Entorno**:
- `HYPER_MAKER_API_KEY` - clave API del proveedor de IA
- `HYPER_MAKER_MODEL` - nombre del modelo (predeterminado: "claude-3-5-sonnet-20241022")
- `HYPER_MAKER_PROVIDER` - "openai" o "anthropic" (predeterminado: "anthropic")
- `HYPER_MAKER_CACHE_DIR` - directorio de caché (predeterminado: "~/.hyper-maker/cache")
- `HYPER_MAKER_MAX_MEMORY_PERCENT` - límite de memoria (predeterminado: 80)

## Pasos de Verificación

Después de cualquier operación, verificar:
1. Archivo de salida existe y tamaño > 0
2. Conteo de filas coincide con esperado (revisar `output.metadata.json`)
3. Puntuación de calidad > umbral (predeterminado: 70 para limpiar, 50 para generar)
4. Sin errores en `~/.hyper-maker/logs/operation-<timestamp>.log`
5. Cumplimiento de esquema si validación fue parte del pipeline

Comando de verificación rápida:
```bash
hyper-maker verify <output_file> --checks existencia,filas,calidad,esquema
```

## Solución de Problemas

**"API key not set"**
- Configurar `HYPER_MAKER_API_KEY` en entorno
- O usar `hyper-maker auth --key <your-key>`

**"Memory limit exceeded"**
- Usar `--chunk-size <N>` para procesar en chunks
- Cambiar a polars: `hyper-maker use-backend polars`
- Reducir `--max-memory-percent`

**"AI prompt timeout"**
- Aumentar timeout: `hyper-maker config set ai_timeout 120`
- Reducir tamaño de muestra: `--sample-rows 50`
- Revisar `~/.hyper-maker/audit/latest_prompt.txt` para complejidad del prompt

**"Schema validation failed"**
- Inspeccionar `output.validation_errors.json`
- Revisar sintaxis de reglas en `rules.yaml`
- Configurar `--strict false` para permitir solo advertencias

**"Slow performance on large files"**
- Habilitar procesamiento paralelo: `--parallel-workers 4`
- Usar streaming: `--streaming true`
- Considerar convertir a Parquet primero

**"AI hallucination in field mapping"**
- Usar mapeos explícitos: `--map "source=target"` (sin inferencia de IA)
- Configurar `--ai-reasoning false` para deshabilitar coincidencia de campos con IA

## Comandos de Rollback

Deshacer operaciones:

```bash
# Revertir a versión anterior desde historial
hyper-maker rollback <output_file> --to <timestamp|previous|commit-hash>

# Listar puntos de rollback disponibles
hyper-maker history <file> --show-versions

# Restaurar original desde backup (si se usó --backup-source)
hyper-maker restore <output_file> --source <original.csv>

# Rollback completo de pipeline (revertir todas las salidas de ejecución de pipeline)
hyper-maker pipeline-rollback <pipeline_run_id>

# Limpiar caché (no hace rollback, pero limpia estado)
hyper-maker cache clear --older-than 7d
```

**Nota**: Historial almacenado en `~/.hyper-maker/operations.db` (SQLite). Archivos de respaldo con flag `--backup-source` para habilitar restauración completa.

## Uso Avanzado

**Prompts personalizados de IA**:
```bash
hyper-maker transform data.csv \
  --custom-prompt "Convertir todas las fechas a formato ISO, eliminar filas donde revenue es negativo, y marcar cualquier fila donde email es inválido" \
  --output cleaned.csv
```

**Definición de pipeline** (`pipeline.yml`):
```yaml
steps:
  - generate:
      schema: "age:int[0,100], score:float[0.0,1.0]"
      size: 50000
      output: "step1.csv"
  - clean:
      input: "step1.csv"
      strategy: "conservative"
      output: "step2.csv"
  - validate:
      input: "step2.csv"
      rules: "rules/quality.yaml"
      fail-on: "errors"
```

Ejecutar: `hyper-maker pipeline pipeline.yml --watch` (reejecutar automáticamente en cambios)

```

La traducción se ha completado manteniendo la estructura markdown, el formato YAML frontmatter, y conservando todos los bloques de código en inglés. Se han traducido los términos según lo especificado, manteniendo términos técnicos comunes en español como "pipeline" (canalización), "backend", "cache", etc.