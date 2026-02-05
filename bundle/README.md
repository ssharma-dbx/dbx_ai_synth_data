# Databricks Asset Bundle - AI Data Generator

This folder contains the Databricks Asset Bundle configuration for deploying the AI Data Generator as automated jobs.

## 📁 Structure

```
bundle/
├── databricks.yml          # Main bundle configuration (with sync)
├── job_parameters.conf     # Parameter templates for quick setup
└── resources/
    └── jobs.yml           # Job definitions (5 pre-configured jobs)

Note: The notebook (ai_data_generator.py) is synced from the parent directory.
```

## 🔄 File Sync

The bundle automatically syncs the notebook to the workspace:

- **Local**: `../ai_data_generator.py` (parent directory)
- **Workspace**: `${workspace.file_path}/ai_data_generator` (in the bundle's root path)
- **Jobs Reference**: `${workspace.file_path}/ai_data_generator` (relative path variable)

During deployment, the notebook is automatically uploaded to your workspace at the bundle's root path, and jobs reference it using the `${workspace.file_path}` variable for portability across environments.

### Why Relative Paths?

Using `${workspace.file_path}` instead of hardcoded paths provides:
- ✅ **Environment-agnostic**: Works in dev, staging, and prod without changes
- ✅ **User-agnostic**: No hardcoded usernames or paths
- ✅ **Portable**: Anyone can deploy to their workspace
- ✅ **Maintainable**: Single source of truth for file locations

## 🚀 Quick Start

### 1. Configure Workspace ⚠️ REQUIRED

Edit `databricks.yml` (line 42) with your actual workspace URL:

```yaml
workspace:
  host: "https://your-actual-workspace.cloud.databricks.net/"
```

**This is mandatory!** The bundle cannot deploy without a valid workspace URL.

### 2. Authenticate

```bash
databricks configure --token
# Enter your workspace URL and personal access token
```

### 3. Validate

```bash
databricks bundle validate -t dev
```

### 4. Deploy

```bash
databricks bundle deploy -t dev
```

### 5. Run a Job

```bash
# Run pre-configured healthcare patients job
databricks bundle run generate_patients_job -t dev

# Or run generic job with custom parameters
databricks bundle run ai_data_generator_job -t dev \
  --param industry="finance" \
  --param domain="transactions" \
  --param num_rows="500"
```

## 📋 Available Jobs

### 1. ai_data_generator_job
**Generic template** - Fully customizable via parameters

**Default Parameters:**
- Industry: healthcare
- Domain: patient records
- Rows: 50,000
- All parameters can be overridden at runtime

### 2. generate_patients_job
**Healthcare patient records** - Pre-configured

**Generates:**
- Table: `{catalog}.{schema}.patients`
- Rows: 50,000
- Constraints: Patient ID from 10000, ages 18-95

### 3. generate_products_job
**Retail product inventory** - Pre-configured with custom schema

**Generates:**
- Table: `{catalog}.{schema}.products`
- Rows: 20,000
- Custom schema with product details

### 4. generate_transactions_job
**Finance transactions** - Pre-configured

**Generates:**
- Table: `{catalog}.{schema}.transactions`
- Rows: 10,000
- Amounts: $10-$10,000

## 🌍 Environments

### Development (dev)
```bash
databricks bundle deploy -t dev
```

- Path: `~/.bundle/ai_data_generator/dev`
- Catalog: `dev_catalog` (default, update in databricks.yml)
- Schema: `dev_schema` (default, update in databricks.yml)

### Staging (staging)
```bash
databricks bundle deploy -t staging
```

- Path: `/Shared/.bundle/ai_data_generator/staging`
- Catalog: `staging_catalog` (update in databricks.yml)
- Schema: `staging_schema` (update in databricks.yml)

### Production (prod)
```bash
databricks bundle deploy -t prod
```

- Path: `/Shared/.bundle/ai_data_generator/prod`
- Catalog: `prod_catalog` (update in databricks.yml)
- Schema: `prod_schema` (update in databricks.yml)

## ⚙️ Configuration

### Variables (databricks.yml)

These variables can be overridden per environment:

| Variable | Default | Description |
|----------|---------|-------------|
| `environment` | "dev" | Environment name |
| `catalog` | "default_catalog" | Unity Catalog name (⚠️ Update for your workspace!) |
| `schema` | "default_schema" | Schema name (⚠️ Update for your workspace!) |
| `cluster_node_type` | (commented out) | Node type for clusters - define if not using serverless |
| `cluster_spark_version` | (commented out) | Databricks runtime version - define if not using serverless |

**Note**: The `cluster_node_type` and `cluster_spark_version` variables are commented out by default. The jobs are configured to use serverless compute. If you want to use traditional clusters, uncomment these variables and update the cluster configuration in `resources/jobs.yml`.

### Override Variables

**Method 1: Edit databricks.yml targets**

```yaml
targets:
  dev:
    variables:
      catalog: my_catalog
      schema: my_schema
```

**Method 2: Environment variables**

```bash
export DATABRICKS_CATALOG="my_catalog"
export DATABRICKS_SCHEMA="my_schema"
databricks bundle deploy -t dev
```

## 📝 Customizing Jobs

### Edit Existing Job

Edit `resources/jobs.yml` and modify base_parameters:

```yaml
base_parameters:
  industry: "your_industry"
  domain: "your_domain"
  table_name: "your_table"
  num_rows: "500"
  # ... other parameters
```

### Add New Job

Add to `resources/jobs.yml`:

```yaml
resources:
  jobs:
    my_custom_job:
      name: "[${bundle.target}] My Custom Job"
      tasks:
        - task_key: generate_my_data
          new_cluster:
            spark_version: ${var.cluster_spark_version}
            node_type_id: ${var.cluster_node_type}
            num_workers: 0
            spark_conf:
              spark.databricks.cluster.profile: serverless
          notebook_task:
            notebook_path: ${workspace.file_path}/ai_data_generator
            base_parameters:
              industry: "manufacturing"
              domain: "equipment maintenance"
              table_name: "maintenance_logs"
              target_catalog: ${var.catalog}
              target_schema: ${var.schema}
              num_rows: "300"
              ai_model_endpoint: "databricks-meta-llama-3-3-70b-instruct"
              custom_schema_json: ""
              column_constraints_json: '{}'
```

**Important**: Use `${workspace.file_path}/ai_data_generator` for the notebook path - this ensures the job works across all environments.

### Use Parameter Templates

Copy configurations from `job_parameters.conf`:

```bash
# View available templates
cat job_parameters.conf

# Examples include:
# - Healthcare: patients, appointments, prescriptions
# - Retail: products, orders, customers
# - Finance: transactions, loans, portfolios
# - Manufacturing: sensors, maintenance
# - And many more!
```

## 🔧 Common Commands

```bash
# Validate configuration
databricks bundle validate -t dev

# View deployment plan
databricks bundle summary -t dev

# Deploy to environment
databricks bundle deploy -t dev

# List deployed jobs
databricks bundle jobs list -t dev

# Run a specific job
databricks bundle run <job_name> -t dev

# Run with custom parameters
databricks bundle run ai_data_generator_job -t dev \
  --param industry="retail" \
  --param num_rows="1000"

# Destroy deployment
databricks bundle destroy -t dev
```

## 🐛 Troubleshooting

### Issue: Variable not defined

**Error:**
```
Error: variable environment is not defined but is assigned a value
```

**Solution:** This is already fixed. The `environment` variable is defined in `databricks.yml`.

### Issue: Notebook not found

**Error:**
```
Error: notebook not found
```

**Solution:** The notebook is automatically synced from the parent directory. The notebook path uses a relative variable:
```yaml
notebook_path: ${workspace.file_path}/ai_data_generator
```

Make sure you've deployed the bundle:
```bash
databricks bundle deploy -t dev
```

### Issue: Authentication failed

**Solution:**
```bash
databricks configure --token
# Enter your workspace URL and token
```

### More Help

See `../docs/DEPLOYMENT.md` for comprehensive deployment guide.

## ✅ Checklist

Before deploying:
- [ ] **Workspace URL updated** in `databricks.yml` (line 42) - THIS IS REQUIRED!
- [ ] Catalog and schema variables updated in `databricks.yml`
- [ ] Authentication configured (`databricks configure --token`)
- [ ] Bundle validated successfully (`databricks bundle validate -t dev`)
- [ ] Target catalog/schema exists in your workspace
- [ ] AI endpoint access verified

## 🔗 Links

- [Databricks Asset Bundles Documentation](https://docs.databricks.com/dev-tools/bundles/)
- [Databricks CLI Documentation](https://docs.databricks.com/dev-tools/cli/)
- [Unity Catalog Documentation](https://docs.databricks.com/data-governance/unity-catalog/)

---

**Quick Start:** 
1. Update workspace URL in `databricks.yml` (line 42) ⚠️ REQUIRED
2. Update catalog/schema in environment targets
3. Run `databricks bundle deploy -t dev`
4. Run `databricks bundle run generate_patients_job -t dev`

**Need Help?** Check `../docs/DEPLOYMENT.md` for detailed deployment guide
