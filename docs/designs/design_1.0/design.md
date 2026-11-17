# ContentHub: Design checkpoint 1.1

Version: 1.1\
Status: Agreed product direction and unified owner-auth command; other interface spellings and operating defaults proposed below\
Predecessor: `designs/design_1.0`, preserved unchanged\
Working name: `contenthub`

ContentHub is a local, connection-authorized content retrieval service with per-agent acknowledged-delivery deduplication. It is not a content archive. Agents discover upstream objects, explicitly fetch content, manage their own subscriptions, and inspect their delivery ledger through MCP. Content exists only in bounded runtime memory; persistent records describe subscriptions and retrieval attempts, never payloads.

The initial target operating system is macOS, running as the owner's normal logged-in user. API keys and OAuth credentials belong in macOS Keychain, not the application database, configuration file, shell command arguments, or agent-visible MCP inputs. Linux/Windows support and unattended system-daemon operation are outside the first release.

This is a standalone checkpoint based on 1.0, not a migration or an implementation. The accompanying `db-structure.dbml` retains two tables: `subscriptions` and `delivery_attempts`. The table name `delivery_attempts` emphasizes that errors and unacknowledged responses are present alongside acknowledged deliveries.

Checkpoint 1.1 adds lean JSON response examples, requires whole-item delivery without per-item chunking, and documents boundaries for an optional future content store and semantic-search index. Those extension notes do not introduce content persistence, embeddings, a vector database, or new tables in this release.

## Decisions and scope

- **Authorization:** Owner-configured agents receive grants to connections. Agents may modify their own subscriptions but cannot edit connections, credentials, grants, or their own identity.
- **Platform and secrets:** Target macOS first, with native Keychain-backed credential storage and owner-operated authentication commands. Browser OAuth is a reusable contract for future OAuth-capable connections, not a prerequisite for the first public YouTube connection.
- **Subscriptions:** One agent owner, one connection, one selector type, one shared content request/representation, and one default lookback. A selector may accept several channels, several playlists, or several timelines when supported; a Gmail query can express broad selection. Do not mix channel and playlist selector types inside one subscription.
- **Direct discovery:** Discover supported targets within an authorized connection and explicit time window. Discovery returns references/metadata, not content-delivery receipts.
- **Direct fetching:** Check the agent's retained acknowledgments. If already acknowledged, return an `already_served` notice unless explicitly overridden; an override performs a new fetch and requires a new acknowledgment.
- **Subscription fetching:** Discover, normalize identities, deduplicate within the request, exclude acknowledged identities, and fetch remaining content. Acknowledgments apply across all of an agent's subscriptions and direct requests.
- **Representation:** Request raw or cleaned output. Either acknowledged representation counts as served for subscription and direct-request checks; different material content requests, such as transcript languages, remain different identities.
- **Persistence:** Store subscriptions and attempt/receipt metadata only. No content archive, raw/processed tables, durable execution queue, or content sharing across agents.
- **Delivery:** Serve complete items one at a time or in bounded batches. Never chunk or truncate an individual item; return `item_too_large` if it cannot fit the applicable whole-item limits. Continue fetching subsequent batches only within bounded prefetch and per-agent memory budgets.
- **Acknowledgment:** A durable ledger entry becomes suppression evidence only when acknowledged. Successful fetching, response preparation, delivery time, and errors alone do not suppress a later request.
- **Errors:** Fetch/processing outcomes are inspectable diagnostics, not persistent retry eligibility. A new request for an unacknowledged item may try again after current authorization, target, capability, and resource checks.
- **Execution:** One service owns shared bounded asyncio discovery and fetch queues, with independently configurable consumer concurrency. Cleaning is part of the fetch task, not a separate process or queue.

The first implementation target is public YouTube discovery and transcript fetching, using the three libraries specified below. Gmail, X, and WhatsApp remain future sources. Supporting an identity or raw byte representation does not promise an adapter for every source, private target, or large-media format.

## Scope exclusions and retained principles

This release does not include durable `items`, `subscription_items`, `raw_payloads`, or `item_contents` archive tables. There are no current-content pointers, saved-input corrections, immutable published-result chains, content-search indexes, or saved-content retrieval snapshots. Any existing archived data must not be deleted automatically as part of adopting this design.

Subscriptions persist editable collection rules, not a YAML-only identity registry. They do not grant access to content. Connection grants define access, and runtime subscription edits apply to subsequent requests without an owner restart.

There is no durable fetch-failure counter or exhausted-allowance policy. Errors do not automatically block future requests, and persisted historical failure counts must not become retry eligibility. Use acknowledgment-based deduplication, not archive-based reuse: eligible fresh requests fetch upstream again, not from disk or another agent's memory.

Retain the six-part canonical identity and SHA-256 collision checking. Retain local MCP/CLI communication, one service owner, disposable execution, explicit windows, structured adapters, and bounded async execution. No fallback extractor is introduced.

## First connection: YouTube

Implement a YouTube connection type with two alternative discovery backends and one transcript-fetch backend per configured instance. The owner may configure several independent instances and grant each to multiple agents; backend choice changes execution, not namespace, item identity, acknowledgment scope, or the subscription's one-selector rule.

The agreed library choices are `yt-dlp` for discovery without configured credentials, `google-api-python-client` for API-key-based discovery, and `youtube_transcript_api` for fetching transcripts. Backend defaults and configuration spellings below are implementation recommendations, not a requirement to introduce additional persistent entities.

### Backend roles and prerequisites

| Role | Library and adapter key | Initial use |
|---|---|---|
| Discovery without credentials | `yt-dlp`, imported as `yt_dlp`; key `yt_dlp` | Embed `YoutubeDL` in Python for metadata discovery, not media downloading; the documented extraction API supports `extract_info(..., download=False)` ([yt-dlp embedding documentation](https://github.com/yt-dlp/yt-dlp/tree/master)). |
| API-key discovery | `google-api-python-client`, imported as `googleapiclient`; key `google_api` | Use YouTube Data API v3 with a secret-resolved API key passed as `developerKey`; the Google client documents this key-based construction for public-data access ([API-key guide](https://googleapis.github.io/google-api-python-client/docs/api-keys.html)). |
| Transcript fetching | Distribution `youtube-transcript-api`, imported as `youtube_transcript_api`; key `youtube_transcript_api` | Use `YouTubeTranscriptApi().fetch(video_id, languages=[language])`; the project documents keyless transcript access, exact-language lists, and a `FetchedTranscript` result ([library documentation](https://github.com/jdepoix/youtube-transcript-api)). |

For `yt_dlp`, the product requirement is public discovery with no configured API key, OAuth flow, browser-cookie import, or login. Do not reinterpret “no credentials” as a guarantee of zero runtime dependencies: yt-dlp documents additional components for full YouTube support and warns that flat-playlist metadata can be incomplete or approximate ([yt-dlp documentation](https://github.com/yt-dlp/yt-dlp/tree/master)). Pin and smoke-test a supported metadata-only configuration; report unmet dependencies or access blocks instead of silently importing user cookies or enabling a paid service.

For Google discovery, the owner must configure a Google Cloud API key with YouTube Data API enabled, and requests are subject to the project's quota ([YouTube Data API setup and quota documentation](https://developers.google.com/youtube/data/getting-started)). An API key is not authorization to read private user data; Google's client documentation distinguishes that from OAuth-based access ([API-key guide](https://googleapis.github.io/google-api-python-client/docs/api-keys.html)). Keep private/account-authenticated YouTube access out of this first connection contract.

The transcript library documents `RequestBlocked`/`IpBlocked` failures, including cloud-hosting limitations, and warns that its undocumented upstream interface can change ([youtube-transcript-api documentation](https://github.com/jdepoix/youtube-transcript-api)). Treat these as explicit runtime failures, not evidence that captions are absent; no proxy purchase, provider substitution, cookie import, or fallback extractor is automatic.

### Selection and owner configuration

Use `yt_dlp` as the proposed default discovery backend. Enable `google_api` only when the owner configures its credential reference; lack of that optional key must not disable keyless discovery or direct transcript fetching. Capability inspection reports each method's availability separately.

Proposed owner configuration shape:

```yaml
connections:
  youtube1:
    id: "2c68e931-7f3a-4da0-8aa6-fda853d8a7e4"
    provider: youtube
    discovery:
      default_method: yt_dlp
      methods:
        yt_dlp: {}
        google_api:
          api_key_ref: contenthub/youtube1/google_api
    fetch:
      method: youtube_transcript_api
```

Here `youtube1` is a unique owner-configured CLI alias, `id` is the stable connection UUID stored in subscriptions/receipts, and `provider: youtube` identifies the adapter type. A second instance such as `youtube2` must have a different stable UUID and ordinarily its own credential reference, for example `contenthub/youtube2/google_api`. Multiple instances may deliberately share a credential reference, but provider or method equality does not imply credential sharing.

This is an illustrative credential reference, not an API key. Resolve `contenthub/youtube1/google_api` through the macOS Keychain credential provider described below. Omit the `google_api` entry for a keyless configuration; never put the key value into subscription JSON, responses, identity options, or the ledger. Missing/invalid configured credentials make that method unavailable for discovery and produce a clear error when selected for execution; they do not prevent the owner from selecting it for authentication setup.

Expose an optional `discovery_method` argument on standalone discovery and subscription-fetch requests. A request may select only owner-enabled methods; omission uses the connection default, snapshotted on admission. Do not automatically run both methods or fail over between them. If a caller explicitly retries with the other backend, retain the same item identity and receipt checks.

A subscription still stores one connection and one selector type; the backend is execution routing, not a selector or material content option. Initially use the connection default or an explicit per-request override rather than adding a subscription column for backend preference. Exact API names remain proposed; this routing recommendation preserves the two-table schema.

### Shared selectors, identities, and date windows

Implement both discovery backends behind the same `youtube_channels` and `youtube_playlists` selector contracts. Accept one or more canonical channel IDs or playlist IDs respectively, not a mixed list; URL/handle convenience resolution can be added only with validation and explicit target normalization. The initial fetch capability is `content_kind=transcript`, not video/audio downloads, comments, or general YouTube search.

Normalize discovered objects to `source=youtube`, `upstream_kind=video`, `namespace=""`, and `source_id=<video_id>`. Channel IDs, playlist IDs, playlist-entry IDs, discovery backend, and API key do not become item identity. Raw/cleaned transcript output continues to share one identity; requested language remains a material option.

Use video publication time for the requested window, not the time a video was added to a playlist. Google's playlist-item resource distinguishes `snippet.publishedAt` (playlist insertion) from `contentDetails.videoPublishedAt` (video publication), and distinguishes playlist-item IDs from video IDs ([playlistItems reference](https://developers.google.com/resources/api-libraries/documentation/youtube/v3/python/latest/youtube_v3.playlistItems.html)). The adapters must normalize that distinction and apply `since <= published_at < until` when sufficiently precise metadata is available.

Do not invent exact timestamps from date-only or approximate values. Enrich metadata through the selected backend where possible; otherwise report unresolved coverage and omit ambiguous candidates from strict-window fetch admission, rather than silently asserting a complete scan. Do not assume playlists are sorted by publication date or stop at the first old entry; scan within configured bounds and report partial coverage when a bound or upstream failure stops the scan.

### Discovery backend implementation

- **`yt_dlp`:** Use the Python library, not shell command construction. Start with flat/lazy metadata extraction; the library documents `extract_flat`, `lazy_playlist`, `skip_download`, `socket_timeout`, and `cachedir=False` for relevant behavior ([YoutubeDL Python parameter definitions](https://github.com/yt-dlp/yt-dlp/blob/master/yt_dlp/YoutubeDL.py)). Disable media/subtitle/thumbnail/info-file writes, download archives, filesystem cache, and automatic browser-cookie loading. Keep lazy enumeration and any per-video metadata enrichment inside bounded discovery execution, not on the event loop.
- **Metadata precision:** Flat-playlist extraction can omit fields, and yt-dlp's `approximate_date` option may make date filters inaccurate ([yt-dlp documentation](https://github.com/yt-dlp/yt-dlp/tree/master)). Do not treat flat IDs alone or approximate dates as proof of exact-window coverage; exercise the enrichment/partial-coverage rule above.
- **`google_api`:** For a channel, obtain `contentDetails.relatedPlaylists.uploads` through `channels.list`, then enumerate that playlist using `playlistItems.list`; this is Google's documented channel-upload discovery flow ([YouTube implementation guide](https://developers.google.com/youtube/data/guides/implementation/videos)). For a playlist selector, enumerate its supplied playlist ID directly and follow `nextPageToken`; that pagination and the video-ID/publication fields are documented in the [playlistItems reference](https://developers.google.com/resources/api-libraries/documentation/youtube/v3/python/latest/youtube_v3.playlistItems.html). Do not use the playlist-entry ID as the video ID.
- **Bounds and errors:** Bound upstream pages, metadata enrichments, total candidate IDs, and timeouts for both methods. Respect connection pacing and Google quota failures; do not silently switch to the other method or create transcript-delivery attempts merely because discovery failed.

### Transcript fetching and representation

Normalize requested language explicitly and pass exactly that language rather than a fallback-language list. The library's documented default prefers a manual transcript over an automatically generated transcript when both exist in the requested language ([transcript selection documentation](https://github.com/jdepoix/youtube-transcript-api)). Use that as the proposed initial track-selection policy, disclose the actual track provenance, and do not translate or substitute a different language automatically. A future caller-selectable track policy must be normalized as a material identity option.

The library returns transcript snippets and provides `to_raw_data()` with `text`, `start`, and `duration`, alongside video/language/generated-track metadata on the fetched result ([transcript result documentation](https://github.com/jdepoix/youtube-transcript-api)). For this connection, define:

- **Raw:** The complete library-exposed snippet list plus relevant transcript metadata, serialized as JSON without application-level text cleaning. “Raw” means this adapter result, not original HTTP/XML bytes; never include authentication or transport diagnostics.
- **Cleaned:** Complete transcript text in the lean response's `content.text`, with deterministic formatting/whitespace cleanup and no summarization or omitted speech. Retain relevant actual language/track provenance internally; expose it where needed without restoring the full `item` wrapper or redundant delivery flags.
- **Delivery:** No per-item chunking. Apply the same serialized-size limits and `item_too_large` behavior to either representation; an error does not count as delivery or acknowledgment.
- **Direct requests:** Accept a validated video ID even if it was not discovered through a subscription. Apply the agent's connection grant and retained acknowledgment check before transcript fetching, with explicit `allow_repeat` behavior unchanged.

Keep exact language required by the content request, with English only as an illustrative normalized default. Raw/cleaned is not part of identity. If explicit timestamp-preserving cleaned output is added, define and normalize that option rather than silently changing the requested output.

### Async integration and first-connection acceptance

Run these library interactions through bounded offloading under the existing discovery/fetch limits; no additional processing queue or standalone service is introduced. Set provider timeouts and retry bounds explicitly, avoid sharing mutable client sessions across concurrent calls unless supported, and test shutdown when an upstream call stalls.

The Google client specifically warns that its `httplib2.Http()` objects are not thread-safe and requires a separate transport per requesting thread ([Google client thread-safety documentation](https://googleapis.github.io/google-api-python-client/docs/thread_safety.html)). Client isolation is an implementation detail; logical connection grants and per-connection quotas remain shared.

Acceptance requires both discovery implementations to pass deterministic adapter fixtures and owner-run live smoke tests, plus transcript fetching in raw and cleaned modes. Include multi-target channels/playlists, overlapping discoveries, video-versus-playlist dates, missing dates, unavailable videos/transcripts, exact-language failure, blocked requests, Google key/quota errors, whole-item size errors, shared-connection agent isolation, acknowledgment suppression, and forced repeats.

Pin tested library versions during implementation and document tested runtime prerequisites rather than relying on unbounded dependency upgrades. These notes verify documentation only: no API credentials have been configured, provider packages installed, or live discovery/fetch requests tested as part of this checkpoint amendment.

## Authorization and owner configuration

### Agent and connection identity

Proposed configuration placement is an owner-managed file containing stable agent IDs, stable connection IDs, connection providers/credential references, agent-to-connection grants, and worker/delivery limits. Labels may change without changing IDs; IDs must not be generated anew at startup or reassigned to unrelated agents/accounts. Secrets remain outside both configuration values and the application database.

MCP starts bound to an owner-selected agent identity. The worker authenticates that local session and derives grants itself; an agent cannot pass a different `agent_id` or an allowed-connection list as authority. Local IPC must restrict access to the appropriate host user. This is application-level isolation, not a defense against an agent with unrestricted access to the host's credentials and files.

Every discovery, fetch, subscription mutation, continuation, acknowledgment, and ledger query checks agent ownership and current connection authorization. A guessed item key, delivery ID, cursor, or subscription ID is never a grant. Reading a receipt does not grant access to fetch its item.

Owner connection/grant configuration initially remains restart-required. Restart invalidates old runtime sessions and cursors; no stale client can retain old permissions. Subscriptions whose owner or connection is absent/ungranted remain stored but unusable, with an explicit configuration/authorization error rather than silent deletion.

### macOS runtime and Keychain storage

Use the user's macOS session for the worker, owner CLI, and agent-bound MCP clients. Choose local IPC and packaging for that target; do not require root, a remote credential service, or a privileged system daemon. A per-user login helper can be considered later, but initial foreground/manual startup must work first. Document and test the supported macOS/Python builds during implementation rather than claiming an untested OS-version floor.

Apple calls the credential facility Keychain, and the Keychain Access application manages stored keys, passwords, and related items ([Apple Keychain Access guide](https://support.apple.com/guide/keychain-access/what-is-keychain-access-kyca1083/mac)). Use a narrow credential-provider interface with a native macOS Keychain implementation; Python's `keyring` library supports that backend and exposes `set_password`, `get_password`, and `delete_password` operations ([keyring documentation](https://keyring.readthedocs.io/en/stable/index.html)).

The proposed implementation uses `keyring.backends.macOS.Keyring` explicitly, or verifies that the selected backend is the native macOS backend. Reject null, plaintext-file, or unexpected fallback backends; a missing/locked/denied/unavailable credential must produce an actionable error rather than silently downgrade secret storage.

Map the opaque configuration credential reference to a stable Keychain entry, for example service `contenthub` and account equal to the complete reference `contenthub/youtube1/google_api`. The account field here is a lookup label, not the upstream email address or item namespace. Although this example reference contains the initial alias, it is an opaque configured value: renaming the alias must not regenerate or delete its referenced credential. Several connections may share a reference only when deliberately configured by the owner.

Store API-key values and future OAuth refresh tokens/token bundles in Keychain. Short-lived access tokens can be held in worker memory and renewed from the protected refresh token; persist any token needed across restarts only through the same credential provider. Never create a plaintext `token.json`, secret `.env`, token-bearing config entry, or custom encryption key stored beside encrypted secrets. Backups of configuration and metadata contain references only and require credentials to be provisioned separately.

Keychain storage is not an additional sandbox against arbitrary code running as the owner: the keyring documentation warns that other scripts using the same Python executable may access its stored credentials without another prompt ([keyring macOS security notes](https://keyring.readthedocs.io/en/stable/index.html)). Keep MCP and local IPC ownership checks, never return secrets to agents, and do not promise protection from an agent with unrestricted host-shell access.

### Unified owner authentication

Use one owner command for credential setup and reauthentication. The owner selects a configured connection and optionally a method, not an authentication mechanism:

```text
contenthub auth <connection_id> [--method <method>]
```

`<connection_id>` accepts either the stable connection UUID or its unique configured alias, such as `youtube1` or `youtube2`; it never means the provider type or a request to authenticate every YouTube connection. Resolve the alias to the stable ID before acting, display the resolved connection/method, and reject unknown or ambiguous identifiers. Alias changes do not rewrite persisted connection IDs, grants, subscriptions, receipts, or credential references; configuration validation must prevent aliases that conflict with another connection's ID.

Select the interaction using the adapter's declared authentication requirement for the owner-enabled method, not the presence/validity of a stored secret:

- **API key:** Ask for hidden terminal input and store it through the configured Keychain reference.
- **OAuth:** Open the system browser and perform the supported provider authorization flow, then store the resulting credentials through the configured Keychain reference.
- **No authentication:** Explain that the selected method requires no credentials; do not prompt for a key, open a browser, or change configuration.
- **Unsupported configuration:** Reject unknown/disabled methods or unsupported authentication flows explicitly. Authentication does not enable new methods, change grants, or silently choose another connection.

When `--method` is omitted, choose the sole configured method requiring authentication, even if the default discovery method is keyless or credentials are missing. If several methods require authentication, ask the owner to choose; in noninteractive use return `method_required` with the permitted choices instead of hanging or guessing. If none require authentication, report that no setup is needed. An explicit `--method` always selects that method and does not fall back.

For the illustrated YouTube configuration, `contenthub auth youtube1` selects `google_api`, while `contenthub auth youtube1 --method yt_dlp` reports that authentication is unnecessary. An entirely keyless configuration does not invent a Google method to authenticate. Independent instances are addressed separately:

```text
contenthub auth youtube1 --method google_api
contenthub auth youtube2 --method google_api
```

This command shape and adaptive interaction are agreed design decisions, not implemented commands. Credential status, remote validation, and local logout remain separate owner operations; their exact CLI spellings are implementation choices, not authentication-type subcommands the user must remember.

### API-key interaction

The unified command detects that `google_api` needs an API key. With the illustrated configuration, the owner experience is:

```text
$ contenthub auth youtube1
Connection: youtube1 (2c68e931-7f3a-4da0-8aa6-fda853d8a7e4)
Method: google_api
Authentication: API key
API key: [hidden input]
```

The intended owner flow is:

1. Create the API key in the provider's console and enable the required API; the YouTube Data API setup requires a Google Cloud project, credentials, and the API enabled ([Google setup guide](https://developers.google.com/youtube/data/getting-started)).
2. Configure only the stable credential reference under the connection's method. Adding/changing that configuration requires the normal manual restart; the secret value itself is not configuration.
3. Run `contenthub auth youtube1`, optionally with `--method google_api`; it opens a hidden terminal prompt for the owner to paste the value. Do not accept the value as a command-line argument, echo it, place it in shell history, or ask an agent to pass it through MCP.
4. Write the value to the configured Keychain entry and report only that it was stored. If writing fails, do not report successful setup.
5. Let the owner credential-check operation make an explicit, bounded provider validation request. Report whether the credential is stored separately from whether the most recent remote check succeeded; a stored key is not proof of validity or sufficient quota.

The worker resolves the reference when executing the configured method and keeps secrets only in necessary in-process client state. A successful owner credential update invalidates the running worker's affected credential/client cache through authenticated owner control, including all connections deliberately sharing that reference; with no running worker, the next startup reads Keychain. Before replacing a shared credential, show its affected connections and obtain owner confirmation. New requests use the replacement after revalidation; already in-flight requests may complete with their original credentials. This does not hot-reload connection definitions or grants.

When the worker cannot access Keychain without user interaction, return `credentials_unavailable` or `authentication_required` to the agent with an owner-facing setup instruction. Do not make an agent request launch an indefinite password/Keychain prompt. Owner setup/check commands may surface macOS access prompts and must handle cancellation without logging the secret.

### Owner OAuth login for future connections

OAuth uses a different setup flow from pasting an API key. For Google desktop apps, application registration, browser consent, PKCE, a loopback redirect, and token exchange are described in the installed-app OAuth guide ([Google desktop OAuth documentation](https://developers.google.com/identity/protocols/oauth2/native-app)). The current public YouTube connection does not need OAuth: its Google discovery route uses an API key, and its other selected routes remain keyless.

For a future configured OAuth-capable connection, the same command chooses the browser interaction:

```text
$ contenthub auth gmail_personal
Connection: gmail_personal
Authentication: OAuth
Opening your browser…
```

`gmail_personal` is an illustrative future connection, not an implemented Gmail adapter. The provider must support and configure an appropriate desktop-app authorization flow; use its supported OAuth library rather than hand-writing token exchange. Unsupported connections return `auth_method_not_supported` instead of attempting a generic login.

- **Application registration:** Before the first login, configure the provider's OAuth client ID, permitted redirect flow, and required scopes. An API key cannot substitute for this registration. Google documents a Desktop app client type and treats desktop clients as public clients that cannot rely on a confidential embedded client secret ([Google desktop OAuth documentation](https://developers.google.com/identity/protocols/oauth2/native-app)). Keep nonsecret client IDs/scopes in owner configuration; if a provider library needs a client-secret value, resolve it through Keychain rather than committing a downloaded credential file.
- **Explicit trigger:** `contenthub auth gmail_personal` displays the target connection/provider and requested scopes, then opens the system browser for the owner to choose an account and consent. Use narrowly scoped read access for retrieval; the agent cannot expand scopes or edit the connection grants.
- **Callback protection:** For providers supporting this desktop flow, use authorization code plus PKCE S256 and random state, with a short-lived listener bound only to loopback on an available port. Validate state, redirect and transaction lifetime, reject unsolicited callbacks, and close the listener on completion/cancellation/timeout. This temporary callback is not a public MCP server.
- **Token and account validation:** Exchange the authorization code through the provider SDK, verify the granted scopes and stable upstream account identity, and require any configured account expectation to match before activating credentials. Store the refresh token/token bundle in Keychain; do not write browser cookies, authorization codes, or tokens to the content ledger or logs.
- **Refresh:** Request supported offline access where needed. Google documents obtaining new access tokens using a refresh token without prompting the user again and treats invalid/expired refresh tokens as a reason to reauthenticate ([Google refresh-token guidance](https://developers.google.com/identity/protocols/oauth2/native-app)). Coordinate refresh per shared connection/credential reference, preserve replacement refresh tokens if returned, and avoid concurrent writes that restore stale tokens.
- **Expired or revoked access:** Report `authentication_required` with the owner command to rerun, rather than automatically opening a browser during an agent request. Normal successful token refresh and same-account login do not require a configuration restart; changing scopes, grants, client settings, or upstream account identity is not credential refresh and follows configuration/account-change rules.
- **Logout:** Define local credential removal/cache invalidation separately from provider-side token revocation. Confirm the affected configured references before deleting a shared credential; deleting a local token alone must not be described as revoking the provider grant. An optional explicit revocation action requires provider support and owner authorization.

Retain the same account/namespace checks on all subsequent discovery and fetch operations. Choosing a different account during reauthentication must not silently rebind an existing private connection ID or expand any agent's access.

### Connections and upstream identity

A connection is an authorized execution route with declared discovery/fetch capabilities, not an item-identity component. It may expose multiple selector types, but each subscription chooses exactly one. A connection's adapter may internally use different provider components; agents do not select arbitrary executables or credentials.

The owner can grant one configured connection to several agents. Configuration, credential references, and connection-level concurrency/rate limits are shared; subscriptions, request state, content buffers, and acknowledgments remain agent-specific. An adapter may reuse or isolate client sessions as needed for safe concurrency without creating separate logical connections for each agent. A matching namespace never grants access to another connection.

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

The YouTube channel/playlist selectors are the initial implementation targets described above; other sources remain illustrative and deferred. None of the examples claims already implemented provider support. Different selector types, connections, content options, representations, or lookbacks require separate subscriptions.

### Editing and lifecycle

Agents list, create, update, and delete only their own subscriptions using currently granted connections and supported options. Validate updates transactionally. A request snapshots its subscription definition on admission; edits affect later requests, not work already accepted. No durable settings revisions or subscription-version table is required.

Deleting a subscription does not delete ledger records, revoke a connection grant, or cancel admitted work implicitly. The ledger keeps the originating subscription ID as historical metadata without a live foreign key. Old subscription IDs are not reused. An explicit request-cancel operation may discard runtime work without fabricating acknowledgments.

### Pull-driven first version

Use pull-driven collection initially: `fetch_new(subscription_id)` starts discovery/fetching. There is no automatic polling, startup scan, or prefetch merely because a subscription exists. This is the simplest proposed operating default for the non-archive service; scheduled collection is deferred.

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
4. Validates required output and whole-item serialized size, records a terminal attempt outcome, and places deliverable content in the requesting agent's bounded buffer.

Raw output means adapter-defined payload representation, not authentication headers, transport credentials, or unrestricted diagnostic dumps. Define each adapter's raw packaging and media type. Binary/multipart delivery requires an explicit transport contract and size bounds; the first implementation may reject unsupported representations rather than silently truncate or write them to disk. Upstream pagination or multipart assembly is allowed internally, but the requested item must be returned in one complete MCP response entry, never across content chunks.

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
                         MCP whole-item batches / next batch
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

## Whole-item delivery, batch continuation, and acknowledgment

### Batches

Support `one_by_one` and `batch` delivery modes (proposed names). One-by-one caps each batch at one item; batch mode caps it at the connection's configured item count, also respecting byte/token limits. For multi-connection requests, keep batches connection-specific so limits remain unambiguous.

Return structured JSON with an opaque request ID, item delivery IDs, complete content, and a next-batch cursor when work/results remain. A delivery ID is the corresponding durable attempt ID, not merely an item key. Keep discovery completion, pending fetch/preparation work, and buffered content availability distinct in request status; do not claim “no more items” before discovery and admitted fetching finish. Ordinary content responses need not repeat every progress field or expose a separate batch ID; a list of delivery IDs is sufficient to acknowledge a batch.

Every successful item entry contains its full requested content. There is no per-item chunking, partial-content result, or content-continuation cursor. If several individually valid items exceed a batch's budget, return fewer whole items and defer the others to another batch. If one item cannot fit on its own within the configured payload, whole-item, or response-size limits, return an explicit `item_too_large` error for that item, without content or truncation. Leave its `delivery_time` and `acknowledgement_time` null, disallow acknowledgment, and continue with other eligible items.

Enforce both bounded input acquisition and serialized output limits, including JSON/envelope or binary-encoding overhead. Start serving ready batches while later pages/items are still being discovered/fetched; no global chronological ordering across unfinished scans is promised.

### Lean JSON response

The following example is a batch from a request for cleaned YouTube transcript text. All entries in `items` contain complete content and require acknowledgment by contract; omit the nested `item` identity wrapper and the redundant `content_complete` and `acknowledgement_required` fields.

```json
{
  "request_id": "req_123",
  "items": [
    {
      "delivery_id": "delivery_456",
      "source_id": "video_789",
      "content": {
        "text": "Welcome to today's discussion."
      }
    }
  ],
  "next_cursor": "cursor_abc"
}
```

For an individual oversized result, return a per-item error rather than a partial item. Successful peers may still appear in the same response's `items` array; this example has none.

```json
{
  "request_id": "req_124",
  "items": [],
  "errors": [
    {
      "source_id": "video_oversized",
      "code": "item_too_large",
      "message": "The complete item exceeds the configured size limit."
    }
  ],
  "next_cursor": null
}
```

- **Request context:** In a homogeneous request, inherit source, content kind/options, representation, and connection from the admitted request rather than echoing the full six-field identity on every item. `source_id` identifies the upstream object, and `delivery_id` identifies the specific attempt to acknowledge.
- **Mixed requests:** A source ID alone may be ambiguous across namespaces, content kinds, or language options. Where needed, add a compact `request_ref` that resolves to the originating request/subscription input and its admitted content context; separate connection-specific batches as already required. Do not change the canonical internal/ledger identity just to simplify the response.
- **Continuation:** `next_cursor` means continue this request for further whole-item batches or progress, not retrieve the rest of an item. It is null only after collection has terminated and no further result entries remain; null alone does not certify successful upstream coverage. An empty batch with a live cursor can mean work is still pending. Request status and structured errors report partial/failed discovery.
- **Acknowledgment:** Every entry in `items` requires explicit acknowledgment; error/skip notices and discovery references do not. An empty `errors` array may be omitted. A failed attempt's diagnostic ID, if exposed for inspection, does not make it acknowledgeable.
- **Content shape:** `content` remains source-specific structured output. Transcript timestamps such as `start_seconds` are content features, not mandatory delivery metadata; return plain transcript text when that is the requested output. If timestamp inclusion changes the requested content contract, normalize that choice in material options. Raw JSON or text stays inside the same outer response with appropriate representation/media-type information where needed.

Exact field spellings are proposed, but whole-item delivery, explicit per-item errors, omission of the redundant wrapper/flags, and unambiguous result-to-request correlation are required. No internal content-result or future storage contract has to mirror this lean public response.

### Acknowledgment semantics

An agent acknowledges completed delivery IDs, individually or as an explicit list for a batch. Validate all IDs against the bound agent, current authorization, successful preparation, and complete delivery; duplicate acknowledgments are idempotent and preserve the original time. A batch acknowledgment covers only its listed fully delivered items, not future work or failed entries.

Update the ledger transactionally before reporting acknowledgment success and then release acknowledged content. If the database write fails, report failure and do not treat the item as acknowledged; the agent can retry. A lost acknowledgment response is safe to retry by delivery ID.

An acknowledgment is an explicit caller declaration, not proof the model read or understood content. Do not mark discovery, fetching, response preparation, an oversized-item error, or mere timeout as acknowledgment.

### Delivery timestamp and restart

`delivery_time` records the server's complete-delivery handoff attempt, not proof of receipt. Persist it for successful item entries immediately before handing off the response containing those complete items so an immediate acknowledgment can be validated. Do not set it for error entries. Database writes and network delivery cannot be one atomic transaction: a failed handoff may leave a delivery timestamp, but without acknowledgment it does not suppress redelivery.

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

Record `item_too_large` at the stage that detects it: exceeding acquisition limits before a complete upstream fetch is a fetch error; failing whole-item output/serialization limits after a successful fetch is a preparation error with `fetching_status=fetched`. Complete size validation before persisting the terminal attempt outcome; do not insert a successful outcome and later mutate its immutable error classification during batching. Either error leaves delivery and acknowledgment unset. Rebatching individually valid items is not an item-size error.

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

## Future content store and semantic-search extension

This section defines extension boundaries, not features to implement in 1.1. The current service continues to store only subscriptions and delivery metadata; no archive tables, stored content versions, embeddings, vector database, indexing worker, or durable indexing queue is introduced.

### Shared internal content-result contract

Fetch/preparation should return a typed, transport-independent content result. It carries the complete canonical item identity/key, requested representation, payload/media type, actual fetch time, and relevant source/provider/parser provenance. Parser provenance applies only when parsing occurred. Keep credential/session material out of this object; it is not a transport-debug dump.

Delivery consumes this result instead of reaching into provider-specific responses. Keep the internal result distinct from the lean MCP response: public field omission must not discard identity or provenance that internal consumers need. Its fields are runtime data in this release, not new ledger columns.

An optional future content store can consume the same result without changing adapters or embedding persistence into the delivery service. Keep a clear function/type boundary; do not build a speculative event bus, plugin framework, or unbounded fan-out pipeline now.

### Separate delivery and storage ownership

Delivery owns temporary agent/request buffers, batch continuation, acknowledgments, and their memory limits. A future store owns retained copies, storage permissions, retention, and deletion. Acknowledgment releases delivery-owned memory; it must not implicitly delete an archived copy or make the delivery service responsible for archive retention.

Future archive retrieval must be an explicit operation with defined receipt behavior. Do not silently change direct fetching or `allow_repeat=true` into a cache lookup: an explicit repeated upstream fetch must still fetch fresh content. Adding a store must not implicitly enable cross-agent sharing.

### Logical items versus stored versions

Retain the existing `item_key` as the identity of the requested object, independent of delivery attempts or storage locations. A future store needs a separate stored-result/version ID because repeated fetching can observe an edited post, changed transcript, different raw/cleaned representation, or changed parser output under the same logical identity.

A future stored version should retain its item identity, representation, fetch/provenance metadata, and content checksum. A checksum describes bytes or a defined canonical representation, not account permission or logical identity; do not silently merge versions merely because item keys match. Distinguish stored-result IDs from attempt/delivery IDs, and do not require a retained ledger row to keep an archive object identifiable.

### Semantic index as a derived capability

Treat semantic search as an optional index over retained searchable content, not as the delivery ledger or the sole source of permission decisions. Preserve a reference from every index entry to a specific stored-result version, so search hits resolve to the content actually indexed.

- **Index metadata:** Track stored-result ID, item key, text-segment identity, and the versions of the text extraction/segmentation and embedding models. Define replacement/reindexing behavior so stale versions do not silently appear current.
- **Index segments versus delivery:** Internal text segmentation for embeddings is unrelated to MCP delivery. It does not introduce per-item response chunking; whole-item delivery and explicit oversized-item errors remain the public contract unless separately revised.
- **Authorization:** Keep sufficient owner/connection and verified upstream provenance to recheck current access. Restrict candidates to the caller's authorized scope and validate access before exposing text, snippets, or source metadata; neither a vector match, namespace match, nor knowledge of a stored ID grants access.
- **Deletion and revocation:** Archive deletion must remove or invalidate derived search entries, and read-time checks must prevent stale index entries from revealing deleted or newly unauthorized content. Rebuild the index from retained content rather than treating embeddings as the only recoverable record.
- **Independent retention:** Purging delivery receipts permits redelivery; it must not accidentally delete archive objects or embeddings. Conversely, deleting archived content must not silently rewrite acknowledgment history. Any combined purge needs explicit scope.

### Explicit durable handoff when storage is added

Do not enqueue a memory-only indexing task, free the only payload after acknowledgment, and assume indexing will eventually succeed. A future persistence/indexing feature must define the durable handoff: for example, confirm a stored copy before releasing storage-owned input, then index from that copy with bounded retry/recovery.

That future feature must explicitly choose whether delivery waits for storage, how storage failure is reported, and how indexing is resumed or rebuilt. Recovery machinery belongs to the optional storage/indexing capability, not implicitly to all retrieval requests. Without that feature enabled, 1.1 keeps the existing disposable execution and memory-only content behavior.

## Proposed interfaces

The MCP names and argument shapes below are implementation proposals. The behavioral separation, authorization, identity, and acknowledgment rules above are normative; the unified owner-auth command is separately agreed.

```text
list_connections()
get_connection_capabilities(connection_id)

discover(connection_id, selector_type, selector, since, until, cursor?,
         discovery_method?)
fetch_items(connection_id, items, representation, delivery_mode, allow_repeat=false)
fetch_new(subscription_ids, window_override?, delivery_mode, allow_repeat=false,
          discovery_method?)
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

`discovery_method` is an optional execution-routing choice among owner-enabled connection backends, not an item option. For a multi-connection subscription request, apply an override only when every involved connection advertises it, otherwise reject before starting work; the first YouTube connection supports the two method keys specified above subject to key availability. Continuation inherits the admitted method and cannot switch it mid-request.

The agreed owner authentication entry point is `contenthub auth <connection_id> [--method <method>]`, which resolves the configured connection and chooses hidden API-key entry, browser OAuth, or a no-authentication explanation. Other proposed owner CLI functions cover configuration validation, worker/MCP startup, credential status/check/logout, ledger inspection/purge, and metadata backup. Interactive credential entry and OAuth login are owner-only, not agent MCP tools; agents receive actionable authentication errors without secret values. There is no archive search, correction/reprocessing-from-disk command, durable-job resume, or mandatory standalone backfill command. Historical collection uses explicit discovery/subscription windows or direct identities.

## Persistent schema and implementation boundary

SQLite is the proposed initial metadata store. The two-table DBML describes logical types, indexes, and invariants; conditional rules and timestamp ordering still require SQL constraints and application transaction tests. UTC timestamps and application-generated opaque UUIDs are proposed for IDs.

`subscriptions` owns active collection rules. `delivery_attempts` owns retained identity/outcome/delivery metadata. Agent and connection IDs refer to owner configuration, not database registries; historical subscription IDs deliberately have no foreign key so deletion does not erase receipts. Validate live references in services and never reconstruct authorization from historical rows.

Keep full item identities on attempt rows rather than adding a normalized item table preemptively. Index `(agent_id, item_key, acknowledgement_time)` for suppression, with full-identity checks; use additional indexes for owner-scoped inspection and retention. Do not place a unique constraint on agent/item keys.

The database must contain no payload/blob, cleaned content, discovery-page dump, search text, credential, queue, cursor, or pending-job data. Configuration and the metadata database can be backed up without archiving content. Startup validates configuration/schema and starts empty queues/buffers; it does not scan ledger errors or subscriptions to reconstruct unfinished work.

### Shutdown

Use best-effort shutdown for nonpersistent content: interactive shutdown may drain accepted work or discard it; OS shutdown uses a configurable total grace period, default 15 seconds. Stop new roots, allow accepted queued/in-flight discovery/fetch follow-up to drain only within the deadline and memory limits, and never wait indefinitely for agent acknowledgment.

Accept acknowledgments for already delivered items while shutdown can safely commit them. At exit discard all remaining content and runtime work; no acknowledgment is synthesized for undelivered/unacknowledged buffers. Committed ledger/subscription data survives; duplicate network work after restart is accepted. Blocking offloads must not keep the process alive beyond the shutdown budget.

## Acceptance tests

- **macOS credentials:** Run integration tests on the supported macOS/Python environment with the native Keychain backend. Test set/read/update/delete, missing entries, denied/locked/unavailable storage, and rejected insecure backends; use fake credentials and clean up test entries. No secret appears in configuration, arguments, logs, metadata backups, DB rows, or MCP responses.
- **Unified owner auth:** Resolve a stable UUID or unique alias to exactly one configured connection. Test independent `youtube1`/`youtube2` credentials, deliberate shared-reference confirmation/cache invalidation, automatic selection of the sole authentication-requiring method despite a keyless discovery default, explicit keyless selection, entirely keyless configuration, ambiguous-method prompting/noninteractive errors, and unknown/disabled-method rejection. Missing credentials remain eligible for setup; authentication never changes grants or enables methods.
- **API-key owner flow:** Hidden input writes to the configured Keychain reference; storage success is distinct from remote validation. Rotation invalidates affected clients without hot-reloading grants; a missing Google key leaves the keyless YouTube methods usable.
- **Future OAuth contract:** When the first OAuth adapter is implemented, test explicit owner browser login, PKCE/state/callback validation, scope/account mismatch, cancellation, token refresh/rotation, shared-credential concurrency, and local logout versus remote revocation. These are not claims of implemented OAuth support for the current YouTube adapter.
- **Agent isolation:** Reject forged agent, subscription, delivery, and cursor ownership; filter ledger results and connection capabilities by current grants. Never reuse one agent's content buffers for another.
- **Shared connections:** Several granted agents may use one connection definition and credential reference while sharing its concurrency/rate budget. Subscriptions, buffers, and acknowledgment history stay agent-specific; a namespace match never grants another route.
- **Connection validation:** Unknown/mismatched private targets fail closed; connection rename/credential rotation preserves identity, and unrelated account reassignment cannot silently inherit a connection ID.
- **First YouTube connection:** Implement `yt_dlp` and `google_api` discovery with `youtube_transcript_api` fetching. Both discovery routes normalize the same public video/transcript identity; Google key absence does not break the keyless route, and neither route silently falls back to the other. Verify exact-language, date-window, raw/cleaned, blocked/quota-error, and whole-item delivery behavior.
- **Subscription shape:** Accept supported multi-channel or multi-playlist arguments but reject mixed selector types in one subscription; enforce shared content/options/lookback and owner-only mutation.
- **Runtime edits:** Existing requests keep their admitted snapshots, new requests use edits, and deletion preserves receipts without granting access or implicitly cancelling work.
- **Identity:** Equivalent routes/selectors/options normalize to one key; different languages or private namespaces remain distinct. Raw/cleaned share a key. Hash collisions raise an integrity error.
- **Discovery-only:** Returning references does not fetch content or create receipts. Window boundaries, unknown dates, partial-target failures, and incomplete total counts are explicit.
- **Request deduplication:** Repeated discoveries collapse before fetching; acknowledged identities suppress across subscription/direct origins and equivalent authorized routes for one agent, not across agents.
- **Direct override:** A normal repeat returns `already_served` without fetching; `allow_repeat=true` performs a fresh fetch and creates a new attempt even for identical representation.
- **Unacknowledged retries:** Fetch errors, processing errors, and delivered-but-unacknowledged attempts do not suppress a fresh request. Forced failure never erases a prior successful acknowledgment.
- **Ledger outcomes:** Fetch success plus processing failure remains distinguishable; delivery/ack fields stay null for failed results. No body, raw bytes, credential, or durable queue state enters the DB.
- **Progressive batches:** Enforce connection item caps and response-byte caps; begin delivering before entire discovery finishes, and report continuation/completion accurately. Split collections into batches of whole items, never split an individual item's content.
- **Lean responses:** Successful entries omit `item`, `content_complete`, and `acknowledgement_required`; preserve unambiguous source/request correlation and delivery IDs. A live cursor may accompany an empty pending batch; null does not turn partial/failed discovery into successful coverage.
- **Large items:** Return `item_too_large` for a single item exceeding acquisition or complete-output limits, accounting for serialization overhead. Do not truncate, deliver fragments, set delivery/acknowledgment time, or block successful peer items. If only the aggregate batch is too large, defer complete items instead of marking them oversized.
- **Size outcome classification:** Detect size failure before terminal outcome insertion. Distinguish fetch-stage size errors from post-fetch preparation errors and preserve retry eligibility without requiring a schema change.
- **Acknowledgment:** Enforce ownership and complete delivery, commit before reporting success, preserve first acknowledgment time, and handle repeated or lost-response acknowledgment safely.
- **Crash windows:** A crash before outcome commit may leave no row; a lost content response with a delivery timestamp but no acknowledgment remains retryable. Restart never rebuilds jobs.
- **Restart receipts:** Known fully delivered IDs can be acknowledged after restart while retained; old runtime cursors expire; purged IDs fail explicitly.
- **Resource bounds:** Bound queues, scan identity sets, per-agent buffers, prefetched/unacknowledged batches, payload bytes, and offloading. A stalled agent cannot occupy all consumers indefinitely.
- **Concurrency:** Independent discovery and fetch limits work above one; per-connection pacing is respected and database sessions are not shared unsafely.
- **Retention:** Expire receipts by acknowledgment age, not publication age. Purge enables redelivery intentionally, respects live deliveries, and does not use one subscription's lookback as an agent-wide retention rule.
- **Shutdown:** Drain accepted work only within the deadline, never fabricate acknowledgments, release content, preserve committed metadata, and avoid indefinite offload waits.
- **Extension boundary:** Fetch/preparation returns an internal typed content result consumed by delivery without provider-specific lookups. Verify identity/provenance survives public-response projection while no archive, embedding, vector index, or background persistence is enabled. Future search text segmentation is not a license for per-item MCP chunking.

## Remaining implementation choices

The initial source/library scope and platform are now specified: macOS with Keychain credential storage, public YouTube channel/playlist discovery through `yt_dlp` or `google_api`, and transcripts fetched through `youtube_transcript_api`. Remaining implementation choices include tested macOS/Python/dependency versions, exact MCP/config schemas, canonical JSON test vectors, macOS-local IPC and shutdown mechanism, retry/timeouts, payload/batch/queue/memory limits, and receipt retention. Concrete OAuth-provider integration, private-source namespace resolution, and binary media delivery belong to later adapters; per-item chunking remains excluded.

Pull-driven collection, owner-file grants, two-table placement, owner-only purge, and other proposed interface spellings remain explicit implementation recommendations, not separate user-approved numeric defaults. The unified owner-auth command and its connection/method selection behavior are agreed. Validate implementation choices without silently restoring archive semantics or treating examples as deployed capabilities.

Begin with a fake adapter, two agents, editable subscriptions, the two-table ledger, acknowledgment/override tests, and bounded shared queues. Then implement both YouTube discovery backends, the common transcript fetcher, and progressive whole-item delivery. Preserve checkpoint 1.0 unchanged. The two persistent tables and their columns/indexes are unchanged by 1.1; adopting this design must not silently delete existing data, and any necessary migration/export requires a separate reviewed decision.
