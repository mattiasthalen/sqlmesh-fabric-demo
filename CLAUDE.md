# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a SQLMesh demo project for Microsoft Fabric integration. SQLMesh is a data transformation framework that enables SQL-based data pipelines with built-in testing, version control, and deployment capabilities. This project demonstrates using SQLMesh with Microsoft Fabric (Azure's analytics platform) as the execution engine.

## Setup Requirements

### Azure Service Principal
Create a service principal with the following Fabric API delegated scopes:
- `Item.ReadWrite.All` - Manage Fabric items

Reference: [Fabric Warehouse API Scopes](https://learn.microsoft.com/en-us/rest/api/fabric/warehouse/items/create-warehouse?tabs=HTTP#required-delegated-scopes)

### Fabric Resources
1. **Fabric Workspace**: Create a workspace for the SQLMesh project
2. **Fabric Warehouse**: Create a warehouse named `demo` for model execution
3. **SQL Database**: Create a database for SQLMesh state management
4. **Lakehouse** (optional): For storing code and data files in Fabric
5. **Notebook** (optional): For running the SQLMesh project

### Workspace Permissions
Add the service principal as a **Contributor** to the Fabric workspace to enable full access to workspace resources.

Reference: [Service Principals in Fabric](https://learn.microsoft.com/en-us/fabric/data-warehouse/service-principals)

### Environment Variables
Set these environment variables before running SQLMesh:
```bash
export FABRIC__WAREHOUSE_ENDPOINT="your-warehouse-endpoint.datawarehouse.fabric.microsoft.com"
export FABRIC__WORKSPACE_ID="your-workspace-id"
export FABRIC__STATE_ENDPOINT="your-state-db-endpoint.database.windows.net"
export FABRIC__STATE_DATABASE="sqlmesh__state"
export CREDENTIALS__AZURE_TENANT_ID="your-tenant-id"
export CREDENTIALS__AZURE_CLIENT_ID="your-service-principal-client-id"
export CREDENTIALS__AZURE_CLIENT_SECRET="your-service-principal-secret"
```

### Database Permissions
Grant the service principal the following role on the SQLMesh state database:
- `db_owner` - Full access to create and manage SQLMesh state tables

## Key Commands

### Install Dependencies
```bash
# Using uv (if available)
uv sync

# Or using pip
pip install "sqlmesh[fabric,mssql,lsp]"

# Or from the main branch of SQLMesh
pip install "sqlmesh[fabric,mssql,lsp] @ git+https://github.com/TobikoData/sqlmesh.git@main"
```

### SQLMesh Commands
```bash
# Run plan for development environment
uv run sqlmesh plan dev

# Run plan for production with auto-apply
uv run sqlmesh plan prod --run --auto-apply --no-prompts

# Include unmodified models in plan
uv run sqlmesh plan prod --run --auto-apply --no-prompts --include-unmodified

# Run tests
uv run sqlmesh test

# Run audits
uv run sqlmesh audit

# UI for interactive development
uv run sqlmesh ui

# Validate models and configuration
uv run sqlmesh validate

# Format SQL files
uv run sqlmesh format
```

### Testing Individual Models
```bash
# Test a specific model
uv run sqlmesh test tests/test_full_model.yaml

# Run all tests
uv run sqlmesh test
```

## Architecture

### Directory Structure
- **models/**: SQL model definitions using SQLMesh syntax
  - `seed_model.sql`: Base model that reads from seed data
  - `incremental_model.sql`: Time-based incremental model
  - `full_model.sql`: Full refresh aggregation model
- **seeds/**: Static CSV data files
- **audits/**: Data quality checks that run after model execution
- **tests/**: Unit tests for models in YAML format
- **notebooks/**: Jupyter notebooks for local development/testing

### Model Dependencies
```
seeds/seed_data.csv
    ↓
seed_model
    ↓
incremental_model
    ↓
full_model (with audit: assert_positive_order_ids)
```

### Environment Configuration
The project uses environment variables for Fabric connection, configured in `config.yaml`:
- `FABRIC__WAREHOUSE_ENDPOINT`: Fabric warehouse endpoint
- `FABRIC__WORKSPACE_ID`: Fabric workspace ID
- `FABRIC__STATE_ENDPOINT`: State database endpoint
- `FABRIC__STATE_DATABASE`: State database name
- `CREDENTIALS__AZURE_TENANT_ID`: Azure AD tenant
- `CREDENTIALS__AZURE_CLIENT_ID`: Service principal client ID
- `CREDENTIALS__AZURE_CLIENT_SECRET`: Service principal secret

### Key SQLMesh Concepts
- **Models**: SQL definitions with metadata (kind, grain, cron schedule)
- **Kinds**: FULL (complete refresh), INCREMENTAL_BY_TIME_RANGE (time-based incremental)
- **Audits**: Post-execution data quality checks
- **Grains**: Column(s) defining model uniqueness
- **State Connection**: Separate database for tracking model versions and execution state

### Fabric Integration
The project integrates with Microsoft Fabric through:
1. Service Principal authentication
2. Fabric Warehouse as execution engine (using `type: fabric` connection)
3. SQL Server database for state management
4. Fabric notebooks for orchestrated execution in cloud environment

### Running in Fabric Notebook
To run SQLMesh in a Fabric environment:
1. Import the notebook from `notebooks/sqlmesh__runner.ipynb` into your Fabric workspace
2. Add the codebase Lakehouse as a resource to the notebook
3. Configure Azure Key Vault with required secrets
4. Update the notebook with your Azure DevOps organization and project details
5. Run the notebook to execute SQLMesh plans

The notebook handles:
- Secret retrieval from Azure Key Vault
- Git repository cloning from Azure DevOps
- SQLMesh execution with proper environment setup