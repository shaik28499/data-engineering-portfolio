# Sample ETL Script for AWS Glue

```python
# examples/python/sample_etl_job.py
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from pyspark.sql import DataFrame
from pyspark.sql.functions import *
from pyspark.sql.types import *
import logging

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ETLProcessor:
    """
    Generic ETL processor for data transformation tasks
    """
    
    def __init__(self, glue_context, job_name):
        self.glue_context = glue_context
        self.spark = glue_context.spark_session
        self.job_name = job_name
        
    def extract_from_database(self, connection_name: str, table_name: str) -> DataFrame:
        """
        Extract data from database using Glue connection
        """
        try:
            logger.info(f"Extracting data from {table_name}")
            
            # Create dynamic frame from database
            dynamic_frame = self.glue_context.create_dynamic_frame.from_catalog(
                database="your_database",
                table_name=table_name,
                connection_name=connection_name
            )
            
            # Convert to Spark DataFrame
            df = dynamic_frame.toDF()
            logger.info(f"Extracted {df.count()} records from {table_name}")
            
            return df
            
        except Exception as e:
            logger.error(f"Error extracting data from {table_name}: {str(e)}")
            raise
    
    def transform_data(self, df: DataFrame) -> DataFrame:
        """
        Apply business transformations to the data
        """
        try:
            logger.info("Starting data transformations")
            
            # Example transformations
            transformed_df = df \
                .filter(col("status") == "active") \
                .withColumn("processed_date", current_timestamp()) \
                .withColumn("year", year(col("created_date"))) \
                .withColumn("month", month(col("created_date"))) \
                .dropDuplicates(["id"]) \
                .na.drop()
            
            # Data quality checks
            self._validate_data_quality(transformed_df)
            
            logger.info(f"Transformation completed. Records: {transformed_df.count()}")
            return transformed_df
            
        except Exception as e:
            logger.error(f"Error during transformation: {str(e)}")
            raise
    
    def _validate_data_quality(self, df: DataFrame):
        """
        Perform data quality validations
        """
        # Check for null values in critical columns
        critical_columns = ["id", "created_date", "status"]
        
        for column in critical_columns:
            null_count = df.filter(col(column).isNull()).count()
            if null_count > 0:
                logger.warning(f"Found {null_count} null values in {column}")
        
        # Check for duplicate IDs
        total_count = df.count()
        unique_count = df.select("id").distinct().count()
        
        if total_count != unique_count:
            logger.warning(f"Found {total_count - unique_count} duplicate records")
    
    def load_to_s3(self, df: DataFrame, output_path: str, partition_columns: list = None):
        """
        Load data to S3 in parquet format
        """
        try:
            logger.info(f"Loading data to {output_path}")
            
            writer = df.write.mode("overwrite").format("parquet")
            
            if partition_columns:
                writer = writer.partitionBy(*partition_columns)
            
            writer.save(output_path)
            
            logger.info("Data successfully loaded to S3")
            
        except Exception as e:
            logger.error(f"Error loading data to S3: {str(e)}")
            raise
    
    def load_to_database(self, df: DataFrame, connection_name: str, table_name: str):
        """
        Load data to target database
        """
        try:
            logger.info(f"Loading data to {table_name}")
            
            # Convert DataFrame to DynamicFrame
            dynamic_frame = DynamicFrame.fromDF(df, self.glue_context, "dynamic_frame")
            
            # Write to database
            self.glue_context.write_dynamic_frame.from_jdbc_conf(
                frame=dynamic_frame,
                catalog_connection=connection_name,
                connection_options={
                    "dbtable": table_name,
                    "database": "target_database"
                }
            )
            
            logger.info("Data successfully loaded to database")
            
        except Exception as e:
            logger.error(f"Error loading data to database: {str(e)}")
            raise

def main():
    """
    Main ETL execution function
    """
    # Get job parameters
    args = getResolvedOptions(sys.argv, [
        'JOB_NAME',
        'source_connection',
        'source_table',
        'target_path',
        'target_connection',
        'target_table'
    ])
    
    # Initialize Spark and Glue contexts
    sc = SparkContext()
    glue_context = GlueContext(sc)
    job = Job(glue_context)
    job.init(args['JOB_NAME'], args)
    
    try:
        # Initialize ETL processor
        etl_processor = ETLProcessor(glue_context, args['JOB_NAME'])
        
        # Extract data
        source_df = etl_processor.extract_from_database(
            connection_name=args['source_connection'],
            table_name=args['source_table']
        )
        
        # Transform data
        transformed_df = etl_processor.transform_data(source_df)
        
        # Load to S3 (partitioned by year and month)
        etl_processor.load_to_s3(
            df=transformed_df,
            output_path=args['target_path'],
            partition_columns=['year', 'month']
        )
        
        # Load to target database
        etl_processor.load_to_database(
            df=transformed_df,
            connection_name=args['target_connection'],
            table_name=args['target_table']
        )
        
        logger.info("ETL job completed successfully")
        
    except Exception as e:
        logger.error(f"ETL job failed: {str(e)}")
        raise
    
    finally:
        job.commit()

if __name__ == "__main__":
    main()
```

## Data Validation Utilities

```python
# examples/python/data_validation_utils.py
from pyspark.sql import DataFrame
from pyspark.sql.functions import *
import logging

logger = logging.getLogger(__name__)

class DataValidator:
    """
    Utility class for data quality validations
    """
    
    @staticmethod
    def check_null_values(df: DataFrame, columns: list) -> dict:
        """
        Check for null values in specified columns
        """
        null_counts = {}
        
        for column in columns:
            null_count = df.filter(col(column).isNull()).count()
            null_counts[column] = null_count
            
            if null_count > 0:
                logger.warning(f"Column '{column}' has {null_count} null values")
        
        return null_counts
    
    @staticmethod
    def check_duplicates(df: DataFrame, key_columns: list) -> int:
        """
        Check for duplicate records based on key columns
        """
        total_count = df.count()
        unique_count = df.dropDuplicates(key_columns).count()
        duplicate_count = total_count - unique_count
        
        if duplicate_count > 0:
            logger.warning(f"Found {duplicate_count} duplicate records")
        
        return duplicate_count
    
    @staticmethod
    def check_data_freshness(df: DataFrame, date_column: str, max_age_days: int) -> bool:
        """
        Check if data is fresh (within specified days)
        """
        cutoff_date = date_sub(current_date(), max_age_days)
        old_records = df.filter(col(date_column) < cutoff_date).count()
        
        if old_records > 0:
            logger.warning(f"Found {old_records} records older than {max_age_days} days")
            return False
        
        return True
    
    @staticmethod
    def generate_data_profile(df: DataFrame) -> dict:
        """
        Generate basic data profiling statistics
        """
        profile = {
            'total_records': df.count(),
            'total_columns': len(df.columns),
            'column_stats': {}
        }
        
        for column in df.columns:
            col_type = df.schema[column].dataType
            
            if str(col_type) in ['StringType', 'IntegerType', 'LongType', 'DoubleType']:
                stats = df.select(
                    count(col(column)).alias('count'),
                    countDistinct(col(column)).alias('distinct_count'),
                    sum(when(col(column).isNull(), 1).otherwise(0)).alias('null_count')
                ).collect()[0]
                
                profile['column_stats'][column] = {
                    'count': stats['count'],
                    'distinct_count': stats['distinct_count'],
                    'null_count': stats['null_count'],
                    'null_percentage': (stats['null_count'] / stats['count']) * 100
                }
        
        return profile
```