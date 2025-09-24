## Pinterest-Blumenau Snowflake Storage Integration: Step-by-Step Process

### **Step 1: secopsteam0 Creates AWS Infrastructure (terraform-control-repo)**

**What secopsteam0 provides:**

1. **AWS IAM Role** (`SnowflakeStorageIntegration_pinterest-blumenau`)
   - **Trust Policy**: Allows Snowflake's IAM user to assume this role
   - **Principal**: `arn:aws:iam::343182919852:user/oqyl-s-v2st0658` (Snowflake's user)`
   - **External ID**: `PINTERESTIT_SFCRole=4_XSO0HbFFydOsgyU3TT9gwnsWA1w=`

2. **AWS IAM Policy** (attached to the role)
   - **S3 Permissions**: GetObject, PutObject, DeleteObject, ListBucket
   - **S3 Resources**: `arn:aws:s3:::pinterest-blumenau` and `arn:aws:s3:::pinterest-blumenau/*`

3. **S3 Bucket Policy** (on `pinterest-blumenau` bucket)
   - **Cross-account access** for Pinterest's main account (`998131032990`)
   - **Security**: SSL-only, no public access

### **Step 2: snowflaketeam Creates Snowflake Storage Integration**

**What snowflaketeam provides:**

1. **Storage Integration** (`S3_PINTEREST_PROD_BLUMENAU`)
   ```sql
   CREATE STORAGE INTEGRATION S3_PINTEREST_PROD_BLUMENAU
       TYPE=EXTERNAL_STAGE
       STORAGE_PROVIDER='S3'
       STORAGE_AWS_ROLE_ARN='arn:aws:iam::621763355519:role/SnowflakeStorageIntegration_pinterest-blumenau'
       ENABLED=true
       STORAGE_ALLOWED_LOCATIONS=('s3://pinterest-blumenau/');
   ```

2. **Snowflake Stages** (multiple file formats)
   - `PINTEREST_BLUMENAU_STAGE` (TSV)
   - `PINTEREST_BLUMENAU_STAGE_CSV` (CSV)
   - `PINTEREST_BLUMENAU_STAGE_PARQUET` (Parquet)
   - etc.

3. **Snowflake Role** (`S3_PINTEREST_PROD_BLUMENAU_AGENT_ROLE`)
   - Permissions to use the stages
   - Access to warehouse, database, schema

### **Step 3: The Complete Flow**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           PINTEREST-BLUMENAU FLOW                              │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   secopsteam0   │    │  snowflaketeam  │    │    Snowflake    │    │   AWS S3       │
│   (terraform)   │    │   (SQL DDLs)    │    │   (Runtime)     │    │  (Storage)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │                       │
         │ 1. Creates AWS        │                       │                       │
         │    IAM Role           │                       │                       │
         ├──────────────────────►│                       │                       │
         │                       │                       │                       │
         │                       │ 2. Creates Storage   │                       │
         │                       │    Integration        │                       │
         │                       ├──────────────────────►│                       │
         │                       │                       │                       │
         │                       │                       │ 3. Assumes AWS Role   │
         │                       │                       ├──────────────────────►│
         │                       │                       │                       │
         │                       │                       │ 4. COPY INTO command  │
         │                       │                       ├──────────────────────►│
         │                       │                       │                       │
         │                       │                       │ 5. Reads/Writes S3    │
         │                       │                       │    files              │
         │                       │                       │◄──────────────────────┤
```

### **Step 4: How COPY INTO Works**

When you run:
```sql
COPY INTO @pinterest-blumenau-stage/myfile.csv
FROM (SELECT * FROM my_table)
FILE_FORMAT = (TYPE = 'CSV');
```

**The process:**

1. **Snowflake** uses the `S3_PINTEREST_PROD_BLUMENAU` storage integration
2. **Snowflake** assumes the AWS IAM role `SnowflakeStorageIntegration_pinterest-blumenau`
3. **Snowflake** uses the assumed role credentials to access `s3://pinterest-blumenau/`
4. **Snowflake** performs the COPY operation (read/write files)

### **Step 5: Security & Trust Chain**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              SECURITY FLOW                                     │
└─────────────────────────────────────────────────────────────────────────────────┘

AWS Account 621763355519 (Partner Data Sharing)
├── IAM Role: SnowflakeStorageIntegration_pinterest-blumenau
│   ├── Trust Policy: Allows Snowflake IAM user
│   ├── External ID: PINTERESTIT_SFCRole=4_XSO0HbFFydOsgyU3TT9gwnsWA1w=
│   └── Permissions: S3 access to pinterest-blumenau bucket
│
Snowflake Account 343182919852
├── IAM User: oqyl-s-v2st0658
│   └── Can assume Pinterest's IAM role with correct External ID
│
Snowflake Database
├── Storage Integration: S3_PINTEREST_PROD_BLUMENAU
│   └── References AWS IAM role ARN
├── Stages: @pinterest-blumenau-stage
│   └── Uses storage integration
└── COPY INTO commands
    └── Uses stages to access S3
```

### **Step 6: What Each Team Provides**

| Team | Provides | Purpose |
|------|----------|---------|
| **secopsteam0** | AWS IAM Role + Policy<br/>S3 Bucket Policy<br/>Trust relationship | Allows Snowflake to access S3 |
| **snowflaketeam** | Storage Integration<br/>Snowflake Stages<br/>Snowflake Roles | Configures Snowflake to use AWS resources |

### **Step 7: Usage Example**

```sql
-- Use the stage created by snowflaketeam
COPY INTO @pinterest-blumenau-stage/data/my_table/
FROM (SELECT * FROM my_source_table)
FILE_FORMAT = (TYPE = 'PARQUET');

-- List files in the stage
LIST @pinterest-blumenau-stage/data/my_table/;
```


[1 tool called]

## Summary

**secopsteam0** created the AWS infrastructure that allows Snowflake to access the `pinterest-blumenau` S3 bucket through a secure IAM role assumption process.

**snowflaketeam** created the Snowflake storage integration and stages that use this AWS infrastructure.

**The result**: You can now use `COPY INTO @pinterest-blumenau-stage/` commands in Snowflake, which will automatically:
1. Assume the AWS IAM role created by secopsteam0
2. Access the S3 bucket with the proper permissions
3. Perform the data transfer operations

The key components are:
- **AWS IAM Role**: `SnowflakeStorageIntegration_pinterest-blumenau` (created by secopsteam0)
- **Snowflake Storage Integration**: `S3_PINTEREST_PROD_BLUMENAU` (created by snowflaketeam)
- **Snowflake Stage**: `@pinterest-blumenau-stage` (created by snowflaketeam)
