# Game Analytics Part 2: Streaming Pipeline - Kafka to BigQuery

**Building a Scalable Data Pipeline with Kafka, Dataflow, BigQuery, and Airflow**

This is Part 2 of a 3-part series on building AI-powered game analytics:
- [Part 1: Client-Side Event Tracking](./game-analytics-part1-event-tracking)
- **Part 2: Streaming Pipeline - Kafka to BigQuery** (this post)
- [Part 3: ML-Powered Game Analytics](./game-analytics-part3-ml-predictions)

---

## Overview

In Part 1, we built a client SDK that reliably sends events to a tracking server. Now we need to:

- Ingest millions of events per second
- Process and enrich events in real-time
- Store data for analytics and ML training
- Orchestrate batch ETL jobs

This post covers the complete server-side data pipeline.

---

## End-to-End Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATA PIPELINE ARCHITECTURE                             │
└─────────────────────────────────────────────────────────────────────────────────────┘

  Mobile Clients                    Ingestion                     Processing
 ┌─────────────┐               ┌─────────────────┐           ┌─────────────────┐
 │   iOS App   │──┐            │                 │           │                 │
 └─────────────┘  │            │   API Gateway   │           │    Dataflow     │
 ┌─────────────┐  │   HTTPS    │   (Load Bal.)   │   Kafka   │    (Apache      │
 │ Android App │──┼───────────▶│                 │──────────▶│     Beam)       │
 └─────────────┘  │            │   ┌─────────┐   │           │                 │
 ┌─────────────┐  │            │   │Tracking │   │           │  • Validation   │
 │   Web App   │──┘            │   │ Server  │   │           │  • Enrichment   │
 └─────────────┘               │   └─────────┘   │           │  • Aggregation  │
                               └─────────────────┘           └────────┬────────┘
                                                                      │
           ┌──────────────────────────────────────────────────────────┘
           │
           ▼
 ┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
 │                 │         │                 │         │                 │
 │    BigQuery     │◀────────│    Airflow      │────────▶│   ML Platform   │
 │   (Warehouse)   │   ETL   │  (Orchestrator) │  Train  │   (Vertex AI)   │
 │                 │         │                 │         │                 │
 └────────┬────────┘         └─────────────────┘         └─────────────────┘
          │
          │
          ▼
 ┌─────────────────┐
 │   Dashboards    │
 │   (Looker/      │
 │    Metabase)    │
 └─────────────────┘
```

---

## Component 1: Event Ingestion with Kafka

### Why Kafka?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         KAFKA BENEFITS                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐               │
│  │  Decouple   │     │   Buffer    │     │  Replay     │               │
│  │  Producers  │     │   Spikes    │     │  Events     │               │
│  │  & Consumer │     │             │     │             │               │
│  └─────────────┘     └─────────────┘     └─────────────┘               │
│                                                                         │
│  • Tracking server pushes, doesn't wait for processing                  │
│  • Handle 10x traffic spikes without data loss                          │
│  • Reprocess historical data for new features                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Topic Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         KAFKA TOPICS                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  game-events-raw          (Partitions: 64, Retention: 7 days)           │
│  ├── Partition 0: user_id % 64 == 0                                     │
│  ├── Partition 1: user_id % 64 == 1                                     │
│  ├── ...                                                                │
│  └── Partition 63: user_id % 64 == 63                                   │
│                                                                         │
│  game-events-enriched     (Partitions: 64, Retention: 3 days)           │
│  └── Events with added metadata (geo, device info, etc.)                │
│                                                                         │
│  game-events-dlq          (Partitions: 8, Retention: 30 days)           │
│  └── Dead letter queue for failed events                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Producer (Tracking Server)

```python
from confluent_kafka import Producer
import json

class EventProducer:
    def __init__(self, bootstrap_servers: str):
        self.producer = Producer({
            'bootstrap.servers': bootstrap_servers,
            'acks': 'all',  # Wait for all replicas
            'retries': 3,
            'retry.backoff.ms': 100,
            'linger.ms': 5,  # Batch for 5ms
            'batch.size': 65536,  # 64KB batches
            'compression.type': 'snappy',
        })

    def send_event(self, event: dict):
        """Send event to Kafka with user_id as partition key."""
        user_id = event.get('user_id', 'unknown')
        topic = 'game-events-raw'

        self.producer.produce(
            topic=topic,
            key=user_id.encode('utf-8'),
            value=json.dumps(event).encode('utf-8'),
            callback=self._delivery_callback
        )

    def _delivery_callback(self, err, msg):
        if err:
            logger.error(f"Delivery failed: {err}")
            metrics.increment('kafka.produce.error')
        else:
            metrics.increment('kafka.produce.success')

    def flush(self):
        """Flush pending messages (call on shutdown)."""
        self.producer.flush(timeout=10)
```

---

## Component 2: Stream Processing with Dataflow

### Apache Beam Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DATAFLOW PIPELINE                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│   │  Read    │───▶│  Parse   │───▶│  Enrich  │───▶│ Validate │         │
│   │  Kafka   │    │  JSON    │    │  Event   │    │  Schema  │         │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘         │
│                                                         │               │
│                          ┌──────────────────────────────┤               │
│                          │                              │               │
│                          ▼                              ▼               │
│                   ┌──────────┐                   ┌──────────┐          │
│                   │   DLQ    │                   │  Window  │          │
│                   │  (Errors)│                   │  (1 min) │          │
│                   └──────────┘                   └──────────┘          │
│                                                         │               │
│                          ┌──────────────────────────────┤               │
│                          │                              │               │
│                          ▼                              ▼               │
│                   ┌──────────┐                   ┌──────────┐          │
│                   │ BigQuery │                   │ BigQuery │          │
│                   │ (events) │                   │ (aggs)   │          │
│                   └──────────┘                   └──────────┘          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pipeline Implementation

```python
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions
from apache_beam.io.kafka import ReadFromKafka
from apache_beam.io.gcp.bigquery import WriteToBigQuery
import json

class ParseEvent(beam.DoFn):
    """Parse JSON and extract fields."""

    def process(self, kafka_message):
        try:
            event = json.loads(kafka_message.value.decode('utf-8'))

            yield {
                'event_id': event['id'],
                'user_id': event.get('user_id'),
                'session_id': event.get('session_id'),
                'timestamp': event.get('timestamp'),
                'event_type': event.get('type'),
                'parameters': json.dumps(event.get('parameters', {})),
                'app_version': event.get('app_version'),
                'platform': event.get('platform'),
            }
        except Exception as e:
            # Send to dead letter queue
            yield beam.pvalue.TaggedOutput('dlq', {
                'raw_message': kafka_message.value.decode('utf-8'),
                'error': str(e),
                'timestamp': datetime.utcnow().isoformat()
            })


class EnrichEvent(beam.DoFn):
    """Add derived fields and external data."""

    def setup(self):
        # Load lookup tables (cached)
        self.geo_lookup = load_geo_database()
        self.device_lookup = load_device_database()

    def process(self, event):
        # Add geo information from IP
        if ip := event.get('ip_address'):
            geo = self.geo_lookup.get(ip, {})
            event['country'] = geo.get('country')
            event['region'] = geo.get('region')
            event['city'] = geo.get('city')

        # Add device information
        if device_id := event.get('device_model'):
            device = self.device_lookup.get(device_id, {})
            event['device_category'] = device.get('category')
            event['device_tier'] = device.get('performance_tier')

        # Add derived timestamp fields
        ts = datetime.fromisoformat(event['timestamp'])
        event['event_date'] = ts.date().isoformat()
        event['event_hour'] = ts.hour
        event['day_of_week'] = ts.weekday()

        yield event


class ValidateSchema(beam.DoFn):
    """Validate event against schema."""

    REQUIRED_FIELDS = ['event_id', 'user_id', 'timestamp', 'event_type']

    def process(self, event):
        # Check required fields
        missing = [f for f in self.REQUIRED_FIELDS if not event.get(f)]
        if missing:
            yield beam.pvalue.TaggedOutput('dlq', {
                'event': event,
                'error': f'Missing required fields: {missing}'
            })
            return

        # Validate timestamp
        try:
            ts = datetime.fromisoformat(event['timestamp'])
            if ts > datetime.utcnow() + timedelta(hours=1):
                yield beam.pvalue.TaggedOutput('dlq', {
                    'event': event,
                    'error': 'Timestamp in future'
                })
                return
        except ValueError:
            yield beam.pvalue.TaggedOutput('dlq', {
                'event': event,
                'error': 'Invalid timestamp format'
            })
            return

        yield event


def run_pipeline():
    options = PipelineOptions([
        '--runner=DataflowRunner',
        '--project=my-gcp-project',
        '--region=us-central1',
        '--temp_location=gs://my-bucket/temp',
        '--streaming',
        '--autoscaling_algorithm=THROUGHPUT_BASED',
        '--max_num_workers=50',
    ])

    with beam.Pipeline(options=options) as p:
        # Read from Kafka
        raw_events = (
            p
            | 'ReadKafka' >> ReadFromKafka(
                consumer_config={
                    'bootstrap.servers': 'kafka-broker:9092',
                    'group.id': 'dataflow-consumer',
                    'auto.offset.reset': 'latest',
                },
                topics=['game-events-raw'],
                with_metadata=True
            )
        )

        # Parse, enrich, validate
        processed = (
            raw_events
            | 'Parse' >> beam.ParDo(ParseEvent()).with_outputs('dlq', main='events')
        )

        enriched = (
            processed.events
            | 'Enrich' >> beam.ParDo(EnrichEvent())
            | 'Validate' >> beam.ParDo(ValidateSchema()).with_outputs('dlq', main='valid')
        )

        # Write valid events to BigQuery
        (
            enriched.valid
            | 'WriteBQ' >> WriteToBigQuery(
                table='project:dataset.events',
                schema=EVENT_SCHEMA,
                write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
                create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED,
                method='STREAMING_INSERTS'
            )
        )

        # Write errors to DLQ table
        dlq_events = (
            (processed.dlq, enriched.dlq)
            | 'FlattenDLQ' >> beam.Flatten()
        )

        (
            dlq_events
            | 'WriteDLQ' >> WriteToBigQuery(
                table='project:dataset.events_dlq',
                schema=DLQ_SCHEMA,
                write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
            )
        )

        # Compute real-time aggregations (1-minute windows)
        (
            enriched.valid
            | 'Window' >> beam.WindowInto(beam.window.FixedWindows(60))
            | 'KeyByLevel' >> beam.Map(lambda e: ((e['event_type'], e.get('level_id')), e))
            | 'CountPerLevel' >> beam.combiners.Count.PerKey()
            | 'FormatAgg' >> beam.Map(format_aggregation)
            | 'WriteAggs' >> WriteToBigQuery(
                table='project:dataset.events_agg_1min',
                schema=AGG_SCHEMA,
            )
        )


if __name__ == '__main__':
    run_pipeline()
```

---

## Component 3: BigQuery Data Warehouse

### Schema Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      BIGQUERY SCHEMA DESIGN                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PARTITIONING: By event_date (daily)                                    │
│  CLUSTERING: By event_type, user_id                                     │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  events (partitioned table)                                      │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  event_id        STRING     NOT NULL                             │   │
│  │  user_id         STRING     NOT NULL                             │   │
│  │  session_id      STRING                                          │   │
│  │  event_type      INT64      NOT NULL                             │   │
│  │  event_name      STRING                                          │   │
│  │  timestamp       TIMESTAMP  NOT NULL                             │   │
│  │  event_date      DATE       NOT NULL  (partition key)            │   │
│  │  parameters      JSON                                            │   │
│  │  platform        STRING     (ios, android, web)                  │   │
│  │  app_version     STRING                                          │   │
│  │  country         STRING                                          │   │
│  │  device_category STRING     (phone, tablet)                      │   │
│  │  device_tier     STRING     (low, mid, high)                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  level_metrics (derived table)                                   │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │  level_id        INT64      NOT NULL                             │   │
│  │  date            DATE       NOT NULL                             │   │
│  │  attempts        INT64      total play attempts                  │   │
│  │  completions     INT64      successful completions               │   │
│  │  pass_rate       FLOAT64    completions / attempts               │   │
│  │  avg_moves       FLOAT64    average moves used                   │   │
│  │  avg_time_sec    FLOAT64    average time to complete             │   │
│  │  p50_moves       INT64      median moves                         │   │
│  │  p95_moves       INT64      95th percentile moves                │   │
│  │  booster_usage   FLOAT64    % of attempts using boosters         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Table Creation

```sql
-- Main events table (partitioned and clustered)
CREATE TABLE `project.analytics.events`
(
    event_id STRING NOT NULL,
    user_id STRING NOT NULL,
    session_id STRING,
    event_type INT64 NOT NULL,
    event_name STRING,
    timestamp TIMESTAMP NOT NULL,
    event_date DATE NOT NULL,
    parameters JSON,
    platform STRING,
    app_version STRING,
    country STRING,
    region STRING,
    device_category STRING,
    device_tier STRING,
    is_paying_user BOOL,
    days_since_install INT64
)
PARTITION BY event_date
CLUSTER BY event_type, user_id
OPTIONS (
    partition_expiration_days = 365,
    require_partition_filter = true
);

-- Level metrics aggregation table
CREATE TABLE `project.analytics.level_metrics`
(
    level_id INT64 NOT NULL,
    date DATE NOT NULL,
    attempts INT64,
    completions INT64,
    pass_rate FLOAT64,
    avg_moves FLOAT64,
    avg_time_seconds FLOAT64,
    p50_moves INT64,
    p95_moves INT64,
    unique_players INT64,
    booster_usage_rate FLOAT64,
    retry_rate FLOAT64
)
PARTITION BY date
CLUSTER BY level_id;
```

---

## Component 4: Airflow Orchestration

### DAG Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      AIRFLOW DAG ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Daily ETL Pipeline (runs at 02:00 UTC)                                 │
│                                                                         │
│  ┌──────────────┐                                                       │
│  │   Sensor:    │                                                       │
│  │ Wait for data│                                                       │
│  └──────┬───────┘                                                       │
│         │                                                               │
│         ▼                                                               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Extract    │───▶│  Transform   │───▶│    Load      │              │
│  │  Raw Events  │    │  & Aggregate │    │   Metrics    │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│                             │                                           │
│                             ▼                                           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │    User      │    │   Level      │    │   Revenue    │              │
│  │   Metrics    │    │   Metrics    │    │   Metrics    │              │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘              │
│         │                   │                   │                       │
│         └───────────────────┼───────────────────┘                       │
│                             │                                           │
│                             ▼                                           │
│                      ┌──────────────┐                                   │
│                      │   Trigger    │                                   │
│                      │   ML Train   │                                   │
│                      └──────────────┘                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### DAG Implementation

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.google.cloud.operators.bigquery import (
    BigQueryInsertJobOperator,
    BigQueryCheckOperator,
)
from airflow.providers.google.cloud.sensors.bigquery import (
    BigQueryTablePartitionExistenceSensor,
)
from airflow.utils.dates import days_ago
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email_on_failure': True,
    'email': ['data-alerts@example.com'],
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

with DAG(
    dag_id='game_analytics_daily',
    default_args=default_args,
    description='Daily game analytics ETL pipeline',
    schedule_interval='0 2 * * *',  # 02:00 UTC daily
    start_date=days_ago(1),
    catchup=False,
    tags=['analytics', 'game', 'daily'],
) as dag:

    # Task 1: Wait for data to be available
    wait_for_data = BigQueryTablePartitionExistenceSensor(
        task_id='wait_for_data',
        project_id='my-project',
        dataset_id='analytics',
        table_id='events',
        partition_id='{{ ds_nodash }}',
        timeout=3600,  # 1 hour
        poke_interval=300,  # Check every 5 minutes
    )

    # Task 2: Compute level metrics
    compute_level_metrics = BigQueryInsertJobOperator(
        task_id='compute_level_metrics',
        configuration={
            'query': {
                'query': """
                    INSERT INTO `project.analytics.level_metrics`
                    (level_id, date, attempts, completions, pass_rate,
                     avg_moves, avg_time_seconds, p50_moves, p95_moves,
                     unique_players, booster_usage_rate, retry_rate)

                    WITH level_events AS (
                        SELECT
                            CAST(JSON_VALUE(parameters, '$.level_id') AS INT64) as level_id,
                            user_id,
                            event_type,
                            CAST(JSON_VALUE(parameters, '$.moves_used') AS INT64) as moves_used,
                            CAST(JSON_VALUE(parameters, '$.time_seconds') AS INT64) as time_seconds,
                            JSON_VALUE(parameters, '$.boosters_used') as boosters_used
                        FROM `project.analytics.events`
                        WHERE event_date = '{{ ds }}'
                          AND event_type IN (1001, 1002, 1003)  -- level_start, complete, fail
                    ),

                    level_stats AS (
                        SELECT
                            level_id,
                            COUNT(*) as attempts,
                            COUNTIF(event_type = 1002) as completions,
                            AVG(IF(event_type = 1002, moves_used, NULL)) as avg_moves,
                            AVG(IF(event_type = 1002, time_seconds, NULL)) as avg_time_seconds,
                            APPROX_QUANTILES(IF(event_type = 1002, moves_used, NULL), 100)[OFFSET(50)] as p50_moves,
                            APPROX_QUANTILES(IF(event_type = 1002, moves_used, NULL), 100)[OFFSET(95)] as p95_moves,
                            COUNT(DISTINCT user_id) as unique_players,
                            COUNTIF(boosters_used IS NOT NULL AND boosters_used != '[]') / COUNT(*) as booster_usage_rate
                        FROM level_events
                        GROUP BY level_id
                    ),

                    retry_stats AS (
                        SELECT
                            level_id,
                            COUNTIF(attempt_num > 1) / COUNT(*) as retry_rate
                        FROM (
                            SELECT
                                level_id,
                                user_id,
                                ROW_NUMBER() OVER (PARTITION BY level_id, user_id ORDER BY timestamp) as attempt_num
                            FROM level_events
                            WHERE event_type = 1001  -- level_start
                        )
                        GROUP BY level_id
                    )

                    SELECT
                        s.level_id,
                        DATE('{{ ds }}') as date,
                        s.attempts,
                        s.completions,
                        SAFE_DIVIDE(s.completions, s.attempts) as pass_rate,
                        s.avg_moves,
                        s.avg_time_seconds,
                        s.p50_moves,
                        s.p95_moves,
                        s.unique_players,
                        s.booster_usage_rate,
                        COALESCE(r.retry_rate, 0) as retry_rate
                    FROM level_stats s
                    LEFT JOIN retry_stats r USING (level_id)
                """,
                'useLegacySql': False,
                'writeDisposition': 'WRITE_APPEND',
            }
        },
    )

    # Task 3: Compute user metrics
    compute_user_metrics = BigQueryInsertJobOperator(
        task_id='compute_user_metrics',
        configuration={
            'query': {
                'query': """
                    INSERT INTO `project.analytics.user_daily_metrics`
                    (user_id, date, sessions, events, levels_played,
                     levels_completed, playtime_minutes, revenue_usd)

                    SELECT
                        user_id,
                        DATE('{{ ds }}') as date,
                        COUNT(DISTINCT session_id) as sessions,
                        COUNT(*) as events,
                        COUNT(DISTINCT IF(event_type = 1001,
                            JSON_VALUE(parameters, '$.level_id'), NULL)) as levels_played,
                        COUNTIF(event_type = 1002) as levels_completed,
                        SUM(CAST(JSON_VALUE(parameters, '$.duration_sec') AS INT64)) / 60.0 as playtime_minutes,
                        SUM(IF(event_type = 2001,
                            CAST(JSON_VALUE(parameters, '$.price_usd') AS FLOAT64), 0)) as revenue_usd
                    FROM `project.analytics.events`
                    WHERE event_date = '{{ ds }}'
                    GROUP BY user_id
                """,
                'useLegacySql': False,
            }
        },
    )

    # Task 4: Data quality checks
    data_quality_check = BigQueryCheckOperator(
        task_id='data_quality_check',
        sql="""
            SELECT
                COUNTIF(level_id IS NULL) = 0 as no_null_levels,
                COUNTIF(pass_rate < 0 OR pass_rate > 1) = 0 as valid_pass_rates,
                COUNT(*) > 0 as has_data
            FROM `project.analytics.level_metrics`
            WHERE date = '{{ ds }}'
        """,
        use_legacy_sql=False,
    )

    # Task 5: Trigger ML training (weekly)
    def should_trigger_ml_training(**context):
        # Trigger on Sundays
        execution_date = context['execution_date']
        return execution_date.weekday() == 6

    trigger_ml_training = PythonOperator(
        task_id='trigger_ml_training',
        python_callable=trigger_vertex_ai_pipeline,
        op_kwargs={
            'pipeline_name': 'difficulty-prediction-training',
            'parameters': {
                'training_date': '{{ ds }}',
                'lookback_days': 30,
            }
        },
    )

    # Task 6: Export to feature store
    export_features = BigQueryInsertJobOperator(
        task_id='export_features',
        configuration={
            'query': {
                'query': """
                    EXPORT DATA OPTIONS(
                        uri='gs://my-bucket/features/level_features/{{ ds }}/*.parquet',
                        format='PARQUET',
                        overwrite=true
                    ) AS
                    SELECT
                        level_id,
                        pass_rate,
                        avg_moves,
                        p95_moves,
                        retry_rate,
                        booster_usage_rate
                    FROM `project.analytics.level_metrics`
                    WHERE date = '{{ ds }}'
                """,
                'useLegacySql': False,
            }
        },
    )

    # Define task dependencies
    (
        wait_for_data
        >> [compute_level_metrics, compute_user_metrics]
        >> data_quality_check
        >> [trigger_ml_training, export_features]
    )
```

---

## Monitoring & Alerting

### Pipeline Metrics Dashboard

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PIPELINE HEALTH DASHBOARD                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │  Events/sec     │  │  Lag (seconds)  │  │  Error Rate     │         │
│  │  ████████░░     │  │  ██░░░░░░░░     │  │  █░░░░░░░░░     │         │
│  │  125,432        │  │  12s            │  │  0.02%          │         │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘         │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Event Volume (24h)                                              │   │
│  │  150k ┤                    ╭──╮                                  │   │
│  │  100k ┤  ╭──╮   ╭──╮    ╭─╯  ╰─╮   ╭──╮                        │   │
│  │   50k ┤╭─╯  ╰─╮╭╯  ╰─╮╭─╯      ╰─╮╭╯  ╰─                       │   │
│  │     0 ┼──────────────────────────────────▶                       │   │
│  │       00:00    06:00    12:00    18:00    24:00                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Data Quality Alerts                                             │   │
│  │  ✓ Schema validation: PASS                                       │   │
│  │  ✓ Null rate < 1%: PASS                                         │   │
│  │  ✓ Event count within bounds: PASS                              │   │
│  │  ⚠ Unusual drop in level_complete events: WARNING               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Alert Rules

```yaml
# alerts.yaml
alerts:
  - name: high_kafka_lag
    condition: kafka_consumer_lag > 100000
    severity: critical
    message: "Kafka consumer lag exceeds 100k messages"
    runbook: "https://wiki/runbooks/kafka-lag"

  - name: high_dlq_rate
    condition: dlq_events_per_minute / total_events_per_minute > 0.01
    severity: warning
    message: "DLQ rate exceeds 1%"
    runbook: "https://wiki/runbooks/dlq-investigation"

  - name: dataflow_job_failed
    condition: dataflow_job_state == "FAILED"
    severity: critical
    message: "Dataflow streaming job failed"
    runbook: "https://wiki/runbooks/dataflow-recovery"

  - name: airflow_dag_failed
    condition: airflow_dag_run_state == "failed"
    severity: high
    message: "Daily ETL DAG failed"
    runbook: "https://wiki/runbooks/airflow-dag-recovery"

  - name: data_freshness
    condition: hours_since_last_event > 1
    severity: critical
    message: "No events received in over 1 hour"
    runbook: "https://wiki/runbooks/data-freshness"
```

---

## Cost Optimization

### BigQuery Best Practices

```sql
-- Use partitioned tables (reduces scan costs)
-- Partition by date, cluster by frequently filtered columns
SELECT * FROM events
WHERE event_date = '2024-01-15'  -- Only scans one partition
  AND event_type = 1001          -- Uses clustering

-- Use approximate functions for large aggregations
SELECT
    APPROX_COUNT_DISTINCT(user_id) as unique_users,  -- Faster than COUNT(DISTINCT)
    APPROX_QUANTILES(moves, 100)[OFFSET(50)] as median_moves
FROM events

-- Materialize frequently used aggregations
CREATE MATERIALIZED VIEW level_daily_stats AS
SELECT level_id, event_date, COUNT(*) as events
FROM events
GROUP BY level_id, event_date;
```

### Kafka Optimization

```python
# Producer batching reduces network overhead
producer_config = {
    'linger.ms': 10,        # Wait up to 10ms to batch
    'batch.size': 131072,   # 128KB batches
    'compression.type': 'lz4',  # Fast compression
}

# Consumer parallel processing
consumer_config = {
    'fetch.min.bytes': 65536,    # Wait for 64KB before fetching
    'fetch.max.wait.ms': 500,    # Or 500ms, whichever first
    'max.partition.fetch.bytes': 1048576,  # 1MB per partition
}
```

---

## Summary

| Component | Technology | Purpose |
|-----------|------------|---------|
| Ingestion | Kafka | Buffer, decouple, replay |
| Processing | Dataflow/Beam | Validate, enrich, aggregate |
| Storage | BigQuery | Analytics, ML training data |
| Orchestration | Airflow | ETL scheduling, dependencies |
| Monitoring | Prometheus/Grafana | Metrics, alerting |

**Key Design Principles:**

1. **Idempotent processing** - Handle duplicates gracefully
2. **Schema evolution** - Use JSON/Avro for flexibility
3. **Partition everything** - Time-based partitioning for cost/performance
4. **Monitor lag** - Critical metric for streaming health
5. **DLQ everything** - Never lose data, investigate errors later

---

## Next: ML Predictions

In [Part 3](./game-analytics-part3-ml-predictions), we'll use this data to build ML models for:

- Level difficulty prediction
- Player churn prediction
- Automated level testing with RL agents
- A/B test analysis

---

*Part of the [Game Analytics Series](./index#game-analytics)*
