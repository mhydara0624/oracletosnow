# Oracle to Snowflake Migration with Spatial Data Support
*Complete Guide: RSA Authentication, WKT Conversion, and Snowflake Python Connector*

## 📚 Table of Contents
- [Overview](#-overview)
- [Architecture](#-architecture) 
- [Prerequisites](#-prerequisites)
- [RSA Key Setup Process](#-rsa-key-setup-process)
- [Environment Configuration](#-environment-configuration)
- [Snowflake Python Connector Testing](#-snowflake-python-connector-testing)
- [Oracle Spatial Data Conversion](#-oracle-spatial-data-conversion)
- [Complete Migration Process](#-complete-migration-process)
- [Troubleshooting](#-troubleshooting)
- [References](#-references)

---

## 🎯 Overview

This guide documents a complete Oracle to Snowflake data migration solution that handles spatial geometry data using Oracle's native `SDO_UTIL.TO_WKTGEOMETRY()` function and the Snowflake Python Connector with RSA key-pair authentication.

### Key Features
- ✅ **RSA Key-Pair Authentication** for Snowflake
- ✅ **Native Oracle Spatial Conversion** using `SDO_UTIL.TO_WKTGEOMETRY()`
- ✅ **Staged File Upload** using Snowflake's PUT command
- ✅ **Bulk Data Loading** using COPY INTO
- ✅ **WKT Geometry Support** in Snowflake
- ✅ **Production-Ready Error Handling**

---

## 🏗️ Architecture
```┌─────────────┐ ┌──────────────────┐ ┌─────────────────┐ ┌──────────────┐
│ Oracle │───▶│ Python Script │───▶│ Snowflake Stage │───▶│ Snowflake │
│ Database │ │ w/ Spatial Conv. │ │ (Files) │ │ Tables │
└─────────────┘ └──────────────────┘ └─────────────────┘ └──────────────┘
│ │ │ │
│ ┌────────▼────────┐ │ │
│ │ SDO_UTIL.TO_ │ │ │
│ │ WKTGEOMETRY() │ │ │
│ └─────────────────┘ │ │
│ │ │
└─── Geometry Data ────▶ WKT Format ────────────┘ │
│
RSA Authentication ────────────────────────────┘```


---

## 🔧 Prerequisites

### System Requirements
- **Python 3.8+** with pip
- **Oracle Database** with spatial data (`SDO_GEOMETRY` columns)
- **Snowflake Account** with appropriate permissions
- **Terminal/Command Line** access

### Required Python Packages
```bash
pip install snowflake-connector-python pandas python-dotenv cryptography oracledb
```

---

## 🔑 RSA Key Setup Process

### Step 1: Generate RSA Key Pair

```bash
# Create secure directory for keys
mkdir -p ~/.snowflake_keys
cd ~/.snowflake_keys

# Generate 2048-bit RSA private key
openssl genrsa -out snowflake_private_key.pem 2048

# Generate public key from private key
openssl rsa -in snowflake_private_key.pem -pubout -out snowflake_public_key.pem

# Set secure permissions
chmod 600 snowflake_private_key.pem  # Private key: read/write for owner only
chmod 644 snowflake_public_key.pem   # Public key: readable by others
```

### Step 2: Extract Public Key for Snowflake

```bash
# Get public key in Snowflake format (single line, no headers)
grep -v "BEGIN PUBLIC KEY\|END PUBLIC KEY" snowflake_public_key.pem | tr -d '\n'
```

**Copy the output** - this is your public key for Snowflake configuration.

### Step 3: Configure Snowflake User

```sql
-- Connect to Snowflake as ACCOUNTADMIN or user with appropriate privileges
USE ROLE ACCOUNTADMIN;

-- Set the public key for your user (replace YOUR_USER and YOUR_PUBLIC_KEY)
ALTER USER YOUR_USER SET RSA_PUBLIC_KEY='YOUR_PUBLIC_KEY_STRING_HERE';

-- Verify the key was set
DESC USER YOUR_USER;

-- Grant necessary privileges
GRANT USAGE ON WAREHOUSE YOUR_WAREHOUSE TO USER YOUR_USER;
GRANT USAGE ON DATABASE YOUR_DATABASE TO USER YOUR_USER;
GRANT USAGE ON SCHEMA YOUR_DATABASE.YOUR_SCHEMA TO USER YOUR_USER;
GRANT CREATE TABLE ON SCHEMA YOUR_DATABASE.YOUR_SCHEMA TO USER YOUR_USER;
GRANT CREATE STAGE ON SCHEMA YOUR_DATABASE.YOUR_SCHEMA TO USER YOUR_USER;
```

---

## ⚙️ Environment Configuration

### Step 1: Create Project Directory

```bash
# Create and navigate to project directory
mkdir oracle_snowflake_migration
cd oracle_snowflake_migration

# Create Python virtual environment
python3 -m venv snowflake_env
source snowflake_env/bin/activate  # Mac/Linux
# snowflake_env\Scripts\activate   # Windows

# Install required packages
pip install snowflake-connector-python pandas python-dotenv cryptography oracledb
```

### Step 2: Create Configuration File

```bash
cat > .env << 'EOF'
# Snowflake RSA Authentication Configuration
SNOWFLAKE_USER=your_snowflake_username
SNOWFLAKE_ACCOUNT=your_account_identifier
SNOWFLAKE_WAREHOUSE=your_warehouse_name
SNOWFLAKE_DATABASE=your_database_name
SNOWFLAKE_SCHEMA=your_schema_name
SNOWFLAKE_ROLE=your_role_name

# RSA Key Authentication
SNOWFLAKE_PRIVATE_KEY_PATH=/Users/yourusername/.snowflake_keys/snowflake_private_key.pem
SNOWFLAKE_PRIVATE_KEY_PASSPHRASE=

# Oracle Database Configuration
ORACLE_USER=your_oracle_username
ORACLE_PASSWORD=your_oracle_password
ORACLE_HOST=your_oracle_host
ORACLE_PORT=1521
ORACLE_SERVICE_NAME=your_service_name

# Migration Settings
BATCH_SIZE=50000
FILE_FORMAT=csv
STAGING_DIRECTORY=snowflake_staging
CLEANUP_FILES=true
GEOMETRY_TO_WKT=true
USE_SCHEMA_INFERENCE=true
EOF
```

**Important:** Update all placeholder values with your actual configuration.

---

## 🧪 Snowflake Python Connector Testing

### Step 1: Basic Connection Test

```python
#!/usr/bin/env python3
"""
Basic Snowflake connection test using RSA authentication
"""
import os
import snowflake.connector
from dotenv import load_dotenv
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.serialization import load_pem_private_key

load_dotenv()

def load_private_key():
    private_key_path = os.getenv('SNOWFLAKE_PRIVATE_KEY_PATH')
    with open(private_key_path, 'rb') as key_file:
        private_key = load_pem_private_key(key_file.read(), password=None)
    return private_key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )

def test_connection():
    try:
        conn = snowflake.connector.connect(
            user=os.getenv('SNOWFLAKE_USER'),
            account=os.getenv('SNOWFLAKE_ACCOUNT'),
            private_key=load_private_key(),
            warehouse=os.getenv('SNOWFLAKE_WAREHOUSE'),
            database=os.getenv('SNOWFLAKE_DATABASE'),
            schema=os.getenv('SNOWFLAKE_SCHEMA'),
            role=os.getenv('SNOWFLAKE_ROLE')
        )
        
        cursor = conn.cursor()
        cursor.execute("SELECT CURRENT_USER()")
        current_user = cursor.fetchone()[0]
        print(f"✅ Connected as: {current_user}")
        
        cursor.close()
        conn.close()
        return True
        
    except Exception as e:
        print(f"❌ Connection failed: {e}")
        return False

if __name__ == "__main__":
    test_connection()
```

### Step 2: Comprehensive Feature Testing

The complete test suite validates:

1. **Basic SQL Execution** - Confirms connector can run queries
2. **Stage Creation** - Tests internal stage creation for file uploads
3. **PUT Commands** - Validates file upload to Snowflake stages
4. **COPY INTO** - Tests bulk data loading from staged files
5. **Pandas Integration** - Validates `write_pandas` functionality
6. **WKT Geometry Handling** - Tests Oracle spatial data workflow

**Run the comprehensive tests:**
```bash
python test_snowflake_complete.py
```

**Expected Results:**
🚀 Starting Comprehensive Snowflake Python Connector Tests
🧪 Test 1: Basic Query Execution
✅ Test 1 PASSED: Basic queries working
🧪 Test 2: Creating Internal Stage
✅ Test 2 PASSED: Stage creation working
🧪 Test 3: Testing PUT Command
✅ Test 3 PASSED: PUT command working
🧪 Test 4: Testing COPY INTO Command
✅ Test 4 PASSED: COPY INTO working
🧪 Test 5: Testing Pandas Integration
✅ Test 5 PASSED: Pandas integration working
🧪 Test 6: Testing Oracle Pre-Converted WKT Data
✅ Test 6 PASSED: Oracle WKT pre-conversion simulation successful!
📊 TEST SUMMARY
==================================================
Total Tests: 6
Passed: 6
Failed: 0
Success Rate: 100.0%
🎉 ALL TESTS PASSED! Your Snowflake Python Connector is fully functional!
✅ Ready for Oracle to Snowflake migration with WKT support!


---

## 🗺️ Oracle Spatial Data Conversion

### The Oracle SDO_UTIL.TO_WKTGEOMETRY() Approach

Oracle provides the native [`SDO_UTIL.TO_WKTGEOMETRY()`](https://docs.oracle.com/database/121/SPATL/sdo_util-to_wktgeometry.htm#SPATL1251) function that converts `SDO_GEOMETRY` objects to Well-Known Text (WKT) format, which is:

- **OGC Compliant**: Follows Open Geospatial Consortium standards
- **ISO Standard**: Adheres to ISO 13249-3 specifications  
- **Snowflake Compatible**: WKT strings work perfectly in Snowflake VARCHAR columns
- **Production Ready**: Oracle-validated geometry conversion

### Oracle Query Examples

#### Basic Spatial Data Extraction
```sql
-- Extract spatial data with WKT conversion
SELECT 
    feature_id,
    feature_name,
    feature_type,
    SDO_UTIL.TO_WKTGEOMETRY(shape) AS shape_wkt,
    area_sqft,
    created_date
FROM spatial_features 
WHERE created_date >= DATE '2024-01-01'
```

#### Complex Spatial Query with Filtering
```sql
-- Extract large polygons with WKT conversion
SELECT 
    parcel_id,
    owner_name,
    SDO_UTIL.TO_WKTGEOMETRY(parcel_boundary) AS boundary_wkt,
    land_use_code,
    assessed_value
FROM land_parcels lp
WHERE SDO_GEOM.SDO_AREA(parcel_boundary, 0.005) > 10000  -- Area > 10,000 sq units
  AND land_use_code IN ('RESIDENTIAL', 'COMMERCIAL')
ORDER BY assessed_value DESC
```

#### Multi-Geometry Table Extraction
```sql
-- Handle multiple geometry columns
SELECT 
    building_id,
    building_name,
    SDO_UTIL.TO_WKTGEOMETRY(building_footprint) AS footprint_wkt,
    SDO_UTIL.TO_WKTGEOMETRY(property_boundary) AS boundary_wkt,
    height_feet,
    construction_year
FROM buildings 
WHERE construction_year >= 2020
```

### WKT Output Examples

The `SDO_UTIL.TO_WKTGEOMETRY()` function produces standard WKT strings:

```sql
-- Point geometry
POINT(-122.4194 37.7749)

-- LineString geometry  
LINESTRING(-122.4194 37.7749, -122.4094 37.7849, -122.3994 37.7949)

-- Polygon geometry
POLYGON((-122.4194 37.7749, -122.4094 37.7749, -122.4094 37.7849, -122.4194 37.7849, -122.4194 37.7749))

-- MultiPolygon geometry
MULTIPOLYGON(((-122.4194 37.7749, -122.4094 37.7749, -122.4094 37.7849, -122.4194 37.7849, -122.4194 37.7749)))
```

---

## 🔄 Complete Migration Process

### Process Flow Overview
Oracle Query (with WKT conversion)
↓
Python Data Extraction
↓
CSV File Generation
↓
Snowflake Stage Upload (PUT)
↓
Snowflake Table Loading (COPY INTO)
↓
Validation & Cleanup


### Step 1: Oracle Data Extraction

```python
import pandas as pd
import oracledb

def extract_oracle_spatial_data(connection_string, table_name, geometry_columns=None):
    """
    Extract Oracle spatial data with automatic WKT conversion
    """
    
    # Build query with WKT conversion for geometry columns
    if geometry_columns:
        # Get all column names first
        with oracledb.connect(connection_string) as conn:
            cursor = conn.cursor()
            cursor.execute(f"""
                SELECT column_name 
                FROM user_tab_columns 
                WHERE table_name = UPPER('{table_name}')
                ORDER BY column_id
            """)
            all_columns = [row[0] for row in cursor.fetchall()]
        
        # Build SELECT with WKT conversion
        select_parts = []
        for col in all_columns:
            if col.upper() in [g.upper() for g in geometry_columns]:
                select_parts.append(f"SDO_UTIL.TO_WKTGEOMETRY({col}) AS {col}_WKT")
            else:
                select_parts.append(col)
        
        query = f"SELECT {', '.join(select_parts)} FROM {table_name}"
    else:
        query = f"SELECT * FROM {table_name}"
    
    # Extract data
    df = pd.read_sql(query, oracledb.connect(connection_string))
    
    print(f"📊 Extracted {len(df)} rows from Oracle table: {table_name}")
    if geometry_columns:
        wkt_columns = [f"{col}_WKT" for col in geometry_columns]
        print(f"🗺️ Converted geometry columns to WKT: {wkt_columns}")
    
    return df
```

### Step 2: Snowflake Upload and Loading

```python
import snowflake.connector
from snowflake.connector.pandas_tools import write_pandas
import tempfile
import os

def migrate_to_snowflake(df, table_name, connection_params):
    """
    Migrate DataFrame to Snowflake using staging approach
    """
    
    # Connect to Snowflake
    conn = snowflake.connector.connect(**connection_params)
    cursor = conn.cursor()
    
    try:
        # Method 1: Direct pandas upload (for smaller datasets)
        if len(df) < 100000:  # Less than 100K rows
            success, nchunks, nrows, _ = write_pandas(
                conn=conn,
                df=df,
                table_name=table_name,
                auto_create_table=True,
                overwrite=True
            )
            print(f"✅ Direct upload: {nrows} rows loaded into {table_name}")
            
        # Method 2: Staged upload (for larger datasets)
        else:
            # Create internal stage
            stage_name = f"{table_name}_STAGE"
            cursor.execute(f"CREATE OR REPLACE STAGE {stage_name}")
            
            # Create temporary CSV file
            with tempfile.NamedTemporaryFile(mode='w', suffix='.csv', delete=False) as tmp_file:
                df.to_csv(tmp_file.name, index=False)
                temp_file_path = tmp_file.name
            
            # Upload file to stage
            put_command = f"PUT file://{temp_file_path} @{stage_name}"
            cursor.execute(put_command)
            print(f"📤 File uploaded to stage: {stage_name}")
            
            # Create table and load data
            # Auto-create table based on pandas dtypes
            write_pandas(
                conn=conn,
                df=df.head(0),  # Empty DataFrame to create schema
                table_name=table_name,
                auto_create_table=True,
                overwrite=True
            )
            
            # Load data from stage
            copy_command = f"""
            COPY INTO {table_name}
            FROM @{stage_name}
            FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1)
            """
            cursor.execute(copy_command)
            
            # Verify row count
            cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
            row_count = cursor.fetchone()[0]
            print(f"✅ Staged upload: {row_count} rows loaded into {table_name}")
            
            # Cleanup
            os.unlink(temp_file_path)
            cursor.execute(f"DROP STAGE {stage_name}")
        
        return True
        
    except Exception as e:
        print(f"❌ Migration failed: {e}")
        return False
        
    finally:
        cursor.close()
        conn.close()
```

### Step 3: Complete Migration Script

```python
#!/usr/bin/env python3
"""
Complete Oracle to Snowflake Migration with Spatial Data Support
"""

import os
from dotenv import load_dotenv
import oracledb
import snowflake.connector
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.serialization import load_pem_private_key

def main():
    load_dotenv()
    
    # Oracle connection
    oracle_dsn = f"{os.getenv('ORACLE_HOST')}:{os.getenv('ORACLE_PORT')}/{os.getenv('ORACLE_SERVICE_NAME')}"
    oracle_conn_string = f"{os.getenv('ORACLE_USER')}/{os.getenv('ORACLE_PASSWORD')}@{oracle_dsn}"
    
    # Snowflake connection parameters
    snowflake_params = {
        'user': os.getenv('SNOWFLAKE_USER'),
        'account': os.getenv('SNOWFLAKE_ACCOUNT'),
        'private_key': load_private_key(),
        'warehouse': os.getenv('SNOWFLAKE_WAREHOUSE'),
        'database': os.getenv('SNOWFLAKE_DATABASE'),
        'schema': os.getenv('SNOWFLAKE_SCHEMA'),
        'role': os.getenv('SNOWFLAKE_ROLE')
    }
    
    # Migration configuration
    migration_config = {
        'source_table': 'SPATIAL_FEATURES',
        'target_table': 'SPATIAL_FEATURES_MIGRATED',
        'geometry_columns': ['SHAPE', 'BOUNDARY']  # Columns containing SDO_GEOMETRY
    }
    
    print("🚀 Starting Oracle to Snowflake Migration...")
    
    # Step 1: Extract from Oracle
    df = extract_oracle_spatial_data(
        oracle_conn_string,
        migration_config['source_table'],
        migration_config['geometry_columns']
    )
    
    # Step 2: Migrate to Snowflake
    success = migrate_to_snowflake(
        df,
        migration_config['target_table'],
        snowflake_params
    )
    
    if success:
        print("🎉 Migration completed successfully!")
    else:
        print("❌ Migration failed!")

def load_private_key():
    """Load RSA private key for Snowflake authentication"""
    private_key_path = os.getenv('SNOWFLAKE_PRIVATE_KEY_PATH')
    with open(private_key_path, 'rb') as key_file:
        private_key = load_pem_private_key(key_file.read(), password=None)
    return private_key.private_bytes(
        encoding=serialization.Encoding.DER,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption()
    )

if __name__ == "__main__":
    main()
```

---

## 🛠️ Troubleshooting

### Common RSA Authentication Issues

**Problem**: `250001 (08001): Failed to connect to DB: ... certificate verify failed`
```bash
# Solution: Check private key format and path
openssl rsa -in snowflake_private_key.pem -text -noout
ls -la ~/.snowflake_keys/snowflake_private_key.pem
```

**Problem**: `250001 (08001): JWT token is invalid`
```bash
# Solution: Verify public key is set correctly in Snowflake
DESC USER your_username;
-- Check RSA_PUBLIC_KEY property
```

### Oracle Connection Issues

**Problem**: `ModuleNotFoundError: No module named 'oracledb'`
```bash
# Solution: Install Oracle client
pip install oracledb
```

**Problem**: `DPI-1047: Cannot locate a 64-bit Oracle Client library`
```bash
# Solution: Install Oracle Instant Client
# Download from: https://www.oracle.com/database/technologies/instant-client/downloads.html
```

### Geometry Conversion Issues

**Problem**: `ORA-00942: table or view does not exist (SDO_UTIL)`
```sql
-- Solution: Ensure Oracle Spatial is enabled
SELECT * FROM v$option WHERE parameter = 'Spatial';
-- Should return TRUE
```

**Problem**: `ORA-13030: Invalid dimension for this operation`
```sql
-- Solution: Validate geometry before conversion
SELECT SDO_GEOM.VALIDATE_GEOMETRY_WITH_CONTEXT(shape, 0.005) 
FROM your_table 
WHERE SDO_GEOM.VALIDATE_GEOMETRY_WITH_CONTEXT(shape, 0.005) != 'TRUE';
```

### Snowflake Loading Issues

**Problem**: Column name case sensitivity
```python
# Solution: Use quoted column names in Snowflake
cursor.execute('SELECT "column_name" FROM table_name')
```

**Problem**: WKT data truncation
```sql
-- Solution: Ensure adequate VARCHAR size
ALTER TABLE your_table MODIFY COLUMN geometry_wkt VARCHAR(16777216);
```

---

## 📖 References

### Oracle Documentation
- [SDO_UTIL.TO_WKTGEOMETRY Function](https://docs.oracle.com/database/121/SPATL/sdo_util-to_wktgeometry.htm#SPATL1251)
- [Oracle Spatial Developer's Guide](https://docs.oracle.com/database/121/SPATL/toc.htm)

### Snowflake Documentation  
- [Snowflake Python Connector](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector)
- [Key Pair Authentication](https://docs.snowflake.com/en/user-guide/key-pair-auth)
- [COPY INTO Command](https://docs.snowflake.com/en/sql-reference/sql/copy-into-table)

### Python Libraries
- [snowflake-connector-python](https://pypi.org/project/snowflake-connector-python/)
- [oracledb](https://pypi.org/project/oracledb/)
- [cryptography](https://pypi.org/project/cryptography/)

---

## 🏁 Conclusion

This migration approach successfully combines:
- **Oracle's native spatial capabilities** for reliable geometry conversion
- **Snowflake's Python connector** for robust data loading  
- **RSA authentication** for secure, programmatic access
- **Staged file processing** for handling large datasets
- **WKT format** for cross-platform spatial data compatibility

The result is a production-ready solution that efficiently migrates Oracle spatial data to Snowflake while maintaining data integrity and performance.

**Total Process Time**: RSA setup (15 min) + Testing (30 min) + Migration script (varies by data size)

**Ready for production Oracle to Snowflake spatial data migration!** 🚀



