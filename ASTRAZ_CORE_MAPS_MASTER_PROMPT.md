# 🗺️ ASTRAZ CORE MAPS ENGINE — MASTER PROMPT ULTRA DETALHADO
> **Para uso direto no Claude Code, Cursor, ou qualquer agente de desenvolvimento com acesso ao sistema de arquivos.**
> Versão: 2.0 · Stack: Elixir/Phoenix + Rust NIFs · Arquitetura: Distributed BEAM Cluster

---

## ⚙️ INSTRUÇÕES DE USO

Cole este prompt inteiro no terminal do seu agente de código (Claude Code, Cursor, Aider, etc.). O agente irá:

1. Criar a estrutura completa de diretórios do projeto
2. Gerar os 30 arquivos de skill em `.astraz_maps_skills/`
3. Bootstrapar a arquitetura core do sistema
4. Manter testes verdes em todos os ciclos

---

## 🚀 PROMPT PRINCIPAL (COPIE TUDO ABAIXO DESTA LINHA)

---

```
ACT AS A DISTRIBUTED SYSTEMS ARCHITECT, GEOSPATIAL ENGINEER, AND ELIXIR/RUST PERFORMANCE SPECIALIST WITH 15+ YEARS OF EXPERIENCE BUILDING MISSION-CRITICAL REAL-TIME SYSTEMS AT SCALE.

Your mission is to design and implement ASTRAZ CORE MAPS — a hyper-performant, fault-tolerant, real-time spatial positioning, routing, and telemetry microservice ecosystem for Astraz Studio.

═══════════════════════════════════════════════════════════
SYSTEM IDENTITY & CONSTRAINTS
═══════════════════════════════════════════════════════════

PROJECT NAME    : ASTRAZ CORE MAPS ENGINE
VERSION         : 1.0.0-alpha
PRIMARY LANG    : Elixir 1.16+ / OTP 26+
SECONDARY LANG  : Rust (via Rustler NIFs for CPU-intensive ops)
FRAMEWORK       : Phoenix 1.7+ with LiveView and Channels
DATABASE        : PostgreSQL 16 + PostGIS extension
CACHE LAYER     : ETS (hot) → Redis (warm) → PostgreSQL (cold)
CLUSTER         : libcluster + Horde for distributed process management
CONTAINER       : Docker multi-stage + docker-compose for local dev
MESSAGE BUS     : Broadway + GenStage for backpressure-managed pipelines
PROTOCOLS       : Protobuf (telemetry), MVT/PBF (map tiles), WebSocket (realtime)
SECURITY        : Asymmetric JWT + ECDSA tile signatures + strict tenant isolation
OBSERVABILITY   : :telemetry + OpenTelemetry + Prometheus-compatible metrics

TARGET METRICS (NON-NEGOTIABLE):
  - Telemetry ingestion: > 500,000 coordinate updates/sec per node
  - Geofence evaluation latency: < 2ms p99
  - Route calculation (A*): < 50ms for urban graphs up to 2M nodes
  - Tile serving (cached): < 5ms p99
  - System availability: 99.99% (< 52 minutes downtime/year)
  - Cluster recovery after node failure: < 10 seconds

═══════════════════════════════════════════════════════════
PHASE 0 — MANDATORY SKILL MANIFEST GENERATION
═══════════════════════════════════════════════════════════

BEFORE writing any application code, you MUST:

  1. Create the directory: .astraz_maps_skills/
  2. Write exactly 30 specialized markdown skill files inside it
  3. Each file must include: purpose, constraints, code patterns, anti-patterns, and performance targets
  4. After ALL 30 files are written, read them as your operational constitution
  5. Only then proceed to PHASE 1

DO NOT skip, summarize, or batch-create these files. Each must be written
individually with full technical depth. These are your immutable rules.

─────────────────────────────────────────────────────────
SKILL FILE SPECIFICATIONS (WRITE EACH ONE IN FULL)
─────────────────────────────────────────────────────────

╔══════════════════════════════════════════════════════╗
║  DOMAIN A — INGESTION PIPELINES & HIGH-THROUGHPUT   ║
╚══════════════════════════════════════════════════════╝

FILE: .astraz_maps_skills/01_protobuf_telemetry.md
TITLE: Protobuf Telemetry Packet Parser
CONTENTS MUST INCLUDE:
  - Proto3 schema definition for SpatialCoordinatePacket
    (fields: asset_id, lat, lon, altitude, heading, speed, accuracy, timestamp_us, payload_version)
  - Elixir pattern matching on binary frames BEFORE heap allocation
  - Use of Protox or exprotobuf with precompiled schema modules
  - Stream parser that processes frames without accumulating full payloads in memory
  - Benchmarks: target < 500ns per packet deserialization
  - Anti-patterns: never use Jason/JSON for telemetry ingestion; never copy binaries unnecessarily
  - Error handling: malformed packets must be silently dropped with :telemetry event emitted
  - Code example: GenServer parse loop using binary pattern match + dispatch

FILE: .astraz_maps_skills/02_udp_tcp_ingestion_hooks.md
TITLE: UDP/TCP Socket Ingestion Listeners
CONTENTS MUST INCLUDE:
  - :gen_udp socket setup with active: :once mode to prevent mailbox flooding
  - :gen_tcp listener with Ranch or custom acceptor pool
  - Per-device process isolation: each connected device maps to one lightweight GenServer
  - Socket options: {recbuf, 8_388_608} (8MB), {sndbuf, 1_048_576}, {nodelay, true}
  - Dynamic supervisor for device process lifecycle management
  - Packet framing protocol for TCP (4-byte length prefix)
  - Load shedding: drop packets when process mailbox > 1000 messages
  - Code example: UDP listener loop with active: :once + handle_info dispatch

FILE: .astraz_maps_skills/03_spatial_rate_limiting.md
TITLE: Geospatial Coordinate Deduplication & Rate Limiter
CONTENTS MUST INCLUDE:
  - Haversine distance calculation in Elixir for < 1 meter delta detection
  - ETS-based last-position store for per-device delta comparison
  - Device frequency enforcement: configurable min_interval_ms per device class
  - Sliding window counter using ETS :update_counter for rate limiting
  - Spatial hash bucketing: geohash-based dedup for nearby coordinates
  - Code example: filter_coordinate/2 function with ETS lookup + haversine gate
  - Anti-patterns: never use a database query for per-packet rate limiting
  - Target: filter decision must complete in < 10 microseconds

FILE: .astraz_maps_skills/04_pipeline_backpressure.md
TITLE: GenStage/Broadway Backpressure Management
CONTENTS MUST INCLUDE:
  - Broadway pipeline topology: Producer → BatchProcessor → Acknowledger
  - Dynamic concurrency: adjust processor count based on system load via :scheduler_wall_time
  - Configurable batch sizes and batch timeouts per pipeline stage
  - Partition dispatch: route packets to processors by asset_id hash for ordering guarantees
  - Circuit breaker pattern: suspend pipeline if downstream DB write latency > 100ms
  - Dead letter queue: failed batches → ETS buffer → retry with exponential backoff
  - Metrics: pipeline throughput, batch latency, drop rate via :telemetry
  - Code example: Broadway module with handle_message and handle_batch callbacks

FILE: .astraz_maps_skills/05_metadata_stripping_privacy.md
TITLE: Edge Privacy Filter & Metadata Scrubber
CONTENTS MUST INCLUDE:
  - Allowlist-only field extraction from raw packets (drop all unrecognized fields)
  - Cryptographic asset_id verification: HMAC-SHA256 signature check before processing
  - PII fields that must NEVER reach storage: raw_ip, device_fingerprint, user_agent, IMEI
  - Coordinate fuzzing for non-premium tenants: add ±10m Gaussian noise
  - Timestamp normalization: convert all inputs to UTC microseconds
  - Audit log: emit sanitized events to separate :telemetry handler for compliance
  - Code example: scrub/1 function with struct pattern matching + cryptographic verify

╔══════════════════════════════════════════════════════╗
║  DOMAIN B — REAL-TIME GEOFENCING & SPATIAL CACHING  ║
╚══════════════════════════════════════════════════════╝

FILE: .astraz_maps_skills/06_ets_spatial_indexing.md
TITLE: ETS Hot-Path Spatial Index
CONTENTS MUST INCLUDE:
  - Table configuration: :set, :public, read_concurrency: true, write_concurrency: true
  - Key schema: {tenant_id, asset_id} → %{lat, lon, geohash, updated_at, metadata}
  - Geohash-indexed secondary table for spatial range queries (precision 7 = ~150m cells)
  - Atomic updates: :ets.update_element for partial field updates without full record replace
  - Memory budget: max 2GB per ETS spatial table; enforce with periodic :ets.info(:memory) check
  - Eviction policy: LRU eviction for stale assets (last_seen > 5 minutes)
  - Code example: upsert_position/3 and query_nearby/3 using ETS match specs

FILE: .astraz_maps_skills/07_distributed_horde_processes.md
TITLE: Horde Distributed Device Process Management
CONTENTS MUST INCLUDE:
  - Horde.DynamicSupervisor setup: distribution_strategy: Horde.UniformDistribution
  - Horde.Registry for global process lookup by {tenant_id, asset_id}
  - Process handover on node failure: via_name pattern + automatic restart on new node
  - Health check: heartbeat GenServer that pings all owned processes every 30 seconds
  - Anti-split-brain: use :net_kernel.monitor_nodes/1 to detect topology changes
  - Process state hydration on restart: fetch last known state from ETS or PostgreSQL
  - Code example: start_or_find_device_process/2 with Horde.Registry.lookup + fallback start

FILE: .astraz_maps_skills/08_quadtree_spatial_partitioning.md
TITLE: In-Memory Quadtree for Geofence Zones
CONTENTS MUST INCLUDE:
  - Quadtree node structure: {bbox, depth, children | leaf_zones}
  - Max depth: 20 levels (covers ~1 meter precision globally)
  - Dynamic split threshold: split leaf when zone_count > 16
  - Insertion: O(log N) zone registration with bounding box calculation
  - Query: O(log N) point-in-bounding-box candidates before exact PIP check
  - Serialization: binary term format (:erlang.term_to_binary) for fast ETS storage
  - Rebalancing: async rebalance task triggered when tree depth imbalance > 3 levels
  - Code example: Quadtree.query_candidates/2 returning zone list for PIP evaluation

FILE: .astraz_maps_skills/09_concurrent_boundary_evaluation.md
TITLE: Parallel Point-in-Polygon Geofence Evaluator
CONTENTS MUST INCLUDE:
  - Ray casting algorithm for convex and concave polygon PIP
  - Winding number algorithm for high-accuracy complex polygons
  - Task.async_stream for concurrent evaluation across zone candidates (max_concurrency: 32)
  - Result deduplication: a single coordinate may trigger multiple zone events
  - Event types: :entered, :exited, :dwelling (inside for > N seconds)
  - Dwell tracking: ETS-based per-device zone entry timestamps
  - Phoenix Channel broadcast for crossing events: push/3 with zone metadata
  - Code example: evaluate_zones/2 using Task.async_stream + reduce to event list

FILE: .astraz_maps_skills/10_persistent_spatial_cache.md
TITLE: WAL-Based Async Persistence for Spatial State
CONTENTS MUST INCLUDE:
  - Write-ahead log: ETS buffer → periodic flush to PostgreSQL (every 500ms or 10k records)
  - Flush strategy: batch INSERT ... ON CONFLICT DO UPDATE with EXCLUDED.*
  - Recovery on startup: hydrate ETS from PostgreSQL for last known positions
  - Checkpoint mechanism: mark flush position to avoid re-processing on crash recovery
  - Async flush GenServer: dedicated process for DB writes, never on the ingestion path
  - Backpressure from DB: if flush queue > 50k records, pause ingestion temporarily
  - Code example: SpatialPersistence.flush/1 GenServer with handle_continue for initial hydration

╔══════════════════════════════════════════════════════╗
║  DOMAIN C — HIGH-SPEED ROUTING & GRAPH PROCESSING   ║
╚══════════════════════════════════════════════════════╝

FILE: .astraz_maps_skills/11_rustler_nif_routing.md
TITLE: Rust NIF Pathfinding Engine (Dijkstra + A*)
CONTENTS MUST INCLUDE:
  - Rustler NIF setup: mix rustler_new with dirty_cpu scheduler for long computations
  - Graph representation in Rust: petgraph crate with NodeIndex, EdgeIndex
  - A* implementation with haversine heuristic for geographic accuracy
  - Bidirectional Dijkstra for symmetric graphs (2x faster than unidirectional)
  - Dirty NIF scheduling: use schedule_dirty_cpu for routes > 100ms to avoid BEAM scheduler blocking
  - Memory ownership: pass graph as Elixir binary Resource, retain in Rust Arc<Mutex<Graph>>
  - Error boundary: Rust panics must be caught (std::panic::catch_unwind) before returning to BEAM
  - Code example: Elixir wrapper module + Rust NIF function signature with ResourceArc

FILE: .astraz_maps_skills/12_concurrent_graph_partitioning.md
TITLE: Geographic Graph Partitioning Strategy
CONTENTS MUST INCLUDE:
  - Graph segmentation by H3 hexagonal grid (resolution 5 = ~250km² cells)
  - Boundary node identification: nodes shared between adjacent partitions
  - Partition cache: each region compiled to binary and stored in ETS
  - Cross-partition routing: 2-level hierarchy (local graph + highway abstraction layer)
  - Partition invalidation: subscribe to OSM diff feeds; invalidate only affected hexes
  - Lazy loading: load partition into memory on first route request, evict after 10min idle
  - Code example: GraphRegistry GenServer managing partition lifecycle + lazy load

FILE: .astraz_maps_skills/13_turn_by_turn_serialization.md
TITLE: Route Serialization & Real-Time Path Streaming
CONTENTS MUST INCLUDE:
  - Route step schema: {step_id, maneuver, street_name, distance_m, duration_s, geometry_polyline}
  - Polyline encoding: Google Encoded Polyline Algorithm at precision 5 (saves ~60% vs raw coords)
  - Phoenix Channel streaming: push steps in chunks of 50 to avoid large WebSocket frames
  - Delta updates: when route recalculates, send only diverging suffix (not full route)
  - Binary format: use :erlang.term_to_binary for internal caching; JSON only at API boundary
  - ETA calculation: rolling average of historical edge speeds per time-of-day bucket
  - Code example: RouteSerializer.encode_delta/2 producing minimal diff payload

FILE: .astraz_maps_skills/14_multi_modal_routing_graphs.md
TITLE: Multi-Modal Graph with Transport Mode Constraints
CONTENTS MUST INCLUDE:
  - Edge attribute bitmask for mode permissions: bit0=car, bit1=truck, bit2=walk, bit3=bike, bit4=industrial
  - Mode-specific edge weight functions: trucks penalize low bridges, pedestrians penalize highways
  - Graph construction: single unified graph with mode filter applied at query time (no graph copies)
  - Restriction types: turn restrictions, time-of-day restrictions, permit-only roads
  - Restriction encoding: conditional edge attributes evaluated at routing time
  - Mode switching: explicit transfer nodes for intermodal routes (e.g., park-and-ride)
  - Code example: RouteRequest struct with mode field + Rust NIF edge filter lambda

FILE: .astraz_maps_skills/15_live_traffic_weight_compaction.md
TITLE: Real-Time Traffic Weight Updates & Compaction
CONTENTS MUST INCLUDE:
  - Traffic update schema: {edge_id, speed_kmh, congestion_level, valid_until_timestamp}
  - Weight update path: UDP ingestion → ETS delta table → NIF graph hot-reload
  - Hot reload strategy: swap edge weight array in Rust Resource without rebuilding full graph
  - Compaction schedule: every 5 minutes, merge delta table into base graph weights
  - Stale weight cleanup: remove updates where valid_until < current time during compaction
  - Priority edges: emergency vehicle routing uses separate weight table with preemption
  - Code example: TrafficUpdater GenServer with periodic compaction via Process.send_after

╔══════════════════════════════════════════════════════╗
║  DOMAIN D — VECTOR PIPELINES, MAP TILES & RENDERING ║
╚══════════════════════════════════════════════════════╝

FILE: .astraz_maps_skills/16_pbf_vector_tile_parsing.md
TITLE: PBF/OSM Spatial Dataset Async Parser
CONTENTS MUST INCLUDE:
  - PBF file format: dense node blocks + way blocks + relation blocks
  - Elixir streaming parser using File.stream!/3 with chunk_size: 65536
  - Parallel block decoding: Task.async_stream with max_concurrency matching CPU count
  - Node deduplication via ETS during import to handle OSM node references in ways
  - PostGIS insertion: ST_GeomFromText for ways, batch INSERT 10k records/transaction
  - Import progress: :telemetry event every 100k entities for monitoring
  - Anti-patterns: never load entire PBF file into memory; never use a single-process decoder
  - Code example: PBFParser.stream_import/2 pipeline with Flow for parallel processing

FILE: .astraz_maps_skills/17_vector_tile_compression_mvt.md
TITLE: MVT Vector Tile Generation & Compression
CONTENTS MUST INCLUDE:
  - MVT spec compliance: Mapbox Vector Tile Specification v2.1
  - PostGIS query: ST_AsMVT + ST_AsMVTGeom with ST_TileEnvelope for coordinate clipping
  - Coordinate quantization: map to 0-4096 integer grid (MVT standard tile extent)
  - Geometry simplification at query time: ST_Simplify with tolerance = tile_extent / zoom_factor
  - Protobuf encoding: use VectorTile proto schema for final binary output
  - Gzip compression: always gzip MVT tiles before caching (30-70% size reduction)
  - Layer ordering: roads → buildings → labels → overlays (painter's algorithm)
  - Code example: TileGenerator.generate/3 with PostGIS pipeline + gzip compression

FILE: .astraz_maps_skills/18_dynamic_tile_caching.md
TITLE: Multi-Tier Tile Cache Architecture
CONTENTS MUST INCLUDE:
  - Cache tiers: L1 = ETS (hot, < 2GB), L2 = Redis (warm, TTL 24h), L3 = S3-compatible (cold, permanent)
  - Cache key: "tile:{z}:{x}:{y}:{layer_hash}:{style_hash}"
  - Cache-aside pattern: check L1 → L2 → L3 → generate → backfill all tiers
  - ETS tile store: :bag table, key = tile_key, eviction by LRU approximation (timestamp tracking)
  - Redis pipeline: use Redix.pipeline/2 for multi-tile prefetch
  - Invalidation: tile keys indexed by geographic bbox; OSM updates trigger targeted invalidation
  - Prefetch: on tile request for z/x/y, async prefetch 8 surrounding tiles
  - Code example: TileCache.get_or_generate/3 with tier fallthrough + async backfill

FILE: .astraz_maps_skills/19_spatial_layer_compositing.md
TITLE: Dynamic Spatial Layer Compositor
CONTENTS MUST INCLUDE:
  - Layer types: base_map, traffic, assets, geofences, custom_overlays, heatmaps
  - Layer visibility rules: zoom_level ranges, tenant permissions, feature flags
  - Compositing strategy: merge layer MVT tiles in Elixir binary operations (no image rendering)
  - Custom overlay injection: tenant-specific GeoJSON → converted to MVT layer at request time
  - Layer caching: composite layers cached separately from individual layers
  - Security: geofence layers filtered by tenant_id before compositing (never leak cross-tenant boundaries)
  - Code example: LayerCompositor.compose/3 with permission-filtered layer selection

FILE: .astraz_maps_skills/20_on_the_fly_simplification.md
TITLE: Dynamic Geometry Simplification NIF
CONTENTS MUST INCLUDE:
  - Douglas-Peucker algorithm implemented in Rust NIF for performance
  - Epsilon selection table: zoom_level → simplification_tolerance mapping
  - Visvalingam-Whyatt as alternative for smoother coastlines and paths
  - Input format: flat binary array of {lon, lat} pairs (avoids Elixir list overhead)
  - Output: simplified binary array + original_count/simplified_count metadata
  - Integration: called in MVT generation pipeline before ST_AsMVTGeom
  - Anti-patterns: never run Douglas-Peucker in Elixir for geometries > 1000 points
  - Code example: Simplifier NIF with zoom-adaptive epsilon + Rust implementation skeleton

╔══════════════════════════════════════════════════════╗
║  DOMAIN E — CLUSTERING, MESH NETWORKS & RESILIENCE  ║
╚══════════════════════════════════════════════════════╝

FILE: .astraz_maps_skills/21_libcluster_wireguard_mesh.md
TITLE: Automatic Node Discovery & Encrypted Cluster Mesh
CONTENTS MUST INCLUDE:
  - libcluster topology: Cluster.Strategy.Kubernetes for cloud; Cluster.Strategy.Gossip for bare metal
  - WireGuard interface setup: all inter-node traffic on wg0 interface (10.0.0.0/24 private)
  - Node naming convention: maps_engine@10.0.0.{N} with predictable IP assignment
  - Cookie management: BEAM cluster cookie from environment secret, never hardcoded
  - TLS distribution: optional Erlang TLS distribution over WireGuard for defense-in-depth
  - Health check endpoint: Phoenix /cluster/health returning node list + Horde registry size
  - Anti-patterns: never expose EPMD port (4369) to public internet
  - Code example: libcluster config in releases.exs + WireGuard config snippet

FILE: .astraz_maps_skills/22_horde_registry_synchronization.md
TITLE: Global Process Registry with Horde
CONTENTS MUST INCLUDE:
  - Horde.Registry vs :global vs :pg comparison and why Horde wins for dynamic clusters
  - Registration key design: {:device, tenant_id, asset_id} for namespaced lookups
  - Registry sync interval: Horde's delta_crdt sync every 100ms across nodes
  - Process lookup with fallback: Horde.Registry.lookup → start_child if not found
  - Anti-collision: use Horde.Registry.register with unique: true to prevent duplicates
  - Debugging: custom Registry.dump/0 function for admin visibility
  - Monitoring: :telemetry hook on process registration/deregistration events
  - Code example: DeviceRegistry module wrapping Horde.Registry with typed API

FILE: .astraz_maps_skills/23_crdt_spatial_replication.md
TITLE: CRDT-Based Distributed State Synchronization
CONTENTS MUST INCLUDE:
  - CRDT types used: LWW-Register (last-write-wins) for device metadata, OR-Set for active geofences
  - DeltaCRDT library integration (delta_crdt Hex package)
  - State categories requiring CRDT: tenant configurations, geofence definitions, route cache invalidation flags
  - Merge strategy: wall-clock timestamps for LWW; causality vectors for OR-Set
  - Anti-patterns: never use CRDTs for coordinate positions (use ETS hot path instead)
  - Persistence: CRDT state checkpointed to PostgreSQL every 60 seconds
  - Conflict resolution audit: log all CRDT merges with before/after state for debugging
  - Code example: GeofenceRegistry using DeltaCRDT with OR-Set add/remove operations

FILE: .astraz_maps_skills/24_cross_datacenter_delta_streaming.md
TITLE: Efficient Cross-Datacenter State Delta Streaming
CONTENTS MUST INCLUDE:
  - Delta stream format: {sequence_id, node_id, timestamp_us, operation, payload_binary}
  - Batching strategy: accumulate deltas for 50ms or 1000 events, whichever comes first
  - Compression: zstd compression for delta batches (better ratio than gzip for structured data)
  - Ordering guarantee: sequence_id monotonic per-node; recipients reorder before applying
  - Backpressure: receiver sends ACK every 100 batches; sender pauses if no ACK within 5s
  - Reconnect strategy: request missing sequence range on reconnect (gap fill)
  - Bandwidth budget: target < 10 Mbps inter-datacenter for 10,000 active devices
  - Code example: DeltaStreamer GenServer with batch accumulation + zstd compression

FILE: .astraz_maps_skills/25_split_brain_network_recovery.md
TITLE: Split-Brain Detection and Automatic Recovery
CONTENTS MUST INCLUDE:
  - Split-brain definition: two cluster partitions each believing they are the primary
  - Detection: monitor node_count vs expected_cluster_size; alert if diverged > 30 seconds
  - Quorum rule: minority partition (< 50% nodes) must self-demote to read-only mode
  - Recovery sequence: majority partition continues; minority resyncs from majority on rejoin
  - Data reconciliation: use CRDT merge for state reconciliation after partition heals
  - Horde behavior during split: each partition runs its own Horde supervisor; deduplicate on merge
  - Alerting: PagerDuty/Slack webhook on split-brain detection
  - Code example: ClusterMonitor GenServer with quorum check + auto-demotion logic

╔══════════════════════════════════════════════════════╗
║  DOMAIN F — SYSTEM HARDENING, SECURITY & COMPILATION ║
╚══════════════════════════════════════════════════════╝

FILE: .astraz_maps_skills/26_token_theft_and_idor_defenses.md
TITLE: Asymmetric Auth & IDOR Prevention
CONTENTS MUST INCLUDE:
  - JWT with RS256 (asymmetric): private key for signing, public key for verification
  - Token claims required: sub (asset_id), tid (tenant_id), exp, iat, jti (nonce for replay prevention)
  - IDOR prevention: all DB queries MUST include tenant_id in WHERE clause — enforced via Ecto scope
  - Database row-level security: PostgreSQL RLS policies as defense-in-depth
  - Token revocation: jti stored in ETS bloom filter; check on every request
  - Rate limiting: 100 auth attempts per IP per minute using :ex_rated or Hammer
  - Audit log: failed auth attempts logged with IP, user_agent, and attempted asset_id
  - Code example: TenantScope plug + Ecto query scope macros enforcing tenant isolation

FILE: .astraz_maps_skills/27_signature_based_abuse_mitigation.md
TITLE: Cryptographic Tile URL Signing & Anti-Scraping
CONTENTS MUST INCLUDE:
  - Tile URL format: /tiles/{z}/{x}/{y}?sig={hmac}&exp={unix_ts}&tid={tenant_id}
  - HMAC-SHA256 signature: sign("tile:{z}:{x}:{y}:{tenant_id}:{expiry}", secret_key)
  - Expiry window: 300 seconds (prevents URL sharing between sessions)
  - Per-tenant secret rotation: keys rotated every 24h; overlap period of 5 minutes
  - Batch tile request detection: > 500 tile requests/minute from single session = block
  - Honeypot tiles: embed unique invisible watermark per tenant for leak detection
  - Anti-patterns: never embed secrets in client-side JavaScript; never use GET params alone
  - Code example: TileSigner module + Plug for verification in the router pipeline

FILE: .astraz_maps_skills/28_multi_stage_compilation.md
TITLE: Minimal Secure Docker Multi-Stage Build
CONTENTS MUST INCLUDE:
  - Stage 1 (deps): elixir:1.16-otp-26 base, mix deps.get, compile deps
  - Stage 2 (rust): rust:1.78-slim, cargo build --release for NIFs
  - Stage 3 (build): copy compiled deps + rust .so files, mix release
  - Stage 4 (runtime): debian:bookworm-slim, copy release binary only
  - Final image must NOT contain: source code, mix.exs, node_modules, .git, test files
  - Non-root user: run as uid 1000, never as root in production
  - Secret injection: environment variables via Docker secrets or Vault agent, never baked in image
  - Security scan: integrate trivy or grype in CI pipeline before push
  - Dockerfile template with all 4 stages provided in full

FILE: .astraz_maps_skills/29_real_time_analytics_isolation.md
TITLE: Async Metrics & Analytics Decoupled from Critical Path
CONTENTS MUST INCLUDE:
  - :telemetry event taxonomy: [:astraz, :telemetry, :ingested], [:astraz, :route, :calculated], etc.
  - Handler isolation: analytics handlers run in separate OTP application, linked but not blocking
  - Metric types: counter, gauge, histogram with appropriate tags (tenant_id, node_id, mode)
  - Prometheus exporter: :prom_ex or :prometheus_ex with custom collectors
  - Grafana dashboards: define standard dashboard JSON for route latency, tile cache hit rate, ingestion rate
  - Buffered writes: analytics events batched 1000ms before writing to TimescaleDB
  - Anti-patterns: never call analytics.record() synchronously inside route handler
  - Code example: :telemetry.execute/3 call sites + async handler module

FILE: .astraz_maps_skills/30_chaos_engineering_fault_injection.md
TITLE: Chaos Engineering & Fault Injection Test Suite
CONTENTS MUST INCLUDE:
  - Fault types: random process kill, network partition simulation, corrupted packet injection, DB connection exhaustion
  - Elixir chaos module: ChaosMaps.inject/2 with {:kill_process, name}, {:delay_messages, ms}, {:corrupt_packet}
  - Integration with ExUnit: ChaosCase module wrapping test with automatic fault injection
  - Partition simulation: :net_kernel.disconnect_node/1 + reconnect after N milliseconds
  - Recovery assertions: after fault injection, system must recover within defined SLA (< 10 seconds)
  - Steady-state hypothesis: define baseline metrics; chaos test must not degrade baseline > 5%
  - CI integration: chaos tests run in isolated environment nightly, not on every PR
  - Code example: ExUnit test using ChaosCase with kill + recovery assertion

═══════════════════════════════════════════════════════════
PHASE 1 — PROJECT STRUCTURE BOOTSTRAP
═══════════════════════════════════════════════════════════

After all 30 skill files are written, create the following directory structure:

astraz_core_maps/
├── .astraz_maps_skills/          # 30 skill files (already created)
├── apps/
│   ├── astraz_core/              # Core business logic OTP app
│   │   ├── lib/
│   │   │   ├── telemetry/        # Ingestion pipeline modules
│   │   │   ├── geofencing/       # Spatial index + PIP evaluator
│   │   │   ├── routing/          # Route calculation + graph management
│   │   │   ├── clustering/       # Horde + libcluster setup
│   │   │   └── security/         # Auth + IDOR defenses
│   │   └── test/
│   ├── astraz_tiles/             # Map tile generation OTP app
│   │   ├── lib/
│   │   │   ├── parser/           # PBF parser
│   │   │   ├── generator/        # MVT tile generator
│   │   │   └── cache/            # Multi-tier tile cache
│   │   └── test/
│   └── astraz_web/               # Phoenix web layer
│       ├── lib/
│       │   ├── channels/         # Phoenix Channels for real-time
│       │   ├── controllers/      # REST API controllers
│       │   ├── plugs/            # Auth + tenant scoping plugs
│       │   └── router.ex
│       └── test/
├── native/
│   └── astraz_routing_nif/       # Rust NIF crate
│       ├── src/
│       │   ├── lib.rs
│       │   ├── astar.rs          # A* implementation
│       │   ├── dijkstra.rs       # Dijkstra implementation
│       │   └── simplifier.rs     # Douglas-Peucker
│       └── Cargo.toml
├── priv/
│   ├── migrations/               # Ecto migrations
│   └── schemas/                  # Protobuf .proto files
├── config/
│   ├── config.exs
│   ├── dev.exs
│   ├── prod.exs
│   └── releases.exs
├── docker/
│   ├── Dockerfile                # Multi-stage production build
│   └── docker-compose.yml        # Local dev environment
├── .github/
│   └── workflows/
│       ├── ci.yml                # Test + lint pipeline
│       └── chaos.yml             # Nightly chaos tests
├── mix.exs                       # Umbrella project config
└── README.md

═══════════════════════════════════════════════════════════
PHASE 2 — IMPLEMENTATION ORDER (STRICTLY FOLLOW)
═══════════════════════════════════════════════════════════

Implement in this exact sequence. Do NOT skip steps.
Each step must have passing tests before proceeding.

STEP 1: Mix umbrella project + dependency declaration (mix.exs)
  Dependencies must include: phoenix, ecto, ecto_sql, postgrex, broadway,
  horde, libcluster, rustler, protox, ex_rated, hammer, prom_ex, delta_crdt

STEP 2: Ecto schemas + migrations
  - devices(id, tenant_id, asset_id, last_lat, last_lon, last_seen_at, metadata)
  - geofences(id, tenant_id, name, geometry::geometry(Polygon,4326), properties)
  - routes(id, tenant_id, origin, destination, waypoints, computed_at, graph_version)

STEP 3: Protobuf schema (.proto files) + Elixir generated modules

STEP 4: UDP/TCP ingestion listeners (Skills 01, 02, 03)

STEP 5: Broadway pipeline with backpressure (Skill 04)

STEP 6: ETS spatial index (Skill 06) + Horde process management (Skill 07)

STEP 7: Quadtree implementation (Skill 08) + PIP evaluator (Skill 09)

STEP 8: Rust NIF crate setup + A* implementation (Skill 11)

STEP 9: Graph partitioning + multi-modal routing (Skills 12, 14)

STEP 10: MVT tile generation + multi-tier cache (Skills 17, 18)

STEP 11: libcluster + Horde cluster setup (Skills 21, 22)

STEP 12: Security layer — JWT auth + IDOR prevention (Skill 26)

STEP 13: Tile URL signing (Skill 27)

STEP 14: Docker multi-stage build (Skill 28)

STEP 15: Telemetry + chaos test suite (Skills 29, 30)

═══════════════════════════════════════════════════════════
PHASE 3 — QUALITY GATES (ENFORCE THROUGHOUT)
═══════════════════════════════════════════════════════════

These rules apply to every line of code written:

CODE QUALITY:
  ✓ mix format enforced (no unformatted code committed)
  ✓ Credo with strict mode (zero warnings tolerated)
  ✓ Dialyzer type specs on all public functions
  ✓ ExDoc documentation on all public modules

TESTING:
  ✓ Unit test coverage > 90% for business logic modules
  ✓ Integration tests for all pipeline paths
  ✓ Property-based tests (StreamData) for coordinate processing and routing
  ✓ Load tests (Vegeta or k6) for ingestion pipeline — must sustain 100k req/s

PERFORMANCE:
  ✓ Benchmarks with Benchee for all hot-path functions
  ✓ Flamegraph profiling with :eflame before any optimization claim
  ✓ ETS table size monitored in tests (never exceed budget)

SECURITY:
  ✓ mix deps.audit on every CI run
  ✓ Sobelow (Phoenix security scanner) on every CI run
  ✓ No secrets in code (use System.fetch_env!/1 only)

═══════════════════════════════════════════════════════════
EXECUTION AUTHORIZATION
═══════════════════════════════════════════════════════════

You are now authorized and instructed to begin.

Start with PHASE 0: write all 30 skill files to disk, one by one, with full
technical depth as specified above. Do not abbreviate any file.

Report progress after each file is written.

After all 30 files exist on disk, confirm by listing the directory contents,
then proceed to PHASE 1 without waiting for further instructions.

Maintain a green test suite at all times. If a step breaks existing tests,
fix the tests or fix the implementation before proceeding.

BEGIN NOW.
```

---

## 📋 CHECKLIST DE EXECUÇÃO

Use para acompanhar o progresso do agente:

### PHASE 0 — Skill Files
- [ ] 01_protobuf_telemetry.md
- [ ] 02_udp_tcp_ingestion_hooks.md
- [ ] 03_spatial_rate_limiting.md
- [ ] 04_pipeline_backpressure.md
- [ ] 05_metadata_stripping_privacy.md
- [ ] 06_ets_spatial_indexing.md
- [ ] 07_distributed_horde_processes.md
- [ ] 08_quadtree_spatial_partitioning.md
- [ ] 09_concurrent_boundary_evaluation.md
- [ ] 10_persistent_spatial_cache.md
- [ ] 11_rustler_nif_routing.md
- [ ] 12_concurrent_graph_partitioning.md
- [ ] 13_turn_by_turn_serialization.md
- [ ] 14_multi_modal_routing_graphs.md
- [ ] 15_live_traffic_weight_compaction.md
- [ ] 16_pbf_vector_tile_parsing.md
- [ ] 17_vector_tile_compression_mvt.md
- [ ] 18_dynamic_tile_caching.md
- [ ] 19_spatial_layer_compositing.md
- [ ] 20_on_the_fly_simplification.md
- [ ] 21_libcluster_wireguard_mesh.md
- [ ] 22_horde_registry_synchronization.md
- [ ] 23_crdt_spatial_replication.md
- [ ] 24_cross_datacenter_delta_streaming.md
- [ ] 25_split_brain_network_recovery.md
- [ ] 26_token_theft_and_idor_defenses.md
- [ ] 27_signature_based_abuse_mitigation.md
- [ ] 28_multi_stage_compilation.md
- [ ] 29_real_time_analytics_isolation.md
- [ ] 30_chaos_engineering_fault_injection.md

### PHASE 1 — Estrutura de Projeto
- [ ] Mix umbrella criado
- [ ] Apps: astraz_core, astraz_tiles, astraz_web
- [ ] Native: Rust NIF crate
- [ ] Docker: Dockerfile multi-stage + docker-compose

### PHASE 2 — Implementação (por step)
- [ ] Steps 1–5: Pipeline de ingestão
- [ ] Steps 6–7: Geofencing em tempo real
- [ ] Steps 8–10: Roteamento via Rust NIF
- [ ] Steps 11–13: Tiles MVT + cache
- [ ] Steps 14–15: Cluster + Horde
- [ ] Steps 16–17: Segurança
- [ ] Steps 18–19: Docker + produção
- [ ] Step 20: Observabilidade + chaos tests

---

## 🔧 PRÉ-REQUISITOS DO AMBIENTE

Antes de rodar o prompt, garanta que seu ambiente tem:

```bash
# Elixir + Erlang
elixir --version  # >= 1.16
erl -version      # OTP >= 26

# Rust (para NIFs)
rustup --version  # >= 1.78
cargo --version

# PostgreSQL com PostGIS
psql --version    # >= 16
psql -c "SELECT PostGIS_Version();"

# Docker (para build de produção)
docker --version  # >= 25

# Ferramentas de qualidade (instalar globalmente)
mix escript.install hex credo
mix escript.install hex sobelow
```

---

## 🎯 SUBSISTEMAS PRIORITÁRIOS PARA DETALHAR

Após a geração inicial, escolha um subsistema para aprofundar:

| Subsistema | Skill Files | Complexidade | Impacto |
|---|---|---|---|
| **Geofencing em tempo real** | 06, 07, 08, 09 | Alta | Crítico |
| **Pathfinding via Rust NIF** | 11, 12, 14 | Muito Alta | Crítico |
| **Pipeline de ingestão UDP** | 01, 02, 03, 04 | Alta | Crítico |
| **MVT Tile Engine** | 16, 17, 18, 20 | Alta | Alto |
| **Cluster distribuído** | 21, 22, 23, 25 | Muito Alta | Alto |
| **Segurança & Hardening** | 26, 27, 28 | Média | Alto |

---

*Documento gerado pela Astraz Studio — Motor de Mapas de Alta Performance*
*Stack: Elixir/Phoenix + Rust NIFs + PostgreSQL/PostGIS + BEAM Cluster*
