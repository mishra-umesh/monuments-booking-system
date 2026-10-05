# Deployment & Infrastructure Guide

## Overview

This guide covers deployment architecture, infrastructure setup, and operational procedures for the Monuments Booking System using SQL Server as the relational database.

**Database Platform:** SQL Server 2019 or later (SQL Server 2022 recommended)

**Deployment Targets:**
- Local Development (SQL Server Express or full edition)
- Staging Environment (Azure SQL Database or on-premises SQL Server)
- Production (Azure SQL Database or managed SQL Server instance)

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Database Setup](#database-setup)
3. [Application Deployment](#application-deployment)
4. [Infrastructure Architecture](#infrastructure-architecture)
5. [Configuration Management](#configuration-management)
6. [Security Setup](#security-setup)
7. [Monitoring & Logging](#monitoring--logging)
8. [Backup & Disaster Recovery](#backup--disaster-recovery)
9. [Scaling & Performance](#scaling--performance)
10. [Troubleshooting](#troubleshooting)

---

## System Requirements

### Development Environment

**Operating System:**
- Windows 10/11 (Professional or Enterprise) or Windows Server 2019/2022
- macOS 10.15+ (with Docker)
- Linux (Ubuntu 20.04 LTS or later, with Docker)

**Software:**
- .NET 7 SDK or later (`dotnet --version`)
- SQL Server 2019 Express or Developer Edition
- SQL Server Management Studio (SSMS) 19.0+
- Git 2.32+
- Docker & Docker Compose (optional, for containerized dev)
- Visual Studio 2022 or VS Code with C# extensions

**Hardware (Minimum):**
- 8 GB RAM
- 20 GB free disk space
- Dual-core processor

**Network:**
- Internet access for NuGet package restoration
- TCP ports 1433 (SQL Server), 5000 (API), 5001 (Admin MVC), 5002 (Public MVC)

### Staging Environment

**Infrastructure:**
- Azure App Service (Standard plan or higher)
- Azure SQL Database (Standard S2 tier minimum)
- Azure Storage Account (for images and exports)
- Azure Application Insights (for monitoring)

**Or On-Premises:**
- Windows Server 2019/2022 (Standard or Datacenter Edition)
- SQL Server 2019 Standard or Enterprise Edition
- IIS 10+
- SSL/TLS certificates

### Production Environment

**Infrastructure:**
- Azure App Service (Premium plan recommended for scaling)
- Azure SQL Database (Business Critical or Premium tier for SLAs)
- Azure Application Insights (mandatory)
- Azure Key Vault (for secrets management)
- Azure Storage Account (geo-redundant for images)
- Content Delivery Network (CDN) for static assets
- Azure Front Door or similar (for DDoS protection)

**Or On-Premises:**
- Windows Server 2019/2022 Datacenter Edition
- SQL Server 2019 Enterprise Edition
- Load Balancer (F5, Citrix, or cloud provider)
- IIS 10+ (multiple servers for high availability)
- Redundant SQL Server instances (Always On Availability Groups)
- SSL/TLS certificates (from trusted CA)

---

## Database Setup

### SQL Server Installation

#### Option 1: Local Development (SQL Server Express)

```bash
# Download SQL Server 2022 Developer or Express Edition
# https://www.microsoft.com/en-us/sql-server/sql-server-downloads

# Install with default settings:
# - Named Instance: SQLEXPRESS
# - Mixed Mode Authentication (SQL + Windows)
# - Default collation: SQL_Latin1_General_CP1_CI_AS
# - Tempdb data/log: Default paths
```

#### Option 2: Azure SQL Database

```bash
# Create Azure resource group
az group create --name monuments-rg --location eastus

# Create Azure SQL Server
az sql server create \
    --name monuments-sqlserver \
    --resource-group monuments-rg \
    --location eastus \
    --admin-user sqladmin \
    --admin-password "SecurePassword@123"

# Create Azure SQL Database
az sql db create \
    --server monuments-sqlserver \
    --name MonumentsBooking \
    --resource-group monuments-rg \
    --tier Standard \
    --capacity 20 \
    --backup-storage-redundancy Geo

# Configure firewall rule (allow your IP)
az sql server firewall-rule create \
    --server monuments-sqlserver \
    --name AllowClientIp \
    --resource-group monuments-rg \
    --start-ip-address 203.0.113.0 \
    --end-ip-address 203.0.113.255
```

#### Option 3: Docker Container (Development)

```bash
# Run SQL Server in Docker
docker run -e 'ACCEPT_EULA=Y' \
    -e 'SA_PASSWORD=YourPassword@123' \
    -p 1433:1433 \
    --name mssql-server \
    -d mcr.microsoft.com/mssql/server:2022-latest

# Verify container is running
docker logs mssql-server
```

### Database Creation

#### Via dotnet CLI (Code-First)

```bash
cd src/MonumentsAPI

# Create connection string in appsettings.json or environment variable
# CONNECTIONSTRING="Server=.;Initial Catalog=MonumentsBooking;Integrated Security=true;"

# Create initial migration (if not already done)
dotnet ef migrations add InitialCreate

# Apply migrations to database (creates all tables)
dotnet ef database update --verbose

# Verify database structure
sqlcmd -S . -Q "SELECT name FROM sys.tables WHERE name LIKE 'Bookings' OR name LIKE 'Monuments';"
```

#### Via SQL Management Studio (Manual)

```sql
-- Connect to SQL Server (local or Azure)
-- Execute in New Query window

-- Create database
CREATE DATABASE MonumentsBooking
    COLLATE SQL_Latin1_General_CP1_CI_AS;
GO

-- Create login and user (Windows Authentication)
CREATE LOGIN [DOMAIN\appservice_account] FROM WINDOWS;
GO

USE MonumentsBooking;
GO

CREATE USER [appservice_account] FOR LOGIN [DOMAIN\appservice_account];
GO

-- Grant permissions
ALTER ROLE db_owner ADD MEMBER [appservice_account];
GO

-- Create SQL Server login for development
CREATE LOGIN sa_dev WITH PASSWORD = 'DevPassword@123';
GO

CREATE USER [sa_dev] FOR LOGIN [sa_dev];
GO

ALTER ROLE db_owner ADD MEMBER [sa_dev];
GO

-- Verify tables are created (after running migrations)
SELECT name FROM sys.tables ORDER BY name;
GO
```

### Connection String Configuration

#### Development (SQL Server Express)

```
Server=.;Initial Catalog=MonumentsBooking;Integrated Security=true;Encrypt=false;
```

#### Development (Docker)

```
Server=localhost,1433;Initial Catalog=MonumentsBooking;User Id=sa;Password=YourPassword@123;Encrypt=false;
```

#### Azure SQL Database

```
Server=tcp:monuments-sqlserver.database.windows.net,1433;Initial Catalog=MonumentsBooking;Persist Security Info=False;User ID=sqladmin;Password=SecurePassword@123;MultipleActiveResultSets=False;Encrypt=True;Connection Timeout=30;
```

#### Production (On-Premises with SQL Server)

```
Server=sql-server-prod.monuments.local;Initial Catalog=MonumentsBooking;User ID=monuments_app;Password=ProductionPassword@123;Encrypt=true;Connection Timeout=30;
```

### Post-Installation Tasks

```sql
-- Enable SQL Server Agent (for backup jobs)
USE master;
GO
EXEC msdb.dbo.sp_set_sqlagent_properties @auto_start = 1;
GO

-- Configure backup location
-- Create directory: C:\SQLBackups
-- Grant full permissions to SQL Server service account

-- Create backup job
USE msdb;
GO
EXEC sp_add_job @job_name = 'MonumentsBackupFull';
GO

EXEC sp_add_jobstep 
    @job_name = 'MonumentsBackupFull',
    @step_name = 'Backup Full Database',
    @subsystem = 'TSQL',
    @command = 'BACKUP DATABASE MonumentsBooking TO DISK = ''C:\SQLBackups\monuments_full_$(ESCAPE_SQUOTE(STRDATETIME(GETDATE(), 102))).bak'' WITH INIT, COMPRESSION;';
GO

-- Schedule job to run daily at 2 AM
EXEC sp_add_schedule @schedule_name = 'DailyAt2AM', @freq_type = 4, @freq_interval = 1, @active_start_time = 020000;
GO

EXEC sp_attach_schedule @job_name = 'MonumentsBackupFull', @schedule_name = 'DailyAt2AM';
GO

-- Enable job
EXEC sp_update_job @job_name = 'MonumentsBackupFull', @enabled = 1;
GO
```

---

## Application Deployment

### Local Development Deployment

#### Prerequisites
```bash
# Verify .NET SDK installation
dotnet --version

# Verify SQL Server connectivity
sqlcmd -S . -Q "SELECT @@VERSION;"

# Clone repository
git clone https://github.com/mishra-umesh/monuments-booking-system.git
cd monuments-booking-system
```

#### Build & Run

```bash
# Restore NuGet packages
dotnet restore

# Build solution
dotnet build --configuration Release

# Run EF Core migrations
cd src/MonumentsAPI
dotnet ef database update --verbose

# Start API server (terminal 1)
dotnet run --configuration Release
# API accessible at: https://localhost:5000

# Start Admin MVC (terminal 2)
cd ../MonumentsAdmin
dotnet run --configuration Release
# Admin UI accessible at: https://localhost:5001

# Start Public MVC (terminal 3)
cd ../MonumentsPublic
dotnet run --configuration Release
# Public UI accessible at: https://localhost:5002
```

#### Docker Compose (Full Stack)

```yaml
version: '3.9'

services:
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      SA_PASSWORD: "YourPassword@123"
      MSSQL_AGENT_ENABLED: "true"
    ports:
      - "1433:1433"
    volumes:
      - sqlserver-data:/var/opt/mssql/data
      - sqlserver-log:/var/opt/mssql/log
    healthcheck:
      test: ["CMD", "/opt/mssql-tools/bin/sqlcmd", "-S", "localhost", "-U", "sa", "-P", "YourPassword@123", "-Q", "SELECT 1"]
      interval: 10s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ./src/MonumentsAPI
      dockerfile: Dockerfile
    environment:
      ConnectionString: "Server=sqlserver;Database=MonumentsBooking;User Id=sa;Password=YourPassword@123;Encrypt=false;"
      ASPNETCORE_ENVIRONMENT: "Development"
      ASPNETCORE_URLS: "https://+:5000"
    ports:
      - "5000:5000"
    depends_on:
      sqlserver:
        condition: service_healthy
    volumes:
      - ./src/MonumentsAPI:/app

  admin-mvc:
    build:
      context: ./src/MonumentsAdmin
      dockerfile: Dockerfile
    environment:
      ApiBaseUrl: "http://api:5000"
      ASPNETCORE_ENVIRONMENT: "Development"
      ASPNETCORE_URLS: "https://+:5001"
    ports:
      - "5001:5001"
    depends_on:
      - api
    volumes:
      - ./src/MonumentsAdmin:/app

  public-mvc:
    build:
      context: ./src/MonumentsPublic
      dockerfile: Dockerfile
    environment:
      ApiBaseUrl: "http://api:5000"
      ASPNETCORE_ENVIRONMENT: "Development"
      ASPNETCORE_URLS: "https://+:5002"
    ports:
      - "5002:5002"
    depends_on:
      - api
    volumes:
      - ./src/MonumentsPublic:/app

volumes:
  sqlserver-data:
  sqlserver-log:
```

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f api

# Stop services
docker-compose down

# Remove volumes (clears database)
docker-compose down -v
```

### Staging Deployment (Azure App Service + Azure SQL)

#### Azure CLI Setup

```bash
# Login to Azure
az login

# Create resource group
az group create --name monuments-staging --location eastus

# Create App Service Plan
az appservice plan create \
    --name monuments-staging-plan \
    --resource-group monuments-staging \
    --sku S1 \
    --is-linux

# Create Web App for API
az webapp create \
    --name monuments-api-staging \
    --resource-group monuments-staging \
    --plan monuments-staging-plan \
    --runtime "DOTNETCORE|7.0"

# Create Web App for Admin MVC
az webapp create \
    --name monuments-admin-staging \
    --resource-group monuments-staging \
    --plan monuments-staging-plan \
    --runtime "DOTNETCORE|7.0"

# Create Web App for Public MVC
az webapp create \
    --name monuments-public-staging \
    --resource-group monuments-staging \
    --plan monuments-staging-plan \
    --runtime "DOTNETCORE|7.0"

# Create Azure SQL Database
az sql db create \
    --server monuments-sqlserver \
    --name MonumentsBookingStaging \
    --resource-group monuments-staging \
    --tier Standard \
    --capacity 20
```

#### Deploy via GitHub Actions

Create `.github/workflows/deploy-staging.yml`:

```yaml
name: Deploy to Staging

on:
  push:
    branches: [develop]

env:
  AZURE_WEBAPP_NAME: monuments-api-staging
  DOTNET_VERSION: '7.0.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Restore dependencies
      run: dotnet restore
    
    - name: Build
      run: dotnet build --configuration Release --no-restore
    
    - name: Run tests
      run: dotnet test --configuration Release --no-build --verbosity normal
    
    - name: Publish API
      run: dotnet publish src/MonumentsAPI -c Release -o ${{env.DOTNET_ROOT}}/api
    
    - name: Deploy API to Azure App Service
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ env.AZURE_WEBAPP_NAME }}
        slot-name: staging
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE_API }}
        package: ${{env.DOTNET_ROOT}}/api
    
    - name: Run database migrations
      run: |
        cd src/MonumentsAPI
        dotnet tool install -g dotnet-ef
        dotnet ef database update \
          --connection "Server=tcp:monuments-sqlserver.database.windows.net,1433;Initial Catalog=MonumentsBookingStaging;User ID=${{ secrets.DB_USER }};Password=${{ secrets.DB_PASSWORD }};Encrypt=True;"
```

#### Manual Deployment (Azure CLI)

```bash
# Build release packages
cd src/MonumentsAPI
dotnet publish -c Release -o ./publish

# Deploy API using Az CLI
az webapp up \
    --name monuments-api-staging \
    --resource-group monuments-staging \
    --plan monuments-staging-plan \
    --package ./publish \
    --runtime "DOTNETCORE|7.0"

# Configure connection string
az webapp config appsettings set \
    --name monuments-api-staging \
    --resource-group monuments-staging \
    --settings \
    ConnectionString="Server=tcp:monuments-sqlserver.database.windows.net,1433;Initial Catalog=MonumentsBookingStaging;User ID=sqladmin;Password=$DB_PASSWORD;Encrypt=True;" \
    ASPNETCORE_ENVIRONMENT="Staging"

# Run migrations
az webapp deployment slot create \
    --name monuments-api-staging \
    --resource-group monuments-staging \
    --slot migrations

# Deploy to slot, run migrations, then swap to production slot
```

### Production Deployment (Azure or On-Premises)

#### Production Checklist

- [ ] SSL/TLS certificates installed and valid
- [ ] Database backups configured and tested
- [ ] Application Insights monitoring enabled
- [ ] Key Vault secrets configured (API keys, connection strings, passwords)
- [ ] Firewall rules configured (IP whitelisting)
- [ ] Load balancer health checks configured
- [ ] Log aggregation setup (Azure Monitor or ELK stack)
- [ ] Auto-scaling policies configured
- [ ] Disaster recovery procedures documented and tested
- [ ] Security scanning completed (OWASP, dependency audit)

#### Azure Production Deployment

```bash
# Create production resource group
az group create --name monuments-prod --location eastus --tags environment=production

# Create Premium App Service Plan (for scaling)
az appservice plan create \
    --name monuments-prod-plan \
    --resource-group monuments-prod \
    --sku P1V2 \
    --is-linux

# Create Web Apps
az webapp create \
    --name monuments-api-prod \
    --resource-group monuments-prod \
    --plan monuments-prod-plan \
    --runtime "DOTNETCORE|7.0"

# Create Azure SQL Database (Premium tier)
az sql db create \
    --server monuments-sqlserver-prod \
    --name MonumentsBookingProd \
    --resource-group monuments-prod \
    --tier Premium \
    --capacity 250 \
    --backup-storage-redundancy Geo

# Enable automatic failover group (for HA)
az sql failover-group create \
    --name monuments-prod-failover \
    --server monuments-sqlserver-prod \
    --partner-server monuments-sqlserver-secondary \
    --failover-policy Automatic \
    --grace-period 1

# Create Key Vault for secrets
az keyvault create \
    --name monuments-kv-prod \
    --resource-group monuments-prod \
    --enable-purge-protection

# Store secrets
az keyvault secret set \
    --vault-name monuments-kv-prod \
    --name ConnectionString \
    --value "Server=tcp:monuments-sqlserver-prod.database.windows.net,1433;Initial Catalog=MonumentsBookingProd;User ID=sqladmin;Password=$DB_PASSWORD;Encrypt=True;"

az keyvault secret set \
    --vault-name monuments-kv-prod \
    --name ApiJwtSecret \
    --value "$(openssl rand -base64 32)"

# Configure App Service to access Key Vault
az webapp identity assign \
    --name monuments-api-prod \
    --resource-group monuments-prod

# Grant App Service access to Key Vault
PRINCIPAL_ID=$(az webapp identity show --name monuments-api-prod --resource-group monuments-prod --query principalId -o tsv)
az keyvault set-policy \
    --name monuments-kv-prod \
    --object-id $PRINCIPAL_ID \
    --secret-permissions get list

# Deploy application
az webapp deployment source config-zip \
    --name monuments-api-prod \
    --resource-group monuments-prod \
    --src ./api.zip

# Enable Application Insights
az monitor app-insights component create \
    --app monuments-insights-prod \
    --location eastus \
    --resource-group monuments-prod \
    --application-type web

# Link to App Service
az webapp config set \
    --name monuments-api-prod \
    --resource-group monuments-prod \
    --app-insights-key $(az monitor app-insights component show --app monuments-insights-prod --resource-group monuments-prod --query instrumentationKey -o tsv)
```

#### On-Premises IIS Deployment

```powershell
# On Windows Server with IIS installed

# Install ASP.NET Core Runtime
# https://dotnet.microsoft.com/download/dotnet

# Create IIS Application Pool
New-IISAppPool -Name "MonumentsAPI" -ManagedRuntimeVersion "v4.0" -ProcessModel "Integrated"

# Create IIS Website
New-IISSite -Name "monuments-api" `
    -PhysicalPath "C:\inetpub\monuments-api" `
    -BindingInformation "*:443:api.monuments.local" `
    -Protocol https

# Add SSL Certificate
# Import PFX certificate or create self-signed cert for testing
New-SelfSignedCertificate -DnsName "api.monuments.local" -CertStoreLocation "Cert:\LocalMachine\My"

# Publish application
dotnet publish -c Release -o C:\inetpub\monuments-api

# Configure app pool to run as app service account
Set-IISAppPoolConfig -Name "MonumentsAPI" -ProcessModel @{
    IdentityType = 3  # ApplicationPoolIdentity
    UserName = "DOMAIN\appservice_account"
    Password = ConvertTo-SecureString "AppServicePassword@123" -AsPlainText -Force
}

# Set permissions on published folder
$acl = Get-Acl "C:\inetpub\monuments-api"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule("DOMAIN\appservice_account", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.SetAccessRule($rule)
Set-Acl "C:\inetpub\monuments-api" $acl

# Configure web.config for ASP.NET Core
# (typically auto-generated during publish)

# Restart IIS
Restart-WebAppPool -Name "MonumentsAPI"
```

---

## Infrastructure Architecture

### Development Architecture

```
┌─────────────────────────────────────────────┐
│         Developer Machine                   │
├─────────────────────────────────────────────┤
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │  Visual Studio / VS Code             │  │
│  │  - MonumentsAPI.csproj               │  │
│  │  - MonumentsAdmin.csproj             │  │
│  │  - MonumentsPublic.csproj            │  │
│  └────────────┬─────────────────────────┘  │
│               │                            │
│  ┌────────────▼─────────────────────────┐  │
│  │  dotnet run / IIS Express            │  │
│  │  Port 5000, 5001, 5002               │  │
│  └────────────┬─────────────────────────┘  │
│               │                            │
│  ┌────────────▼─────────────────────────┐  │
│  │  SQL Server (Local)                  │  │
│  │  MonumentsBooking database           │  │
│  │  Port 1433                           │  │
│  └──────────────────────────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

### Production Architecture (Azure)

```
┌─────────────────────────────────────────────────────────────┐
│                     Azure Cloud                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Azure Front Door / CDN (DDoS Protection)              │ │
│  └────────────────────┬────────────────────────────────┘ │
│                       │                                   │
│  ┌────────────────────▼────────────────────────────────┐ │
│  │  Azure App Service (Premium Plan)                   │ │
│  │  - 3+ instances (auto-scaling)                       │ │
│  │  - SSL/TLS termination                              │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │ ┌──────────────────────────────────────────────┐   │ │
│  │ │ API (monuments-api-prod)                     │   │ │
│  │ │ Admin MVC (monuments-admin-prod)             │   │ │
│  │ │ Public MVC (monuments-public-prod)           │   │ │
│  │ └──────────────────────────────────────────────┘   │ │
│  └────────────────────┬────────────────────────────────┘ │
│                       │                                   │
│  ┌────────────────────▼────────────────────────────────┐ │
│  │  Azure SQL Database (Premium, Geo-Redundant)        │ │
│  │  - Primary: East US                                 │ │
│  │  - Failover: West US                               │ │
│  │  - Automatic backups (35 days)                      │ │
│  │  - Point-in-time restore enabled                   │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌─────────────────────┐    ┌──────────────────────────┐  │
│  │ Azure Storage       │    │ Azure Key Vault          │  │
│  │ - Monument images   │    │ - Connection strings     │  │
│  │ - Ticket exports    │    │ - API secrets            │  │
│  │ - Backups (GRS)     │    │ - Certificates           │  │
│  └─────────────────────┘    └──────────────────────────┘  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ Azure Monitor / Application Insights                │  │
│  │ - Performance monitoring                            │  │
│  │ - Error tracking                                    │  │
│  │ - Custom metrics                                    │  │
│  │ - Alerting and notifications                        │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### On-Premises Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Load Balancer (F5/Citrix)                  │
│              - SSL/TLS Termination                      │
│              - Session Persistence                      │
│              - Health Checks                            │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────┼────────────┐
        │            │            │
┌───────▼──┐   ┌────▼────┐   ┌──▼────────┐
│   IIS    │   │   IIS   │   │   IIS    │
│  Web 1   │   │  Web 2  │   │  Web 3   │
└───────┬──┘   └────┬────┘   └──┬───────┘
        │           │           │
        └───────────┼───────────┘
                    │
        ┌───────────▼──────────┐
        │  SQL Server Cluster   │
        │  - Primary Instance   │
        │  - Secondary Instance │
        │  - Always On AG       │
        │  - Shared Storage     │
        └──────────────────────┘

Network:
- Internal DNS for service discovery
- Active Directory for authentication
- Backup storage (separate SAN/NAS)
```

---

## Configuration Management

### Environment Variables (appsettings.json)

#### Development

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=.;Database=MonumentsBooking;Integrated Security=true;Encrypt=false;"
  },
  "Jwt": {
    "Key": "dev-secret-key-not-for-production",
    "Issuer": "monuments-dev",
    "Audience": "monuments-api",
    "ExpirationMinutes": 60
  },
  "Booking": {
    "ReservationHoldExpiryMinutes": 15,
    "BookingCutoffMinutes": 30
  },
  "Email": {
    "Provider": "SendGrid",
    "ApiKey": "SG.test-key-for-dev"
  }
}
```

#### Staging

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=tcp:monuments-sqlserver.database.windows.net,1433;Initial Catalog=MonumentsBookingStaging;User ID=sqladmin;Password=#{DatabasePassword}#;Encrypt=True;"
  },
  "Jwt": {
    "Key": "#{JwtSecretKey}#",
    "Issuer": "monuments-staging",
    "Audience": "monuments-api",
    "ExpirationMinutes": 120
  },
  "Cors": {
    "AllowedOrigins": ["https://admin-staging.monuments.local", "https://public-staging.monuments.local"]
  },
  "ApplicationInsights": {
    "InstrumentationKey": "#{AppInsightsKey}#"
  }
}
```

#### Production

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Error"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "@Microsoft.KeyVault(SecretUri=https://monuments-kv-prod.vault.azure.net/secrets/ConnectionString/)"
  },
  "Jwt": {
    "Key": "@Microsoft.KeyVault(SecretUri=https://monuments-kv-prod.vault.azure.net/secrets/JwtSecret/)",
    "Issuer": "monuments-prod",
    "Audience": "monuments-api",
    "ExpirationMinutes": 60
  },
  "Cors": {
    "AllowedOrigins": ["https://admin.monuments.com", "https://monuments.com"]
  },
  "RateLimit": {
    "PublicEndpointsPerMinute": 100,
    "AuthenticatedEndpointsPerMinute": 1000
  },
  "ApplicationInsights": {
    "InstrumentationKey": "@Microsoft.KeyVault(SecretUri=https://monuments-kv-prod.vault.azure.net/secrets/AppInsightsKey/)"
  },
  "Security": {
    "RequireHttps": true,
    "HstsMaxAgeSeconds": 31536000,
    "HstsIncludeSubdomains": true,
    "HstsPreload": true
  }
}
```

### Secret Management (Azure Key Vault)

```bash
# Store secrets in Azure Key Vault
az keyvault secret set \
    --vault-name monuments-kv-prod \
    --name DatabasePassword \
    --value "SecureDBPassword@123"

az keyvault secret set \
    --vault-name monuments-kv-prod \
    --name JwtSecret \
    --value "$(openssl rand -base64 32)"

az keyvault secret set \
    --vault-name monuments-kv-prod \
    --name SendGridApiKey \
    --value "SG.xxxxxxxx"

# Reference in appsettings.json using Key Vault references
# @Microsoft.KeyVault(SecretUri=https://monuments-kv-prod.vault.azure.net/secrets/SecretName/)

# Application code accesses via Azure Identity SDK (automatic authentication)
```

---

## Security Setup

### SSL/TLS Configuration

#### Self-Signed Certificate (Development)

```bash
# Generate self-signed certificate
dotnet dev-certs https --clean
dotnet dev-certs https --trust

# Or using OpenSSL
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

#### Azure App Service (Production)

```bash
# Managed certificate (automatic renewal)
az webapp config ssl bind \
    --name monuments-api-prod \
    --certificate-thumbprint $(az webapp config ssl list \
        --name monuments-api-prod \
        --query '[0].thumbprint' -o tsv)

# Or upload custom certificate
az webapp config ssl upload \
    --name monuments-api-prod \
    --certificate-file ./certificate.pfx \
    --certificate-password "CertificatePassword"

# Bind certificate
az webapp config ssl bind \
    --name monuments-api-prod \
    --certificate-thumbprint "THUMBPRINT"
```

#### HTTPS Enforcement

```csharp
// In Program.cs
var builder = WebApplication.CreateBuilder(args);

// Enforce HTTPS redirect
if (!builder.Environment.IsDevelopment())
{
    builder.Services.AddHttpsRedirection(options =>
    {
        options.HttpsPort = 443;
        options.RedirectStatusCode = StatusCodes.Status301MovedPermanently;
    });

    // Add HSTS (HTTP Strict Transport Security)
    builder.Services.AddHsts(options =>
    {
        options.MaxAge = TimeSpan.FromDays(365);
        options.IncludeSubDomains = true;
        options.Preload = true;
    });
}

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHttpsRedirection();
    app.UseHsts();
}
```

### Authentication & Authorization

```csharp
// Configure JWT authentication
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        
        ValidateAudience = true,
        ValidAudience = builder.Configuration["Jwt:Audience"],
        
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"])),
        
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero
    };
});

// Configure CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowMonumentsUI", policy =>
    {
        var allowedOrigins = builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>();
        policy
            .WithOrigins(allowedOrigins)
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

var app = builder.Build();
app.UseCors("AllowMonumentsUI");
app.UseAuthentication();
app.UseAuthorization();
```

### SQL Injection & Input Validation Prevention

```csharp
// ✅ Good: Parameterized queries with EF Core
var bookings = await dbContext.Bookings
    .Where(b => b.MonumentId == monumentId) // EF Core parameterizes
    .ToListAsync();

// ❌ Bad: String interpolation (NEVER DO THIS!)
// var query = $"SELECT * FROM Bookings WHERE MonumentId = '{monumentId}'"; // SQL Injection!

// Input validation in DTOs
[ApiController]
[Route("api/[controller]")]
public class BookingsController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> CreateBooking([FromBody] CreateBookingDto dto)
    {
        // ModelState validation automatically applied
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        // Model binding validates [StringLength], [Range], [Required] attributes
        // ...
    }
}

// DTO with validations
public class CreateBookingDto
{
    [Required]
    [StringLength(100)]
    public string VisitorName { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Range(1, 10)]
    public int TotalVisitors { get; set; }
}
```

---

## Monitoring & Logging

### Application Insights Configuration

```csharp
// Configure Application Insights
builder.Services.AddApplicationInsightsTelemetry();

builder.Services.AddApplicationInsightsKusto(options =>
{
    options.InstrumentationKey = builder.Configuration["ApplicationInsights:InstrumentationKey"];
});

// Log custom metrics
var telemetry = new TelemetryClient();

// Track booking created event
telemetry.TrackEvent("BookingCreated", new Dictionary<string, string>
{
    { "MonumentId", booking.MonumentId },
    { "BookingChannel", booking.BookingChannel }
}, new Dictionary<string, double>
{
    { "TotalPrice", (double)booking.TotalPrice }
});

// Track dependencies (API calls, database)
using (telemetry.StartOperation<DependencyTelemetry>("API_Call"))
{
    var response = await httpClient.GetAsync("https://external-api.com");
}
```

### Structured Logging (Serilog)

```csharp
// Configure Serilog
builder.Host.UseSerilog((context, loggerConfig) =>
{
    loggerConfig
        .MinimumLevel.Information()
        .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
        .WriteTo.File("logs/monuments-.txt", rollingInterval: RollingInterval.Day)
        .WriteTo.ApplicationInsights(new TelemetryClient(), TelemetryConverter.Traces)
        .Enrich.FromLogContext()
        .Enrich.WithProperty("Environment", context.HostingEnvironment.EnvironmentName);
});

// Usage in code
public class BookingService
{
    private readonly ILogger<BookingService> _logger;

    public BookingService(ILogger<BookingService> logger)
    {
        _logger = logger;
    }

    public async Task<Booking> CreateBookingAsync(CreateBookingDto dto)
    {
        _logger.LogInformation("Creating booking for monument {MonumentId}, visitor {VisitorName}", 
            dto.MonumentId, dto.VisitorName);

        try
        {
            // ... business logic
            _logger.LogInformation("Booking {BookingReference} created successfully", booking.Reference);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating booking for monument {MonumentId}", dto.MonumentId);
            throw;
        }
    }
}
```

### Alerts & Notifications

```bash
# Set up alerts in Azure Monitor
az monitor metrics alert create \
    --name monuments-api-errors \
    --resource-group monuments-prod \
    --scopes /subscriptions/{subscription-id}/resourceGroups/monuments-prod/providers/Microsoft.Insights/components/monuments-insights-prod \
    --condition "avg 'Http4xx' > 50" \
    --description "Alert when error rate exceeds 50 per minute" \
    --evaluation-frequency 1m \
    --window-size 5m \
    --action create \
    --webhook-receiver monuments-ops \
    --webhook-service-uri "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

---

## Backup & Disaster Recovery

### Backup Strategy

| Type | Frequency | Retention | Method |
|---|---|---|---|
| Full Backup | Daily 2 AM UTC | 35 days (Azure SQL) or 30 days | Automatic (Azure) or SQL Agent Job |
| Transaction Log | Every 15 minutes | 7 days | Automatic (Azure) or SQL Agent Job |
| Blob Storage | Daily | 365 days (GRS) | Azure Storage LRS → GRS replication |

### Azure SQL Backup (Automatic)

```bash
# Azure SQL automatically maintains backups
# View retention policy
az sql db show \
    --name MonumentsBookingProd \
    --server monuments-sqlserver-prod \
    --resource-group monuments-prod \
    --query 'backupShortTermRetentionPolicy'

# Extend short-term retention (default 7 days)
az sql db short-term-retention-policy set \
    --name MonumentsBookingProd \
    --server monuments-sqlserver-prod \
    --resource-group monuments-prod \
    --retention-days 35

# Backup to long-term storage (years)
az sql db ltr-backup set \
    --name MonumentsBookingProd \
    --server monuments-sqlserver-prod \
    --resource-group monuments-prod \
    --monthly-retention P5Y \
    --yearly-retention P10Y
```

### Manual Backup (SQL Server)

```sql
-- Full backup to disk
BACKUP DATABASE MonumentsBooking 
TO DISK = 'C:\SQLBackups\monuments_full_2026-10-05.bak'
WITH INIT, COMPRESSION, STATS = 10;

-- Backup to Azure Storage (Hybrid approach)
BACKUP DATABASE MonumentsBooking 
TO URL = 'https://monuments-backup.blob.core.windows.net/backups/monuments_full_2026-10-05.bak'
WITH SAS CREDENTIAL = 'shared-access-signature-credential', INIT, COMPRESSION;

-- Transaction log backup
BACKUP LOG MonumentsBooking
TO DISK = 'C:\SQLBackups\monuments_tlog_2026-10-05_1400.trn'
WITH INIT, COMPRESSION;
```

### Point-in-Time Recovery

```bash
# List available backups (Azure SQL)
az sql db ltr-backup list \
    --name MonumentsBookingProd \
    --server monuments-sqlserver-prod \
    --resource-group monuments-prod

# Restore to point-in-time (Azure SQL)
az sql db restore \
    --name MonumentsBookingProd-Restored \
    --server monuments-sqlserver-prod \
    --resource-group monuments-prod \
    --source-server monuments-sqlserver-prod \
    --source-name MonumentsBookingProd \
    --restore-point-in-time "2026-10-05T12:00:00Z"
```

### Disaster Recovery Plan

**Recovery Time Objective (RTO):** 4 hours  
**Recovery Point Objective (RPO):** 15 minutes

**Steps:**
1. Detect outage (alerting via Application Insights)
2. Activate failover (automatic in Azure SQL with failover groups)
3. Validate data integrity (check last successful backup)
4. Redirect traffic to secondary region
5. Run post-recovery tests
6. Document incident and update procedures

---

## Scaling & Performance

### Auto-Scaling Configuration (Azure)

```bash
# Enable auto-scaling for App Service
az appservice plan update \
    --name monuments-prod-plan \
    --resource-group monuments-prod \
    --enable-autoscale

# Create auto-scale rules
az monitor autoscale create \
    --name monuments-api-autoscale \
    --resource-group monuments-prod \
    --resource monuments-prod-plan \
    --resource-type "Microsoft.Web/serverfarms" \
    --min-count 3 \
    --max-count 10 \
    --count 3

# Scale up rule (CPU > 70%)
az monitor autoscale rule create \
    --autoscale-name monuments-api-autoscale \
    --resource-group monuments-prod \
    --metric-name CpuPercentage \
    --metric-resource-target-value 70 \
    --scale-type ChangeCount \
    --scale-value 1

# Scale down rule (CPU < 30%)
az monitor autoscale rule create \
    --autoscale-name monuments-api-autoscale \
    --resource-group monuments-prod \
    --metric-name CpuPercentage \
    --metric-resource-target-value 30 \
    --scale-type ChangeCount \
    --scale-value -1 \
    --cooldown-time 5m
```

### Database Performance Tuning

```sql
-- Check query performance
SELECT name, execution_count, total_elapsed_time / 1000000 as total_sec, 
       total_elapsed_time / execution_count / 1000 as avg_ms
FROM sys.dm_exec_procedure_stats
ORDER BY total_elapsed_time DESC;

-- Identify missing indexes
SELECT * FROM sys.dm_db_missing_index_details
ORDER BY equality_columns DESC;

-- Update statistics
UPDATE STATISTICS Bookings;
UPDATE STATISTICS SlotInstances;

-- Rebuild fragmented indexes
ALTER INDEX idx_Bookings_MonumentVisitDate ON Bookings REBUILD;
```

### Caching Strategy

```csharp
// Implement MemoryCache for listings
public class AvailabilityService
{
    private readonly IMemoryCache _cache;
    private readonly MonumentsDbContext _dbContext;

    public AvailabilityService(IMemoryCache cache, MonumentsDbContext dbContext)
    {
        _cache = cache;
        _dbContext = dbContext;
    }

    public async Task<AvailabilityDto> GetAvailabilityAsync(string monumentId, int days)
    {
        string cacheKey = $"availability_{monumentId}_{days}";

        if (!_cache.TryGetValue(cacheKey, out AvailabilityDto availability))
        {
            availability = await _dbContext.SlotInstances
                .Where(s => s.MonumentId == monumentId && s.Date >= DateTime.Today)
                .GroupBy(s => s.Date)
                .Select(g => new AvailabilityDto { /* ... */ })
                .ToListAsync();

            _cache.Set(cacheKey, availability, TimeSpan.FromHours(1));
        }

        return availability;
    }

    // Invalidate cache when bookings change
    public async Task InvalidateAvailabilityCacheAsync(string monumentId)
    {
        _cache.Remove($"availability_{monumentId}_30");
        _cache.Remove($"availability_{monumentId}_60");
        _cache.Remove($"availability_{monumentId}_90");
    }
}
```

---

## Troubleshooting

### Common Issues & Solutions

| Issue | Cause | Solution |
|---|---|---|
| "Connection timeout" | Network issue or database offline | Check SQL Server is running; verify firewall rules; test connectivity with `sqlcmd` |
| "Authentication failed" | Wrong credentials or user permissions | Verify SQL login; check user roles in `sys.database_principals` |
| "Cannot insert duplicate booking reference" | Unique constraint violation | Check for transaction rollback leaving orphan records; run integrity check |
| "Sold out error even with capacity" | Race condition with concurrent bookings | Ensure transaction isolation level is SERIALIZABLE for slot updates |
| "QR token not recognized" | Token revoked or expired | Verify ticket status in Tickets table; check expiration timestamp |

### Debugging Commands

```bash
# Check SQL Server connectivity
sqlcmd -S tcp:monuments-sqlserver.database.windows.net,1433 -U sqladmin -d MonumentsBooking -Q "SELECT @@VERSION;"

# View application logs (local)
Get-Content "logs/monuments-20261005.txt" -Tail 100

# Check Azure App Service logs
az webapp log tail --name monuments-api-prod --resource-group monuments-prod

# Retrieve diagnostics from Key Vault
az keyvault secret show --vault-name monuments-kv-prod --name ConnectionString --query value -o tsv

# Test API endpoint
curl -X GET https://api.monuments.com/api/monuments/public \
  -H "Authorization: Bearer $TOKEN" \
  -v
```

---

**Deployment Guide Version:** 1.0  
**Last Updated:** October 5, 2026  
**Database Platform:** SQL Server 2019 or later  
**Target Deployment:** Azure App Service + Azure SQL Database (Primary), On-Premises IIS + SQL Server (Secondary)