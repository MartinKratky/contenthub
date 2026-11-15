# ContentHub: Design checkpoint 1.0

Version: 1.0\
Status: Agreed product direction; concrete interface spellings and operating defaults proposed below\
Predecessor: `designs/design_0.9`, preserved unchanged\
Working name: `contenthub`

ContentHub is a local, connection-authorized content retrieval service with per-agent acknowledged-delivery deduplication. It is not a content archive. Agents discover upstream objects, explicitly fetch content, manage their own subscriptions, and inspect their delivery ledger through MCP. Content exists only in bounded runtime memory; persistent records describe subscriptions and retrieval attempts, never payloads.

This is a standalone replacement design, not a migration or an implementation. The accompanying `db-structure.dbml` proposes two tables: `subscriptions` and `delivery_attempts`. The table name `delivery_attempts` emphasizes that errors and unacknowledged responses are present alongside acknowledged deliveries.

## Decisions and scope

- **Authorization:** Owner-configured agents receive grants to connections. Agents may modify their own subscriptions but cannot edit connections, credentials, grants, or their own identity.
- **Subscriptions:** One agent owner, one connection, one selector type, one shared content request/representation, and one default lookback. A selector may accept several channels, several playlists, or several timelines when supported; a Gmail query can express broad selection. Do not mix channel and playlist selector types inside one subscription.
- **Direct discovery:** Discover supported targets within an authorized connection and explicit time window. Discovery returns references/metadata, not content-delivery receipts.
- **Direct fetching:** Check the agent's retained acknowledgments. If already acknowledged, return an `already_served` notice unless explicitly overridden; an override performs a new fetch and requires a new acknowledgment.
- **Subscription fetching:** Discover, normalize identities, deduplicate within the request, exclude acknowledged identities, and fetch remaining content. Acknowledgments apply across all of an agent's subscriptions and direct requests.
- **Representation:** Request raw or cleaned output. Either acknowledged representation counts as served for subscription and direct-request checks; different material content requests, such as transcript languages, remain different identities.
- **Persistence:** Store subscriptions and attempt/receipt metadata only. No content archive, raw/processed tables, durable execution queue, or content sharing across agents.
- **Delivery:** Serve one item at a time or bounded batches. Continue fetching subsequent batches only within bounded prefetch and per-agent memory budgets.
- **Acknowledgment:** A durable ledger entry becomes suppression evidence only when acknowledged. Successful fetching, response preparation, delivery time, and errors alone do not suppress a later request.
- **Errors:** Fetch/processing outcomes are inspectable diagnostics, not persistent retry eligibility. A new request for an unacknowledged item may try again after current authorization, target, capability, and resource checks.
- **Execution:** One service owns shared bounded asyncio discovery and fetch queues, with independently configurable consumer concurrency. Cleaning is part of the fetch task, not a separate process or queue.

Intended sources include Gmail, YouTube, X, and WhatsApp, subject to the capabilities of implemented adapters and authorized connections. Supporting an identity or raw byte representation does not promise an adapter for every source, private target, or large-media format.

## What changes from 0.9

The durable `items`, `subscription_items`, `raw_payloads`, and `item_contents` archive model is removed. There are no current-content pointers, saved-input corrections, immutable published-result chains, content-search indexes, or saved-content retrieval snapshots. Previously archived data must not be deleted automatically as part of adopting this design.

Subscriptions now persist editable collection rules, rather than being a YAML-only identity registry. They no longer grant access to content. Connection grants define access, and runtime subscription edits apply to subsequent requests without an owner restart.

The durable fetch-failure counter and exhausted-allowance policy are superseded. Errors do not automatically block future requests; no old failure counter is migrated into retry eligibility. Acknowledgment-based deduplication replaces archive-based reuse: eligible fresh requests fetch upstream again, not from disk or another agent's memory.

Retain the six-part canonical identity and SHA-256 collision checking. Retain local MCP/CLI communication, one service owner, disposable execution, explicit windows, structured adapters, and bounded async execution. No fallback extractor is introduced.

## Authorization and owner configuration

### Agent and connection identity

Proposed configuration placement is an owner-managed file containing stable agent IDs, stable connection IDs, connection providers/credential references, agent-to-connection grants, and worker/delivery limits. Labels may change without changing IDs; IDs must not be generated anew at startup or reassigned to unrelated agents/accounts. Secrets remain outside both configuration values and the application database.

MCP starts bound to an owner-selected agent identity. The worker authenticates that local session and derives grants itself; an agent cannot pass a different `agent_id` or an allowed-connection list as authority. Local IPC must restrict access to the appropriate host user. This is application-level isolation, not a defense against an agent with unrestricted access to the host's credentials and files.

Every discovery, fetch, subscription mutation, continuation, acknowledgment, and ledger query checks agent ownership and current connection authorization. A guessed item key, delivery ID, cursor, or subscription ID is never a grant. Reading a receipt does not grant access to fetch its item.

Owner connection/grant configuration initially remains restart-required. Restart invalidates old runtime sessions and cursors; no stale client can retain old permissions. Subscriptions whose owner or connection is absent/ungranted remain stored but unusable, with an explicit configuration/authorization error rather than silent deletion.

### Connections and upstream identity

A connection is an authorized execution route with declared discovery/fetch capabilities, not an item-identity component. It may expose multiple selector types, but each subscription chooses exactly one. A connection's adapter may internally use different provider components; agents do not select arbitrary executables or credentials.

Resolve private namespaces from verified stable target account/mailbox/tenant identity and any required narrower boundary. Public content uses the canonical empty namespace where the adapter contract permits it; unresolved private identity is an error, never public. Verify the target on discovery and fetch, including after reauthentication.

Two authorized connections to the same upstream content may produce the same item identity. They do not share agent content buffers or permissions. Changing a connection to an unrelated private account requires a new connection ID; renaming or credential rotation for the same verified target does not. Ledger queries are owner-scoped and filtered to currently granted connections; deduplication may internally recognize the same verified identity through another granted route without exposing that route's private diagnostics.

## Subscriptions

Persist a stable ID, owner agent ID, name, connection ID, selector type/arguments, content kind/options, representation, lookback duration, and creation/update times. Names are unique per agent, not globally. The selector arguments hold only validated selection criteria, not credentials, code, historical provider state, or content.

Examples of single-selector subscriptions are:

| Selector type | Selector arguments | Shared request |
|---|---|---|
| YouTube channels | Several channel IDs | English transcript, cleaned, seven-day lookback |
| YouTube playlists | Several playlist IDs | English transcript, cleaned, seven-day lookback |
| Gmail query | One query matching several senders/labels | Message body, cleaned, seven-day lookback |
| X timelines | Several supported timeline targets | Post content, raw, configured lookback |

These are illustrative selector names and values, not promises of implemented provider support. Different selector types, connections, content options, representations, or lookbacks require separate subscriptions.

### Editing and lifecycle

Agents list, create, update, and delete only their own subscriptions using currently granted connections and supported options. Validate updates transactionally. A request snapshots its subscription definition on admission; edits affect later requests, not work already accepted. No durable settings revisions or subscription-version table is required.

Deleting a subscription does not delete ledger records, revoke a connection grant, or cancel admitted work implicitly. The ledger keeps the originating subscription ID as historical metadata without a live foreign key. Old subscription IDs are not reused. An explicit request-cancel operation may discard runtime work without fabricating acknowledgments.

### Pull-driven first version

Use pull-driven collection initially: `fetch_new(subscription_id)` starts discovery/fetching. There is no automatic polling, startup scan, or prefetch merely because a subscription exists. This is the simplest proposed operating default for the non-archive service; scheduled collection is deferred rather than assumed from 0.9.

Absent an explicit override, fix `until` to request admission time and `since` to `until - lookback`. An explicit timezone-aware window may replace those bounds. A wider window does not bypass acknowledgments unless the caller explicitly sets the repeat override.

Multiple subscriptions can be requested together, with ownership/grants checked separately and their snapshots/windows disclosed. Deduplicate equivalent item identities across those inputs. If requests for the same identity disagree about raw/cleaned representation, require an explicit common representation or separate requests; do not silently discard a requested representation.

## Item identity and receipt lookup

An item is the requested content object, not a fetch attempt, delivery, subscription, or stored archive row. Compute its identity from:

```json
{
  "source": "youtube",
  "upstream_kind": "video",
  "namespace": "",
  "source_id": "video_123",
  "content_kind": "transcript",
  "options_json": {"language": "en"}
}
```

```text
item_key = sha256(canonical_json(normalized_identity).encode("utf-8")).hexdigest()
```

- **Included:** Source, upstream object kind, verified namespace, external ID, requested content kind, and canonical material options.
- **Excluded:** Agent, connection/provider route, subscription, discovery method, raw/cleaned representation, parser version, batching, retry, dates of delivery, and repeat override.
- **Normalization:** Normalize equivalent defaults and option keys centrally; preserve array order where meaningful. Define stable serialization and test vectors before implementation. Hash structured canonical JSON, not concatenated strings.
- **Collision protection:** Compare all six identity components on a matching hash. A mismatch is an integrity error, not a merge. Retain full identity metadata on attempts; canonicalization changes require explicit compatibility handling.
- **No item registry:** Ledger attempt IDs identify individual operations. No separate archive item UUID or `items` table is required.

A video and its English/German transcripts are separate requested items. Raw and cleaned forms of the same English transcript share an item key. Different private mailboxes remain separate even if an upstream external ID is identical; channel versus playlist discovery does not distinguish the same public video.

Suppress an ordinary request only when a retained `delivery_attempts` row matches the current agent and full item identity and has non-null `acknowledgement_time`. Do not filter this check by subscription, connection label, or representation. The row index is nonunique because repeated attempts are intentional.

Lookup happens only after the chosen route and target have passed authorization and identity validation. Recheck acknowledgment before starting queued network work; a receipt that arrives while another request is pending may make that work unnecessary. Already in-flight requests may still overlap: cross-request exactly-once fetching is not promised.

## Discovery and fetching contracts

### Standalone discovery

Expose capability-driven methods such as channel, playlist, timeline, and query discovery, or a typed generic discovery method. The exact MCP spelling is an implementation choice. Inputs include an authorized connection, supported selector, and explicit time window; opaque provider pagination stays in memory.

Return bounded pages of upstream references, dates, and useful discovery metadata. Discovery alone creates no delivery ledger entry and does not mark content acknowledged. References must be validated against the requested target; a provider-supplied URL is not authority to access arbitrary hosts or accounts.

Each adapter defines its source-date semantics, available coverage, precision, ordering, and treatment of unknown dates. Use half-open windows where supported: `since <= source_date < until`. Translate coarse upstream filtering and apply precise filtering when dates permit; disclose unknown coverage rather than invent dates or silently claim completeness. There is no durable discovery-date fallback or stored retrieval-date index.

### Direct fetching

Accept one or more explicit upstream/request identities through authorized connections, plus representation, delivery mode, and `allow_repeat` (proposed spelling, default false). If acknowledgment exists, return `already_served` with permitted prior timestamps/representation and instructions for the explicit override; perform no fetch and create no fake attempt for a skipped item.

With `allow_repeat=true`, perform a fresh upstream fetch even if prior content remains in memory. Create a new attempt and require acknowledgment again. A new failed or unacknowledged forced attempt never erases an earlier acknowledged record.

### Subscription fetching

For each admitted subscription snapshot, discover references, form complete requested-item identities, deduplicate them, check acknowledgments, and enqueue eligible fetching. Equivalent discoveries in one request should not duplicate fetching or delivery even before acknowledgment.

Maintain a bounded per-request seen-identity set. If the configured scan/item bound would be exceeded, stop with explicit partial coverage and a request to narrow the window rather than evict identities silently or grow without limit. This bound is separate from delivery batch size. A provider page token is not a durable restart cursor.

A multi-target selector is complete only after every required target scan completes. Report partial discovery and per-target errors while still allowing successful targets to supply content. Do not infer a total result count before discovery actually establishes it.

### Fetch task includes preparation

One fetch task holds its concurrency slot while it:

1. Revalidates authorization, target identity, capability, acknowledgment override, and resource limits.
2. Fetches the complete required upstream response or bundle into bounded memory.
3. Returns raw content or invokes a compatible parser for cleaned content using the requested identity/options as explicit context.
4. Validates required output, records a terminal attempt outcome, and places deliverable content in the requesting agent's bounded buffer.

Raw output means adapter-defined payload representation, not authentication headers, transport credentials, or unrestricted diagnostic dumps. Define each adapter's raw packaging and media type. Binary/multipart delivery requires an explicit transport contract and size bounds; the first implementation may reject unsupported representations rather than silently truncate or write them to disk.

There is no separate processing queue/process, saved raw input, fallback extractor, durable raw/cleaned lifecycle, or correction operation. Cleaned requests fail explicitly when parsing/validation fails. Fast parsing can run inline; blocking SDKs or expensive synchronous preparation use bounded offloading and must respect shutdown. No content spool files or hidden disk cache are introduced.

On success, retain only the representation/buffers needed for delivery and release unnecessary raw intermediates. On failure, release content buffers and retain sanitized diagnostics only. Do not log payloads, message bodies, tokens, cookies, or credentials.

## Shared execution and memory limits

```text
Agent-bound MCP / owner CLI
              |
       admission + validation
              |
      +-------+-----------------------+
      |                               |
standalone/subscription discovery   direct fetch
      |                               |
bounded discovery queue              |
      |                               |
async discovery consumers            |
      |                               |
references or normalize/dedup/receipt check
                                      |
                              bounded fetch queue
                                      |
                              async fetch consumers
                              (fetch + preparation)
                                      |
                           agent-owned delivery buffers
                                      |
                            MCP batches / continuation
                                      |
                         acknowledgment -> durable ledger
```

One service process owns both stages, all runtime request state, and database access coordination. “Worker” denotes a managed stage with configurable async consumer concurrency, not a required OS process. Separate MCP clients connect locally; they do not directly manipulate asyncio queues.

- **Concurrency:** Independently configure discovery and fetch concurrency. Add per-connection limits/pacing where required by the adapter. No multiprocessing service or message broker initially.
- **Queue bounds:** Bound discovery and fetch queue sizes. Admission backpressure must not accumulate unbounded blocked enqueue tasks or page results outside queues.
- **Delivery bounds:** Bound per-agent/request buffered bytes, unacknowledged batches, payload size, prefetch batches, scan identities, and idle lifetime. Item counts alone are insufficient.
- **Fairness:** Use per-agent/request admission caps and page-sized discovery work. Do not let a stalled agent monopolize all consumers while waiting indefinitely for delivery space; schedule only work with reserved downstream capacity.
- **Capacity reservation:** Reserve bounded delivery capacity before a fetch begins; reject oversized payloads and release reservations on failure/cancellation. Avoid holding every fetch slot while waiting for an agent to acknowledge.
- **Isolation:** Share queue infrastructure only. Task results and continuations belong to one agent/request, with no cross-agent content reuse or shared discovery-result cache initially.
- **Database safety:** No network or acknowledgment wait inside a write transaction. Async tasks must not share mutable database sessions. Prevent accidental duplicate service owners.

A normal MCP wait timeout returns available batches or progress without immediately cancelling accepted background work. Idle expiration, explicit cancellation, client abandonment policy, and shutdown eventually release bounded runtime state. A new request for an unacknowledged item can fetch again; continuation of the same live request serves its buffered content without creating a new attempt.

## Delivery, continuation, and acknowledgment

### Batches

Support `one_by_one` and `batch` delivery modes (proposed names). One-by-one caps each batch at one item; batch mode caps it at the connection's configured item count, also respecting byte/token limits. For multi-connection requests, keep batches connection-specific so limits remain unambiguous.

Return an opaque runtime request/batch/cursor identity, item delivery IDs, bounded content, per-item outcomes, completion status, and continuation instructions. A delivery ID is the corresponding durable attempt ID, not merely an item key. Separate `discovery_complete`, pending fetch/preparation work, and buffered content availability; do not claim “no more items” before discovery and admitted fetching finish.

Oversized items use explicit content continuation when supported, otherwise a size error. Only complete-item delivery is eligible for acknowledgment. Start serving ready batches while later pages/items are still being discovered/fetched; no global chronological ordering across unfinished scans is promised.

### Acknowledgment semantics

An agent acknowledges completed delivery IDs, individually or as an explicit list for a batch. Validate all IDs against the bound agent, current authorization, successful preparation, and complete delivery; duplicate acknowledgments are idempotent and preserve the original time. A batch acknowledgment covers only its listed fully delivered items, not future work or failed entries.

Update the ledger transactionally before reporting acknowledgment success and then release acknowledged content. If the database write fails, report failure and do not treat the item as acknowledged; the agent can retry. A lost acknowledgment response is safe to retry by delivery ID.

An acknowledgment is an explicit caller declaration, not proof the model read or understood content. Do not mark discovery, fetching, the first chunk, or mere timeout as acknowledgment.

### Delivery timestamp and restart

`delivery_time` records the server's complete-delivery handoff attempt, not proof of receipt. Persist it immediately before the final response/chunk handoff so an immediate acknowledgment can be validated. Database writes and network delivery cannot be one atomic transaction: a failed handoff may leave a delivery timestamp, but without acknowledgment it does not suppress redelivery.

Durable delivery IDs allow late acknowledgment after service restart or runtime-buffer expiration if the complete-delivery record remains and the agent is still authorized. Old batch cursors cannot resume after restart; the caller must acknowledge known delivery IDs directly or start a new request. If the record has been purged, return an explicit expired/unknown-delivery result and do not reconstruct acknowledgment state.

Unacknowledged buffers may expire to protect memory. Expiration does not acknowledge anything and does not necessarily delete the diagnostic ledger row. Only acknowledged retained rows suppress later requests.

## Delivery ledger and error policy

Persist one row per terminal item retrieval attempt. Begin the network attempt in memory; after a terminal fetch/preparation outcome, insert a metadata row with its attempt-start time. A hard crash before that commit can leave no record; the ledger is best-effort operation history, not exactly-once accounting or a recoverable job list.

| Field/group | Meaning |
|---|---|
| `id`, `agent_id`, `connection_id` | Unique delivery/attempt ID, requesting agent, actual authorized route |
| `subscription_id` | Optional originating subscription ID; historical association, not permission |
| Six identity fields and `item_key` | Complete requested-item identity and indexed SHA-256 key |
| `representation` | `raw` or `cleaned`; recorded but excluded from identity |
| `attempted_at` | Time this actual item attempt began |
| `fetching_status` | `fetched` or `error`; errors classified separately |
| Fetch error fields | Sanitized fetch error code/message when fetching failed |
| Processing error fields | Sanitized preparation error when fetching succeeded but cleaned/transport preparation failed |
| `delivery_time` | Nullable complete-response handoff-attempt time |
| `acknowledgement_time` | Nullable first successful explicit acknowledgment time |

If several subscription inputs discover the same item within one request, use the first validated originating subscription ID as diagnostic context; exact multi-origin history is not required. It does not restrict agent-wide deduplication.

Only successful fetch/preparation can acquire `delivery_time`, and acknowledgment requires that timestamp. `fetching_status=fetched` alone does not imply clean output or delivery. Raw deliveries normally have no processing error; transport-preparation errors remain distinguishable from upstream fetch failure.

Permanent-looking errors, authentication failures, rate limits, and processing failures are visible but do not create durable “never retry” flags or failure counters. Current permission/account verification and capability validation still apply. Connection pacing and bounded request-local transient retries may be configured; they are runtime controls, not ledger-driven eligibility. Specific retry defaults remain implementation choices.

There are no ledger rows for standalone discovery results, acknowledged-item skips, or pre-admission permission/validation rejections. Report those in the current response/runtime status. Partial/cancelled network attempts need not become synthetic durable work records.

### Agent inspection

Provide a read-only, bounded, paginated ledger query scoped to the authenticated agent and authorized connections. Allow filters for item identity, originating subscription, connection, representation, fetch/processing outcome, acknowledgment state, and time ranges. Return metadata and sanitized diagnostics only.

Use stable ordering such as `(attempted_at, id)` with a runtime cutoff; disclose that purges can remove rows between pages. Do not permit arbitrary SQL, cross-agent lookup, direct timestamp editing, or bulk “mark everything seen.” Acknowledgment is the only agent operation that changes acknowledgment state.

## Receipt retention and deletion

Retention bounds deduplication history, not content storage. Purging a row removes its evidence of acknowledgment; an otherwise eligible item can be fetched/delivered again. This is intentional and is not an authorization bypass.

Base acknowledged-row retention on `acknowledgement_time`, not upstream publication time. A year-old post fetched and acknowledged today should remain remembered for the retention period. Failed or unacknowledged rows can use a separate age cutoff based on attempt time.

A month of receipts for a week-long lookback is a reasonable configurable example, not proof of permanent deduplication safety or an approved universal default. Wider windows, changed selectors, and resurfaced upstream content can produce repeats after purge. Because receipts are agent-wide, do not delete them based only on one subscription's lookback or deletion.

Initially expose purge as an owner operation with explicit scope, cutoff, dry-run counts, and confirmation. Serialize purge with acknowledgment and protect active deliveries from cleanup while they are live; after buffer expiry, late acknowledgment remains possible only while its row remains. No autonomous agent deletion of ledger evidence is required by this checkpoint.

## Proposed interfaces

Exact names and argument shapes are implementation proposals. The behavioral separation, authorization, identity, and acknowledgment rules above are normative.

```text
list_connections()
get_connection_capabilities(connection_id)

discover(connection_id, selector_type, selector, since, until, cursor?)
fetch_items(connection_id, items, representation, delivery_mode, allow_repeat=false)
fetch_new(subscription_ids, window_override?, delivery_mode, allow_repeat=false)
get_request_status(request_id)
continue_delivery(cursor)
acknowledge(delivery_ids)
cancel_request(request_id)

list_subscriptions()
create_subscription(connection_id, name, selector_type, selector,
                    content_kind, options, representation, lookback_seconds)
update_subscription(subscription_id, changes)
delete_subscription(subscription_id)

list_delivery_attempts(filters, cursor?, limit?)
```

Provider-specific discovery methods may wrap the typed discovery contract. Clients cannot choose executables, filesystem paths, unrestricted network destinations, credentials, or another agent identity. Full payloads are treated as untrusted evidence, not instructions.

Proposed owner CLI functions cover configuration validation, worker/MCP startup, connection authentication, status, ledger inspection/purge, and metadata backup. There is no archive search, correction/reprocessing-from-disk command, durable-job resume, or mandatory standalone backfill command. Historical collection uses explicit discovery/subscription windows or direct identities.

## Persistent schema and implementation boundary

SQLite is the proposed initial metadata store. The two-table DBML describes logical types, indexes, and invariants; conditional rules and timestamp ordering still require SQL constraints and application transaction tests. UTC timestamps and application-generated opaque UUIDs are proposed for IDs.

`subscriptions` owns active collection rules. `delivery_attempts` owns retained identity/outcome/delivery metadata. Agent and connection IDs refer to owner configuration, not database registries; historical subscription IDs deliberately have no foreign key so deletion does not erase receipts. Validate live references in services and never reconstruct authorization from historical rows.

Keep full item identities on attempt rows rather than adding a normalized item table preemptively. Index `(agent_id, item_key, acknowledgement_time)` for suppression, with full-identity checks; use additional indexes for owner-scoped inspection and retention. Do not place a unique constraint on agent/item keys.

The database must contain no payload/blob, cleaned content, discovery-page dump, search text, credential, queue, cursor, or pending-job data. Configuration and the metadata database can be backed up without archiving content. Startup validates configuration/schema and starts empty queues/buffers; it does not scan ledger errors or subscriptions to reconstruct unfinished work.

### Shutdown

Retain the earlier best-effort shutdown policy, adjusted for nonpersistent content: interactive shutdown may drain accepted work or discard it; OS shutdown uses a configurable total grace period, previously agreed default 15 seconds. Stop new roots, allow accepted queued/in-flight discovery/fetch follow-up to drain only within the deadline and memory limits, and never wait indefinitely for agent acknowledgment.

Accept acknowledgments for already delivered items while shutdown can safely commit them. At exit discard all remaining content and runtime work; no acknowledgment is synthesized for undelivered/unacknowledged buffers. Committed ledger/subscription data survives; duplicate network work after restart is accepted. Blocking offloads must not keep the process alive beyond the shutdown budget.

## Acceptance tests

- **Agent isolation:** Reject forged agent, subscription, delivery, and cursor ownership; filter ledger results and connection capabilities by current grants. Never reuse one agent's content buffers for another.
- **Connection validation:** Unknown/mismatched private targets fail closed; connection rename/credential rotation preserves identity, and unrelated account reassignment cannot silently inherit a connection ID.
- **Subscription shape:** Accept supported multi-channel or multi-playlist arguments but reject mixed selector types in one subscription; enforce shared content/options/lookback and owner-only mutation.
- **Runtime edits:** Existing requests keep their admitted snapshots, new requests use edits, and deletion preserves receipts without granting access or implicitly cancelling work.
- **Identity:** Equivalent routes/selectors/options normalize to one key; different languages or private namespaces remain distinct. Raw/cleaned share a key. Hash collisions raise an integrity error.
- **Discovery-only:** Returning references does not fetch content or create receipts. Window boundaries, unknown dates, partial-target failures, and incomplete total counts are explicit.
- **Request deduplication:** Repeated discoveries collapse before fetching; acknowledged identities suppress across subscription/direct origins and equivalent authorized routes for one agent, not across agents.
- **Direct override:** A normal repeat returns `already_served` without fetching; `allow_repeat=true` performs a fresh fetch and creates a new attempt even for identical representation.
- **Unacknowledged retries:** Fetch errors, processing errors, and delivered-but-unacknowledged attempts do not suppress a fresh request. Forced failure never erases a prior successful acknowledgment.
- **Ledger outcomes:** Fetch success plus processing failure remains distinguishable; delivery/ack fields stay null for failed results. No body, raw bytes, credential, or durable queue state enters the DB.
- **Progressive batches:** Enforce connection item caps and response-byte caps; begin delivering before entire discovery finishes, and report continuation/completion accurately.
- **Large items:** Complete all content chunks before enabling acknowledgment. Unsupported oversized/raw-binary delivery fails explicitly rather than truncating.
- **Acknowledgment:** Enforce ownership and complete delivery, commit before reporting success, preserve first acknowledgment time, and handle repeated or lost-response acknowledgment safely.
- **Crash windows:** A crash before outcome commit may leave no row; a lost content response with a delivery timestamp but no acknowledgment remains retryable. Restart never rebuilds jobs.
- **Restart receipts:** Known fully delivered IDs can be acknowledged after restart while retained; old runtime cursors expire; purged IDs fail explicitly.
- **Resource bounds:** Bound queues, scan identity sets, per-agent buffers, prefetched/unacknowledged batches, payload bytes, and offloading. A stalled agent cannot occupy all consumers indefinitely.
- **Concurrency:** Independent discovery and fetch limits work above one; per-connection pacing is respected and database sessions are not shared unsafely.
- **Retention:** Expire receipts by acknowledgment age, not publication age. Purge enables redelivery intentionally, respects live deliveries, and does not use one subscription's lookback as an agent-wide retention rule.
- **Shutdown:** Drain accepted work only within the deadline, never fabricate acknowledgments, release content, preserve committed metadata, and avoid indefinite offload waits.

## Remaining implementation choices

No unresolved product-policy conflict blocks this checkpoint. The important choices before implementation are the first adapter/content combinations, stable account-namespace resolution, exact MCP schemas, canonical JSON test vectors, local IPC and shutdown mechanism, retry/timeouts, payload/batch/queue/memory limits, receipt retention, and raw binary/chunk transport.

Pull-driven collection, owner-file grants, two-table placement, owner-only purge, and concrete interface spellings are explicit implementation recommendations, not separate user-approved numeric defaults. Validate them during implementation without silently restoring archive semantics or treating examples as deployed capabilities.

Begin with a fake adapter, two agents, editable subscriptions, the two-table ledger, acknowledgment/override tests, and bounded shared queues. Then implement one real discovery/fetch path and progressive delivery. Preserve checkpoints 0.1 through 0.9; adoption of 1.0 requires a separate reviewed migration/export decision if older implementations contain data.
