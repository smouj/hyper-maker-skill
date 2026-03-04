---
name: hyper-maker
description: AI-powered maker for data tasks - automate data generation, transformation, and analysis
version: "1.0.0"
author: "Hyper Maker Team"
tags: ["data", "ai", "automation"]
maintainer: "SMOUJBOT"
category: "data-processing"
---

# Hyper Maker

AI-powered CLI tool for automated data tasks including generation, transformation, cleaning, and analysis.

## Purpose

Hyper Maker automates complex data workflows using AI. Real use cases:

- Generate synthetic datasets matching specific schemas and distributions
- Transform data between formats (CSV → JSON, XML → Parquet, etc.) with intelligent mapping
- Clean messy data: detect outliers, handle missing values, normalize formats
- Analyze datasets: generate summary statistics, detect patterns, create data quality reports
- Augment existing datasets with AI-generated synthetic samples
- Validate data against custom rules and constraints
- Create data pipelines from natural language descriptions

## Scope

Commands Hyper Maker executes:

- `hyper-maker generate <schema> --format <csv|json|parquet> --size <N> --seed <int>`
- `hyper-maker transform <input> --to <format> --map <field_mappings> --filter <conditions>`
- `hyper-maker clean <input> --strategy <auto|conservative|aggressive> --output <clean.csv>`
- `hyper-maker analyze <input> --profile --report <report.json> --correlations`
- `hyper-maker augment <input> --samples <N> --preserve-distribution --noise <float>`
- `hyper-maker validate <input> --rules <rules.yaml> --fail-on <errors|warnings>`
- `hyper-maker pipeline <pipeline.yml> --dry-run --parallel --cache-dir <path>`

## Work Process

1. **Input Validation**
   - Verify file existence and format
   - Check disk space (needs 2× input size for temporary files)
   - Validate environment variables: `HYPER_MAKER_API_KEY`, `HYPER_MAKER_MODEL` (default: "claude-3.5-sonnet")

2. **AI Processing**
   - Build context from input data (sample first 100 rows)
   - Construct domain-specific prompts based on task type
   - Execute AI inference with temperature=0.2 for deterministic outputs
   - Parse AI response into structured operations

3. **Transformation Execution**
   - Apply transformations using pandas/polars (auto-detect based on file size: <100MB → pandas, >100MB → polars)
   - Stream processing for files >1GB (chunk size: 100,000 rows)
   - Progress logging to `~/.hyper-maker/logs/`

4. **Validation & Quality**
   - Post-transformation data quality checks
   - Schema validation if target schema provided
   - Compute quality metrics: completeness, uniqueness, validity scores

5. **Output & Metadata**
   - Write output in requested format
   - Generate `output.metadata.json` with operation summary, row counts, quality scores
   - Log operation to `~/.hyper-maker/history.jsonl`

## Golden Rules

1. **Never override source files** - always create new outputs unless `--inplace` explicitly set (default: false)
2. **Always preserve data types** - maintain numeric types, dates, categoricals; never convert everything to strings
3. **Seed everything** - random operations must use `--seed` for reproducibility; default seed from `HYPER_MAKER_SEED` env or timestamp-based
4. **Respect memory limits** - for files >500MB, enforce streaming mode; fail gracefully if memory >80% of system RAM
5. **Log all AI prompts** - store prompts and responses in `~/.hyper-maker/audit/` for debugging
6. **Validate before write** - schema and quality checks must pass before output file is moved to final location
7. **Cache transformations** - reuse identical transformations within 24h if cache enabled (`--cache`)

## Examples

### Example 1: Generate synthetic customer data
```bash
hyper-maker generate '{"name": "string", "age": "int[18,80]", "email": "email", "signup_date": "date[2020-01-01,2024-12-31]"}' \
  --format csv \
  --size 10000 \
  --seed 42 \
  --output customers.csv
```

**Output**: `customers.csv` with 10,000 rows, `customers.metadata.json` with generation parameters

### Example 2: Clean messy product data
```bash
hyper-maker clean products.csv \
  --strategy aggressive \
  --remove-duplicates \
  --normalize-text \
  --output products_clean.csv
```

**What happens**:
- AI analyzes first 100 rows to detect format inconsistencies
- Removes duplicate Product IDs
- Standardizes: "N/A", "null", "" → proper nulls
- Trims whitespace, fixes case in categorical fields

### Example 3: Transform with AI field mapping
```bash
hyper-maker transform legacy_data.xml \
  --to json \
  --map "customer_id=id, full_name=name, phone_number=contact.phone" \
  --filter "status == 'active' AND age >= 18" \
  --output customers.json
```

**Field mapping**: nested target paths supported with dot notation

### Example 4: Analyze dataset with correlations
```bash
hyper-maker analyze sales.csv \
  --profile \
  --correlations \
  --report analysis_report.md
```

**Report includes**:
- Schema summary
- Missing value matrix
- Correlation heatmap data
- Top 10 anomalies detected
- Data quality score (0-100)

### Example 5: Augment imbalanced dataset
```bash
hyper-maker augment fraud_transactions.csv \
  --samples 5000 \
  --preserve-distribution \
  --noise 0.05 \
  --output fraud_augmented.csv
```

AI generates synthetic fraud samples that maintain statistical properties while adding variation.

## Dependencies & Requirements

**Required**:
- Python 3.9+
- `pandas>=2.0.0` (default) or `polars>=0.19.0` (large files)
- `numpy>=1.24.0`
- `scikit-learn>=1.3.0` (for augmentation)
- `openai>=1.0.0` or `anthropic>=0.8.0` (AI provider)
- 4GB RAM minimum, 16GB recommended for >1GB datasets

**Optional**:
- `pyarrow>=12.0.0` (Parquet/Feather support)
- `orjson>=3.9.0` (fast JSON)
- `pandera>=0.16.0` (schema validation)

**Environment Variables**:
- `HYPER_MAKER_API_KEY` - AI provider API key
- `HYPER_MAKER_MODEL` - model name (default: "claude-3-5-sonnet-20241022")
- `HYPER_MAKER_PROVIDER` - "openai" or "anthropic" (default: "anthropic")
- `HYPER_MAKER_CACHE_DIR` - cache directory (default: "~/.hyper-maker/cache")
- `HYPER_MAKER_MAX_MEMORY_PERCENT` - memory limit (default: 80)

## Verification Steps

After any operation, verify:
1. Output file exists and size > 0
2. Row count matches expected (check `output.metadata.json`)
3. Quality score > threshold (default: 70 for clean, 50 for generate)
4. No errors in `~/.hyper-maker/logs/operation-<timestamp>.log`
5. Schema compliance if validation was part of pipeline

Quick verification command:
```bash
hyper-maker verify <output_file> --checks existence,rows,quality,schema
```

## Troubleshooting

**"API key not set"**
- Set `HYPER_MAKER_API_KEY` in environment
- Or use `hyper-maker auth --key <your-key>`

**"Memory limit exceeded"**
- Use `--chunk-size <N>` to process in chunks
- Switch to polars: `hyper-maker use-backend polars`
- Reduce `--max-memory-percent`

**"AI prompt timeout"**
- Increase timeout: `hyper-maker config set ai_timeout 120`
- Reduce sample size: `--sample-rows 50`
- Check `~/.hyper-maker/audit/latest_prompt.txt` for prompt complexity

**"Schema validation failed"**
- Inspect `output.validation_errors.json`
- Review rule syntax in `rules.yaml`
- Set `--strict false` to allow warnings only

**"Slow performance on large files"**
- Enable parallel processing: `--parallel-workers 4`
- Use streaming: `--streaming true`
- Consider converting to Parquet first

**"AI hallucination in field mapping"**
- Use explicit mappings: `--map "source=target"` (no AI inference)
- Set `--ai-reasoning false` to disable AI field matching

## Rollback Commands

Undo operations:

```bash
# Revert to previous version from history
hyper-maker rollback <output_file> --to <timestamp|previous|commit-hash>

# List available rollback points
hyper-maker history <file> --show-versions

# Restore original from backup (if --backup-source was used)
hyper-maker restore <output_file> --source <original.csv>

# Full pipeline rollback (revert all outputs from a pipeline run)
hyper-maker pipeline-rollback <pipeline_run_id>

# Clean cache (doesn't rollback, but clears state)
hyper-maker cache clear --older-than 7d
```

**Note**: History is stored in `~/.hyper-maker/operations.db` (SQLite). Backup source files with `--backup-source` flag to enable full restore capability.

## Advanced Usage

**Custom AI prompts**:
```bash
hyper-maker transform data.csv \
  --custom-prompt "Convert all dates to ISO format, remove rows where revenue is negative, and flag any rows where email is invalid" \
  --output cleaned.csv
```

**Pipeline definition** (`pipeline.yml`):
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

Run: `hyper-maker pipeline pipeline.yml --watch` (auto-rerun on changes)

---