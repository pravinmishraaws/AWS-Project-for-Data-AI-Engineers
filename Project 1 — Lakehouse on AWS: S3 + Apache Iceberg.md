## AWS Technical Architecture

<img width="914" height="514" alt="Screenshot 2025-10-04 at 21 24 24" src="https://github.com/user-attachments/assets/63d15a38-1435-4a7c-8ff3-c0a93f616167" />

## 1) Create S3 buckets

## Step 1) Buckets + Auto Run-Logging (all CLI)

> Works in **any region**. Replace `<yourname>` with something unique.
> Assumes AWS CLI v2 and your profile has admin (or equivalent) permissions.

### 1.0 Set variables

**Note:** Make sure to setup AWS CLI default region same as below. 

```bash
# ==== change these ====
export REGION=eu-north-1
export NAME=<yourname>

# ==== derived ====
export RAW_BUCKET=org-demo-lake-raw-$NAME
export SILVER_BUCKET=org-demo-lake-silver-$NAME
export TMP_BUCKET=org-demo-lake-tmp-$NAME

export DDB_TABLE=ingestion_runs
export LAMBDA_NAME=log-ingestion-run
export ROLE_NAME=lambda-log-ingestion-run-role
export POLICY_NAME=lambda-log-ingestion-run-policy
```

### 1.1 Create buckets + folders + encryption

```bash
aws s3 mb s3://$RAW_BUCKET    --region $REGION
aws s3 mb s3://$SILVER_BUCKET --region $REGION
aws s3 mb s3://$TMP_BUCKET    --region $REGION

# Block Public Access is ON by default for new buckets; double-enforce:
aws s3api put-public-access-block --bucket $RAW_BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-public-access-block --bucket $SILVER_BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-public-access-block --bucket $TMP_BUCKET \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Default encryption (SSE-S3/AES256)
aws s3api put-bucket-encryption --bucket $RAW_BUCKET \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws s3api put-bucket-encryption --bucket $SILVER_BUCKET \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws s3api put-bucket-encryption --bucket $TMP_BUCKET \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Folders (prefixes)
aws s3api put-object --bucket $RAW_BUCKET --key retail/orders/
aws s3api put-object --bucket $RAW_BUCKET --key retail/customers/
aws s3api put-object --bucket $RAW_BUCKET --key retail/products/
aws s3api put-object --bucket $TMP_BUCKET --key athena/
```

> (Optional but recommended) Enable versioning on **raw** for immutability:

```bash
aws s3api put-bucket-versioning --bucket $RAW_BUCKET --versioning-configuration Status=Enabled
```

###  1.2 DynamoDB table for run logs

```bash
aws dynamodb create-table \
  --table-name $DDB_TABLE \
  --attribute-definitions AttributeName=pk,AttributeType=S AttributeName=sk,AttributeType=S \
  --key-schema AttributeName=pk,KeyType=HASH AttributeName=sk,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST
```

* **pk:** `SRC#<source>` (e.g., `SRC#orders`)
* **sk:** `INGEST#<ISO8601>#<etag|noetag>`

### 1.3 IAM role + policy for Lambda

```bash
# Trust policy (Lambda can assume)
cat > trust.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}
  ]
}
JSON

aws iam create-role --role-name $ROLE_NAME --assume-role-policy-document file://trust.json

# Inline policy: DynamoDB PutItem + CloudWatch logs
cat > policy.json <<JSON
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":["dynamodb:PutItem"],
      "Resource":"arn:aws:dynamodb:$REGION:*:table/$DDB_TABLE"
    },
    {
      "Effect":"Allow",
      "Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],
      "Resource":"*"
    }
  ]
}
JSON

aws iam put-role-policy --role-name $ROLE_NAME --policy-name $POLICY_NAME --policy-document file://policy.json
```

### 1.4 Create Lambda (with code)

```bash
# Minimal, idempotent-ish handler
cat > handler.py <<'PY'
import os, hashlib, uuid, datetime
import boto3
from botocore.exceptions import ClientError

DDB_TABLE = os.environ.get("DDB_TABLE", "ingestion_runs")
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(DDB_TABLE)

def iso_utc():
    return datetime.datetime.utcnow().replace(microsecond=0).isoformat() + "Z"

def infer_source(key: str) -> str:
    # expects retail/<source>/...
    parts = key.split("/")
    return parts[1] if len(parts) > 1 else "unknown"

def make_run_id(source: str, ts_iso: str) -> str:
    short = uuid.uuid4().hex[:6].upper()
    return f"{source}_{ts_iso.replace(':','-')}_{short}"

def handler(event, context):
    for rec in event.get("Records", []):
        s3 = rec["s3"]
        bucket = s3["bucket"]["name"]
        key = s3["object"]["key"]
        size = s3["object"].get("size")
        etag = s3["object"].get("eTag", "").strip('"')
        version_id = s3["object"].get("versionId")
        src = infer_source(key)
        ts = iso_utc()
        run_id = make_run_id(src, ts)

        # idempotency: same bucket+key+etag
        dedupe = f"{bucket}#{key}#{etag or 'noetag'}"
        dedupe_hash = hashlib.sha256(dedupe.encode()).hexdigest()[:16]

        item = {
            "pk": f"SRC#{src}",
            "sk": f"INGEST#{ts}#{etag or 'noetag'}",
            "run_id": run_id,
            "source": src,
            "bucket": bucket,
            "key": key,
            "etag": etag,
            "version_id": version_id,
            "size_bytes": size,
            "ingest_ts": ts,
            "status": "LANDED",
            "dedupe": dedupe_hash
        }

        try:
            table.put_item(Item=item, ConditionExpression="attribute_not_exists(dedupe)")
        except ClientError as e:
            if e.response["Error"]["Code"] == "ConditionalCheckFailedException":
                # duplicate landing; ignore
                continue
            raise

    return {"ok": True}
PY

zip function.zip handler.py

aws lambda create-function \
  --function-name $LAMBDA_NAME \
  --runtime python3.12 \
  --handler handler.handler \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/$ROLE_NAME \
  --environment Variables="{DDB_TABLE=$DDB_TABLE}" \
  --timeout 30 \
  --memory-size 256 \
  --zip-file fileb://function.zip \
  --region $REGION
```

### 1.5 Allow S3 to invoke Lambda

```bash
aws lambda add-permission \
  --function-name $LAMBDA_NAME \
  --statement-id s3invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$RAW_BUCKET \
  --region $REGION
```

### 1.6 Configure S3 event notifications → Lambda

> Single rule on the `retail/` prefix (covers orders/customers/products).

```bash
cat > notif.json <<JSON
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "retail-object-created-to-lambda",
      "LambdaFunctionArn": "$(aws lambda get-function --function-name $LAMBDA_NAME --query 'Configuration.FunctionArn' --output text --region $REGION)",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": { "FilterRules": [ { "Name": "prefix", "Value": "retail/" } ] }
      }
    }
  ]
}
JSON

aws s3api put-bucket-notification-configuration \
  --bucket $RAW_BUCKET \
  --notification-configuration file://notif.json
```

---

### 1.7 Quick test

```bash
# upload a tiny test file under retail/orders/
echo "order_id,order_date,customer_id,product_id,qty,unit_price,channel" > /tmp/test_orders.csv
aws s3 cp /tmp/test_orders.csv s3://$RAW_BUCKET/retail/orders/test_orders.csv

# wait 3-5 seconds, then query DDB
aws dynamodb query \
  --table-name $DDB_TABLE \
  --key-condition-expression "pk = :p and begins_with(sk, :s)" \
  --expression-attribute-values '{":p":{"S":"SRC#orders"},":s":{"S":"INGEST#"}}' \
  --region $REGION
```

You should see an item with:

* `run_id`, `ingest_ts`, `bucket`, `key`, `size_bytes`, `status=LANDED`.

> Check Lambda logs if needed:

```bash
aws logs describe-log-streams \
  --log-group-name /aws/lambda/$LAMBDA_NAME \
  --order-by LastEventTime --descending --max-items 1 --region $REGION

# then:
aws logs get-log-events \
  --log-group-name /aws/lambda/$LAMBDA_NAME \
  --log-stream-name <the-stream-name-from-previous-output> \
  --limit 50 --region $REGION
```

---

### 1.8 (Optional) Narrow to per-source rules

If you prefer separate rules per area:

```bash
for p in retail/orders/ retail/customers/ retail/products/; do
  echo $p
done
# Create a JSON with three LambdaFunctionConfigurations and apply with put-bucket-notification-configuration.
```

---


## 2) Land (Immutable) — make it real & date-partitioned

### 2.1 Pick an ingest date (same for all three files)

```bash
export INGEST_DATE=$(date -u +%F)
```

### 2.2 Upload to **date-based** paths (immutable)

> Keep raw clean: no transforms, no renames post-landing.

```bash
aws s3 cp retail_data/products.csv  s3://$RAW_BUCKET/retail/products/ingest_date=$INGEST_DATE/products.csv
aws s3 cp retail_data/customers.csv s3://$RAW_BUCKET/retail/customers/ingest_date=$INGEST_DATE/customers.csv
aws s3 cp retail_data/orders.csv    s3://$RAW_BUCKET/retail/orders/ingest_date=$INGEST_DATE/orders.csv
```

Your Step 1 Lambda will still log the runs in DynamoDB (now with the date partition in the key). That’s your **landing lineage**.

---

## 3) Transformation & Data Enrichment

Transform happens when you **run the Glue job** below. You can:

* **Manual (on demand):** start it after each upload.
* **Automatic:** daily schedule (cron) or **event-driven** via EventBridge when a new `orders` object arrives.

### 3.1 Create an IAM role for Glue

```bash
export GLUE_ROLE_NAME=retail-glue-role
export GLUE_ROLE_ARN=$(aws iam create-role \
  --role-name $GLUE_ROLE_NAME \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"glue.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' --query Role.Arn --output text)

# Managed Glue service policy
aws iam attach-role-policy --role-name $GLUE_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole

# S3 + Glue Catalog access (minimal practical)
cat > glue-inline-policy.json <<JSON
{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["s3:GetObject","s3:ListBucket"],"Resource":[
      "arn:aws:s3:::$RAW_BUCKET","arn:aws:s3:::$RAW_BUCKET/*"
    ]},
    {"Effect":"Allow","Action":["s3:PutObject","s3:ListBucket"],"Resource":[
      "arn:aws:s3:::$SILVER_BUCKET","arn:aws:s3:::$SILVER_BUCKET/*"
    ]},
    {"Effect":"Allow","Action":[
      "glue:GetDatabase","glue:GetDatabases","glue:CreateDatabase",
      "glue:GetTable","glue:CreateTable","glue:UpdateTable",
      "glue:GetUserDefinedFunction","glue:CreateUserDefinedFunction"
    ],"Resource":"*"}
  ]
}
JSON
aws iam put-role-policy --role-name $GLUE_ROLE_NAME --policy-name retail-glue-inline --policy-document file://glue-inline-policy.json
```

### 3.2 Create Glue databases (your Step 2, CLI only)


```bash
aws glue create-database --database-input Name=retail_bronze || true
aws glue create-database --database-input Name=retail_silver || true
```

### 3.3 Create Athena workgroup (your Step 3, CLI only)

```bash
export ATHENA_WG=retail-wg

aws athena create-work-group \
  --name "$ATHENA_WG" \
  --description "Athena v3 workgroup for retail lakehouse" \
  --configuration "{
    \"EnforceWorkGroupConfiguration\": true,
    \"BytesScannedCutoffPerQuery\": 5368709120,
    \"ResultConfiguration\": {
      \"OutputLocation\": \"s3://$TMP_BUCKET/athena/\"
    },
    \"EngineVersion\": {
      \"SelectedEngineVersion\": \"Athena engine version 3\"
    }
  }" || true

```

Verify it worked

```bash
aws athena get-work-group --work-group "$ATHENA_WG" \
  --query 'WorkGroup.{Name:Name,Engine:Configuration.EngineVersion.SelectedEngineVersion,Output:Configuration.ResultConfiguration.OutputLocation,BytesLimit:Configuration.BytesScannedCutoffPerQuery}'

```

(Ensure Athena engine v3 in Console once; WG defaults to latest engine in most regions.)

### 3.4 Glue Spark job (Iceberg) — script + job creation

#### 3.4.1 Save the PySpark script locally

```bash
# retail_bronze_to_silver.py (v7)
import sys
from awsglue.utils import getResolvedOptions
from pyspark.sql import SparkSession, functions as F

# ------------- JOB ARGS -------------
args = getResolvedOptions(sys.argv, ['RAW_BUCKET', 'SILVER_DB', 'SILVER_WAREHOUSE'])
RAW_BUCKET       = args['RAW_BUCKET']
SILVER_DB        = args['SILVER_DB'] or 'retail_silver'
SILVER_WAREHOUSE = args['SILVER_WAREHOUSE']  # e.g. s3://org-demo-lake-silver-<name>/

print("SCRIPT_VERSION = v7")
print("RAW_BUCKET       =", RAW_BUCKET)
print("SILVER_DB        =", SILVER_DB)
print("SILVER_WAREHOUSE =", SILVER_WAREHOUSE)

# ------------- SPARK SESSION -------------
spark = (
    SparkSession.builder
    .appName("retail_bronze_to_silver")
    .config("spark.sql.sources.partitionOverwriteMode", "dynamic")
    # Iceberg + Glue Catalog wiring
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
    .config("spark.sql.catalog.glue_catalog", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.glue_catalog.warehouse", SILVER_WAREHOUSE)
    .config("spark.sql.catalog.glue_catalog.catalog-impl", "org.apache.iceberg.aws.glue.GlueCatalog")
    .config("spark.sql.catalog.glue_catalog.io-impl", "org.apache.iceberg.aws.s3.S3FileIO")
    .config("spark.sql.catalog.spark_catalog", "org.apache.iceberg.spark.SparkSessionCatalog")
    .config("spark.sql.catalog.spark_catalog.type", "hive")
    .getOrCreate()
)

def read_csv(path):
    print("Reading:", path)
    df = (spark.read
          .option("header", True)
          .option("multiLine", False)
          .csv(path))
    return df

def canon_cols(df):
    # Trim + lowercase column names to avoid surprises
    for c in df.columns:
        df = df.withColumnRenamed(c, c.strip().lower())
    return df

# ------------- READ RAW -------------
orders_df    = canon_cols(read_csv(f"s3://{RAW_BUCKET}/retail/orders/"))
customers_df = canon_cols(read_csv(f"s3://{RAW_BUCKET}/retail/customers/"))
products_df  = canon_cols(read_csv(f"s3://{RAW_BUCKET}/retail/products/"))

# ------------- PARSERS -------------
# Your file shows DD/MM/YYYY like 24/07/2025
def parse_order_date(col):
    return F.to_date(F.trim(col), 'dd/MM/yyyy')

# ------------- CASTS -------------
orders_typed = (
    orders_df
      .withColumn("order_id",    F.col("order_id").cast("long"))
      .withColumn("order_date_raw", F.col("order_date"))
      .withColumn("order_date",  parse_order_date(F.col("order_date")))
      .withColumn("customer_id", F.col("customer_id").cast("long"))
      .withColumn("product_id",  F.col("product_id").cast("long"))
      .withColumn("qty",         F.col("qty").cast("int"))
      # Handle possible commas in decimals (e.g., "147,43")
      .withColumn("unit_price",  F.regexp_replace(F.col("unit_price").cast("string"), ",", ".").cast("double"))
      .withColumn("channel",     F.col("channel"))
)

customers_typed = customers_df.withColumn("customer_id", F.col("customer_id").cast("long")) \
                              .withColumn("region", F.col("region"))
products_typed  = products_df .withColumn("product_id",  F.col("product_id").cast("long")) \
                              .withColumn("category", F.col("category"))

# ------------- ENRICH -------------
orders_enriched = (
    orders_typed
      .join(customers_typed.select("customer_id","region"), on="customer_id", how="left")
      .join(products_typed.select("product_id","category"), on="product_id", how="left")
      .withColumn("sales_amount", F.col("qty").cast("double") * F.col("unit_price"))
)

# ------------- SANITY CHECKS (fail fast) -------------
total = orders_enriched.count()
null_dates = orders_enriched.filter(F.col("order_date").isNull()).count()
null_ids   = orders_enriched.filter(
                F.col("order_id").isNull() |
                F.col("customer_id").isNull() |
                F.col("product_id").isNull()
            ).count()

print(f"[CHECK] rows={total}, null_order_date={null_dates}, null_key_fields={null_ids}")
orders_enriched.select("order_id","order_date_raw","order_date").orderBy("order_id").show(10, False)

if total == 0:
    raise RuntimeError("No rows read from orders CSVs.")
if null_dates > 0:
    raise RuntimeError(f"order_date parse failed for {null_dates} rows (expected format dd/MM/yyyy).")
if null_ids > 0:
    raise RuntimeError(f"Found NULLs in key fields (order_id/customer_id/product_id).")

# ------------- DDL (Iceberg) -------------
spark.sql(f"""
    CREATE TABLE IF NOT EXISTS glue_catalog.{SILVER_DB}.orders_silver (
        order_id BIGINT,
        order_date DATE,
        customer_id BIGINT,
        product_id BIGINT,
        qty INT,
        unit_price DOUBLE,
        channel STRING,
        region STRING,
        category STRING,
        sales_amount DOUBLE
    )
    USING ICEBERG
    PARTITIONED BY (days(order_date))
""")

# ------------- WRITE (append) -------------
(orders_enriched.select(
    "order_id","order_date","customer_id","product_id","qty","unit_price",
    "channel","region","category","sales_amount"
).writeTo(f"glue_catalog.{SILVER_DB}.orders_silver").append())

print("Transform complete")


```

#### 3.4.2 Upload the script to S3 (job script location)

```bash
export JOB_SCRIPT_BUCKET=$TMP_BUCKET   # reuse tmp bucket
aws s3 cp retail_bronze_to_silver.py s3://$JOB_SCRIPT_BUCKET/jobs/retail_bronze_to_silver.py
```

#### 3.4.3 Create the Glue job (Glue 4.0 / Spark 3.3)

```bash
export SILVER_WAREHOUSE="s3://$SILVER_BUCKET/"
export GLUE_JOB=retail_bronze_to_silver

aws glue create-job \
  --name "$GLUE_JOB" \
  --role "$GLUE_ROLE_ARN" \
  --glue-version "4.0" \
  --number-of-workers 2 \
  --worker-type G.1X \
  --command '{"Name":"glueetl","ScriptLocation":"s3://'"$JOB_SCRIPT_BUCKET"'/jobs/retail_bronze_to_silver.py","PythonVersion":"3"}' \
  --default-arguments '{
    "--enable-glue-datacatalog": "true",
    "--datalake-formats": "iceberg",

    "--conf:spark.sql.extensions": "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions",
    "--conf:spark.sql.catalog.glue_catalog": "org.apache.iceberg.spark.SparkCatalog",
    "--conf:spark.sql.catalog.glue_catalog.warehouse": "'"$SILVER_WAREHOUSE"'",
    "--conf:spark.sql.catalog.glue_catalog.catalog-impl": "org.apache.iceberg.aws.glue.GlueCatalog",
    "--conf:spark.sql.catalog.glue_catalog.io-impl": "org.apache.iceberg.aws.s3.S3FileIO",
    "--conf:spark.sql.catalog.spark_catalog": "org.apache.iceberg.spark.SparkSessionCatalog",
    "--conf:spark.sql.catalog.spark_catalog.type": "hive",

    "--RAW_BUCKET": "'"$RAW_BUCKET"'",
    "--SILVER_DB": "retail_silver",
    "--SILVER_WAREHOUSE": "'"$SILVER_WAREHOUSE"'"
  }'
```

> Note: I have keep the job creation minimal. You can add `--additional-python-modules` later once you see a successful baseline.

### 3.5 Run the transform (MANUAL)

#### Allow read on the script bucket/prefix

Update the Glue role to include ListBucket + GetObject for the TMP bucket’s jobs/ prefix.

```bash
# Recreate the inline policy to include TMP read
cat > glue-inline-policy.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::org-demo-lake-raw-$NAME"
    },
    {
      "Effect":"Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::org-demo-lake-raw-$NAME/*"
    },

    {
      "Effect":"Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::org-demo-lake-silver-$NAME"
    },
    {
      "Effect":"Allow",
      "Action": ["s3:PutObject","s3:GetObject"],
      "Resource": "arn:aws:s3:::org-demo-lake-silver-$NAME/*"
    },

    {
      "Effect":"Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::org-demo-lake-tmp-$NAME"
    },
    {
      "Effect":"Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::org-demo-lake-tmp-$NAME/jobs/*"
    },

    {
      "Effect":"Allow",
      "Action":[
        "glue:GetDatabase","glue:GetDatabases","glue:CreateDatabase",
        "glue:GetTable","glue:CreateTable","glue:UpdateTable",
        "glue:GetUserDefinedFunction","glue:CreateUserDefinedFunction"
      ],
      "Resource":"*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name "$GLUE_ROLE_NAME" \
  --policy-name retail-glue-inline \
  --policy-document file://glue-inline-policy.json

```


```bash
aws glue start-job-run --job-name $GLUE_JOB
# (Optionally poll)
aws glue get-job-runs --job-name $GLUE_JOB --max-results 1 --query 'JobRuns[0].JobRunState'
```

### 3.6 Run the transform (AUTOMATIC)

#### A) Daily at 01:15 UTC (schedule)

Assignment


#### B) Event-driven (whenever a new **orders** object lands)

##### 1) Create the starter Lambda (IAM + code)

```bash
export START_FN=start-glue-$GLUE_JOB
export START_ROLE=start-glue-role

# Role for Lambda
aws iam create-role --role-name $START_ROLE --assume-role-policy-document '{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]
}'

# Minimal permissions: start Glue jobs + write logs
aws iam put-role-policy --role-name $START_ROLE --policy-name start-glue-inline --policy-document '{
  "Version":"2012-10-17",
  "Statement":[
    {"Effect":"Allow","Action":["glue:StartJobRun","glue:GetJobRuns"],"Resource":"*"},
    {"Effect":"Allow","Action":["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],"Resource":"*"}
  ]
}'
```

**Starter Lambda code** (handles S3 event → derives a batch prefix → starts the job; safe to use your minimal version too, but this one passes a useful arg and avoids overlaps):

```bash
cat > start_glue.py <<'PY'
import os, json, urllib.parse, boto3, re

GLUE_JOB = os.environ["GLUE_JOB"]
glue = boto3.client("glue")

def handler(event, context):
    print("EVENT:", json.dumps(event))
    # S3 notification delivers Records[]
    rec = event["Records"][0]
    bucket = rec["s3"]["bucket"]["name"]
    key = urllib.parse.unquote_plus(rec["s3"]["object"]["key"])

    # Only react to orders CSVs
    if not key.startswith("retail/orders/") or not key.endswith(".csv"):
        print("Skip:", key)
        return {"skipped": True}

    # Extract a batch prefix if your keys look like:
    # retail/orders/ingest_date=YYYY-MM-DD/batch=0001/orders.csv
    m = re.match(r"(retail/orders/.*/batch=\d{4}/)", key)
    if m:
        prefix = m.group(1)
        batch_id = prefix.split("batch=")[1].strip("/").zfill(4)
    else:
        prefix = key.rsplit("/", 1)[0] + "/"
        batch_id = "unknown"

    raw_orders_path = f"s3://{bucket}/{prefix}"
    print("Starting Glue job:", GLUE_JOB, "RAW_ORDERS_PATH=", raw_orders_path, "BATCH_ID=", batch_id)

    # simple concurrency guard: skip if a run is still active
    runs = glue.get_job_runs(JobName=GLUE_JOB, MaxResults=1).get("JobRuns", [])
    if runs and runs[0]["JobRunState"] in ("RUNNING","STARTING","STOPPING"):
        print("Latest run still active; skipping to avoid overlap.")
        return {"skipped": "concurrent"}

    resp = glue.start_job_run(
        JobName=GLUE_JOB,
        Arguments={
            "--RAW_ORDERS_PATH": raw_orders_path,
            "--BATCH_ID": batch_id
        }
    )
    print("JobRunId:", resp["JobRunId"])
    return {"ok": True, "jobRunId": resp["JobRunId"]}
PY

zip start_glue.zip start_glue.py

aws lambda create-function \
  --function-name $START_FN \
  --runtime python3.12 \
  --handler start_glue.handler \
  --zip-file fileb://start_glue.zip \
  --role arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/$START_ROLE \
  --environment Variables="{GLUE_JOB=$GLUE_JOB}" \
  --timeout 30 --memory-size 256 \
  --region $REGION
```

> If you truly want the **ultra-minimal** Lambda you wrote:
>
> ```python
> import os, boto3
> def handler(event, context):
>     boto3.client("glue").start_job_run(JobName=os.environ["GLUE_JOB"])
> ```
>
> It works, but won’t pass the S3 path/batch to Glue and can cause overlapping runs. The “full” version above is safer for teaching.

---

#####  2) Allow S3 to invoke the new Lambda

```bash
aws lambda add-permission \
  --function-name $START_FN \
  --statement-id s3invoke-startglue \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$RAW_BUCKET \
  --region $REGION
```

---

#####  3) Wire S3 bucket notifications → both Lambdas

```bash
# Get ARNs for both Lambdas
LOG_FN_ARN=$(aws lambda get-function --function-name log-ingestion-run --query 'Configuration.FunctionArn' --output text --region $REGION)
START_FN_ARN=$(aws lambda get-function --function-name $START_FN --query 'Configuration.FunctionArn' --output text --region $REGION)

cat > notif.json <<JSON
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "log-customers",
      "LambdaFunctionArn": "$LOG_FN_ARN",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": { "Key": { "FilterRules": [ { "Name": "prefix", "Value": "retail/customers/" } ] } }
    },
    {
      "Id": "log-products",
      "LambdaFunctionArn": "$LOG_FN_ARN",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": { "Key": { "FilterRules": [ { "Name": "prefix", "Value": "retail/products/" } ] } }
    },
    {
      "Id": "start-glue-on-orders-csv",
      "LambdaFunctionArn": "$START_FN_ARN",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": { "Key": { "FilterRules": [
        { "Name": "prefix", "Value": "retail/orders/" },
        { "Name": "suffix", "Value": "orders.csv" }
      ] } }
    }
  ]
}
JSON

aws s3api put-bucket-notification-configuration \
  --bucket "$RAW_BUCKET" \
  --notification-configuration file://notif.json
```

Now any upload like:

```
aws s3 cp retail_data/orders.csv    s3://$RAW_BUCKET/retail/orders/ingest_date=$INGEST_DATE/orders.csv
```

will:

* invoke **log-ingestion-run** (run log in DynamoDB), and
* invoke **start-glue-…** (which starts the Glue job with `--RAW_ORDERS_PATH` pointing to that batch).

---

#### 4) Make sure your Glue script accepts `--RAW_ORDERS_PATH` (optional but recommended)

Add a fallback so you can still run it manually:

```python
# in retail_bronze_to_silver.py
from awsglue.utils import getResolvedOptions
args = getResolvedOptions(sys.argv, ['RAW_BUCKET','SILVER_DB','SILVER_WAREHOUSE'])
RAW_ORDERS_PATH = args.get('RAW_ORDERS_PATH')  # may be absent
BATCH_ID = args.get('BATCH_ID', 'manual')

if RAW_ORDERS_PATH:
    orders_df = spark.read.option("header", True).csv(RAW_ORDERS_PATH)
else:
    orders_df = spark.read.option("header", True).csv(f"s3://{RAW_BUCKET}/retail/orders/")
```

(Optionally add `.withColumn("ingest_batch", F.lit(BATCH_ID))` to tag rows per batch.)

---

## 4) Query (Athena v3) — sanity & time travel

### One-time setup in Athena Console

1. Top-right, switch **Workgroup** to **`retail-wg`** (engine v3).
2. Click **Edit settings** and set the **query result location** to `s3://org-demo-lake-tmp-<yourname>/athena/`.
3. Left panel:

   * **Data source**: `AwsDataCatalog`
   * **Database**: select **`retail_silver`**

(If you prefer, you can also just run `USE retail_silver;` in the editor.)

---

### Sanity & time-travel queries

Paste these exactly (they qualify the DB, so they work even if the left panel DB isn’t switched):

```sql
-- sanity: a few rows
SELECT * FROM retail_silver.orders_silver LIMIT 10;

-- last 7 days sales (by order_date)
SELECT date_trunc('day', order_date) AS d, sum(sales_amount) AS sales
FROM retail_silver.orders_silver
GROUP BY 1
ORDER BY 1 DESC
LIMIT 7;

-- list Iceberg snapshots for this table
SELECT *
FROM retail_silver."orders_silver$snapshots"
ORDER BY committed_at DESC
LIMIT 5;

-- time travel using a snapshot id (copy one from the query above)
SELECT count(*)
FROM retail_silver.orders_silver
FOR VERSION AS OF <snapshot_id>;

-- or time travel by timestamp (UTC)
SELECT *
FROM retail_silver.orders_silver
FOR TIMESTAMP AS OF TIMESTAMP '2025-09-22 00:00:00'
LIMIT 10;

-- (optional) table history view
SELECT *
FROM retail_silver."orders_silver$history"
ORDER BY made_current_at DESC
LIMIT 5;

-- (optional) partition pruning check
EXPLAIN
SELECT sum(sales_amount)
FROM retail_silver.orders_silver
WHERE order_date BETWEEN DATE '2025-09-15' AND DATE '2025-09-21';
```

### What to expect / quick explanations

* **`retail_silver.orders_silver`** is your curated Iceberg table (ACID, partitioned by `days(order_date)`).
* **`$snapshots`** is a system table that lists each commit (snapshot\_id, parent\_id, committed\_at, summary).
* **Time travel**:

  * `FOR VERSION AS OF <snapshot_id>` queries the table **as it was at that snapshot**.
  * `FOR TIMESTAMP AS OF TIMESTAMP '…'` picks the latest snapshot at/before that time.
* **Partition pruning**: with the `WHERE order_date BETWEEN …` filter, the `EXPLAIN` plan should indicate scanning only relevant day partitions (smaller bytes scanned).

### If you don’t see the table

* Run:

  ```sql
  SHOW TABLES IN retail_silver;
  SHOW CREATE TABLE retail_silver.orders_silver;
  ```
* If `SHOW TABLES` returns nothing, re-check in Glue Catalog (Database `retail_silver`) that `orders_silver` exists. If it does, make sure Athena is on **engine v3** (the `retail-wg` workgroup).


## 5) Clean-up (to avoid charges)

> **Before you run:**
>
> * Make sure you don’t need the data anymore.
> * Replace `<yourname>` if you didn’t already export `NAME`.
> * Run in the **same region** you used for the lab.

You can create `cleanup.sh` file or It is ready to run as-is in your shell.


```bash
# ========== 0) Vars ==========
export REGION=eu-north-1
export NAME=<yourname>

export RAW_BUCKET=org-demo-lake-raw-$NAME
export SILVER_BUCKET=org-demo-lake-silver-$NAME
export TMP_BUCKET=org-demo-lake-tmp-$NAME

export DDB_TABLE=ingestion_runs
export LAMBDA_NAME=log-ingestion-run
export START_FN=start-glue-retail_bronze_to_silver
export ROLE_NAME=lambda-log-ingestion-run-role
export POLICY_NAME=lambda-log-ingestion-run-policy

export GLUE_ROLE_NAME=retail-glue-role
export GLUE_JOB=retail_bronze_to_silver

export ATHENA_WG=retail-wg
export RULE_NAME=run-glue-daily
export EV_RULE=on-orders-landed

# ========= 1) Stop triggers first =========
# 1.1 EventBridge rules -> remove targets -> delete rules
aws events list-targets-by-rule --rule "$RULE_NAME" --region $REGION --query 'Targets[].Id' --output text | \
  xargs -I {} aws events remove-targets --rule "$RULE_NAME" --ids {} --region $REGION 2>/dev/null || true
aws events delete-rule --name "$RULE_NAME" --region $REGION 2>/dev/null || true

aws events list-targets-by-rule --rule "$EV_RULE" --region $REGION --query 'Targets[].Id' --output text | \
  xargs -I {} aws events remove-targets --rule "$EV_RULE" --ids {} --region $REGION 2>/dev/null || true
aws events delete-rule --name "$EV_RULE" --region $REGION 2>/dev/null || true

# ========= 2) S3 notifications off (so deletes don’t invoke Lambda) =========
aws s3api put-bucket-notification-configuration --bucket $RAW_BUCKET --notification-configuration '{}' --region $REGION 2>/dev/null || true

# ========= 3) Drop Iceberg tables (Athena) =========
# If you prefer, do this in console; below is CLI fire-and-forget.
aws athena start-query-execution --work-group "$ATHENA_WG" --query-string "DROP TABLE IF EXISTS retail_silver.orders_silver" --region $REGION >/dev/null 2>&1 || true
aws athena start-query-execution --work-group "$ATHENA_WG" --query-string "DROP TABLE IF EXISTS retail_silver.customers_silver" --region $REGION >/dev/null 2>&1 || true
aws athena start-query-execution --work-group "$ATHENA_WG" --query-string "DROP TABLE IF EXISTS retail_silver.products_silver"  --region $REGION >/dev/null 2>&1 || true

# ========= 4) Delete Glue job & databases =========
aws glue delete-job --job-name "$GLUE_JOB" --region $REGION 2>/dev/null || true
aws glue delete-database --name retail_silver --region $REGION 2>/dev/null || true
aws glue delete-database --name retail_bronze --region $REGION 2>/dev/null || true

# (If you created a crawler for RAW, remove it too)
# aws glue delete-crawler --name retail-raw-crawler --region $REGION 2>/dev/null || true

# ========= 5) Athena workgroup (optional delete) =========
# This removes named queries/workgroup settings under this WG.
aws athena delete-work-group --work-group "$ATHENA_WG" --recursive-delete-option --region $REGION 2>/dev/null || true

# ========= 6) Lambda functions & permissions =========
# Remove EventBridge invoke permissions (in case they exist)
aws lambda remove-permission --function-name "$START_FN" --statement-id ev-invoke --region $REGION 2>/dev/null || true
aws lambda remove-permission --function-name "$START_FN" --statement-id ev-orders --region $REGION 2>/dev/null || true
aws lambda remove-permission --function-name "$LAMBDA_NAME" --statement-id s3invoke --region $REGION 2>/dev/null || true

# Delete functions
aws lambda delete-function --function-name "$START_FN" --region $REGION 2>/dev/null || true
aws lambda delete-function --function-name "$LAMBDA_NAME" --region $REGION 2>/dev/null || true

# ========= 7) DynamoDB table =========
aws dynamodb delete-table --table-name "$DDB_TABLE" --region $REGION 2>/dev/null || true

# ========= 8) CloudWatch log groups (tidy) =========
aws logs delete-log-group --log-group-name "/aws/lambda/$LAMBDA_NAME" --region $REGION 2>/dev/null || true
aws logs delete-log-group --log-group-name "/aws/lambda/$START_FN" --region $REGION 2>/dev/null || true
aws logs delete-log-group --log-group-name "/aws-glue/jobs/output" --region $REGION 2>/dev/null || true
aws logs delete-log-group --log-group-name "/aws-glue/jobs/error"  --region $REGION 2>/dev/null || true
aws logs delete-log-group --log-group-name "/aws-glue/jobs/logs-v2" --region $REGION 2>/dev/null || true

# ========= 9) IAM cleanup =========
# 9.1 Start-job Lambda role (if you created it)
export START_ROLE=start-glue-role
aws iam delete-role-policy --role-name "$START_ROLE" --policy-name start-glue-inline 2>/dev/null || true
aws iam delete-role --role-name "$START_ROLE" 2>/dev/null || true

# 9.2 Ingestion Lambda role
aws iam delete-role-policy --role-name "$ROLE_NAME" --policy-name "$POLICY_NAME" 2>/dev/null || true
aws iam delete-role --role-name "$ROLE_NAME" 2>/dev/null || true

# 9.3 Glue role
aws iam delete-role-policy --role-name "$GLUE_ROLE_NAME" --policy-name retail-glue-inline 2>/dev/null || true
aws iam detach-role-policy --role-name "$GLUE_ROLE_NAME" --policy-arn arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole 2>/dev/null || true
aws iam delete-role --role-name "$GLUE_ROLE_NAME" 2>/dev/null || true

# ========= 10) Empty and delete S3 buckets =========
# (order: raw/silver/tmp; remove objects then the bucket)
aws s3 rm s3://$RAW_BUCKET --recursive --region $REGION 2>/dev/null || true
aws s3 rm s3://$SILVER_BUCKET --recursive --region $REGION 2>/dev/null || true
aws s3 rm s3://$TMP_BUCKET --recursive --region $REGION 2>/dev/null || true

# If versioning was enabled on RAW, remove all versions (safety: do for all three)
for B in $RAW_BUCKET $SILVER_BUCKET $TMP_BUCKET; do
  aws s3api list-object-versions --bucket $B --region $REGION \
    --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}, DeleteMarkers: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json | \
    jq -c '.Objects[]?, .DeleteMarkers[]?' 2>/dev/null | \
    while read -r line; do
      KEY=$(echo "$line" | jq -r '.Key')
      VID=$(echo "$line" | jq -r '.VersionId')
      aws s3api delete-object --bucket $B --key "$KEY" --version-id "$VID" --region $REGION >/dev/null 2>&1 || true
    done
done

# Finally remove the buckets
aws s3 rb s3://$RAW_BUCKET --region $REGION 2>/dev/null || true
aws s3 rb s3://$SILVER_BUCKET --region $REGION 2>/dev/null || true
aws s3 rb s3://$TMP_BUCKET --region $REGION 2>/dev/null || true

echo "✅ Cleanup attempted. If anything remains, it likely has a dependency or naming mismatch."
```

## Notes & tips

* **Iceberg tables first, then DBs**: if you try to delete a Glue database with tables still registered, it will fail.
* **Workgroup**: Deleting `retail-wg` is optional; if you keep it, it costs nothing.
* **S3 versioning**: If you enabled versioning, use the “delete versions” loop above; otherwise simple `rm --recursive` is enough.
* **Missing jq?** If the versioned-deletes step complains about `jq`, install it or skip the loop if you didn’t enable versioning.


