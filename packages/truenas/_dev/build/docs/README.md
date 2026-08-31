{{- generatedHeader }}

# TrueNAS Integration for Elastic

## Overview

The TrueNAS integration for Elastic enables collection of audit records and system alerts from [TrueNAS](https://www.truenas.com/), the OpenZFS-based storage operating system from iXsystems (formerly TrueNAS SCALE, renamed TrueNAS Community Edition in 25.04).

### Compatibility

This integration is compatible with TrueNAS SCALE 24.04 through 25.10 and collects data through the TrueNAS REST API v2.0. It has been validated against TrueNAS-SCALE-24.10.2.4.

The REST API was deprecated in TrueNAS 25.04 and is **removed in TrueNAS 26**, which replaces it with a JSON-RPC 2.0 API served only over WebSocket. This integration does not collect data from TrueNAS 26 or later.

### How it works

The integration polls the TrueNAS REST API v2.0 over HTTPS using the Common Expression Language (CEL) input. Each data stream authenticates with a TrueNAS API key sent as a bearer token and requests records on a configurable interval.

The `audit` data stream collects incrementally: it records the timestamp of the most recent audit record it has seen and requests only newer records on the following poll, paging through results with the offset and limit query options. The `alert` data stream retrieves the full set of currently active alerts on each poll, since TrueNAS exposes alerts as a live snapshot rather than an append-only log, and indexes only those alerts whose most recent occurrence is newer than the last one already collected.

#### Alert collection limitations

Because TrueNAS exposes alerts as a snapshot with no event history, the `alert` data stream can only observe alerts appearing or recurring. Specifically:

* **Clearing an alert produces no document.** When the condition behind an alert resolves and TrueNAS removes it from the list, nothing is indexed. Documents record that an alert was raised, not that it is still active. Do not treat the newest document for an alert class as proof of a current condition; correlate with the TrueNAS UI or a follow-up alert instead.
* **Dismissing an alert produces no document.** `truenas.alert.dismissed` reflects the state at the moment the alert was last collected and is not updated afterwards.
* **Back-dated alerts are skipped.** An alert that appears with a `last_occurrence` older than the most recent one already collected is not indexed.

An alert that recurs *is* re-indexed, because TrueNAS advances its `last_occurrence`. Use `event.id` (the alert UUID) together with `@timestamp` to distinguish occurrences of the same alert.

## What data does this integration collect?

The TrueNAS integration collects log messages of the following types:

* **Audit records** from the TrueNAS audit framework, covering middleware API calls and authentications, SMB file operations and session activity, and sudo command execution. TrueNAS 25.10 and later add system login events.
* **Alerts** raised by TrueNAS for storage, hardware, and system health conditions, such as pool degradation, scrub results, and disk temperature warnings.

### Supported use cases

Audit records give visibility into who accessed the storage system and what they did with it: administrative API calls, SMB share access and file operations, and privilege escalation through sudo. This supports file access monitoring, administrative change tracking, and investigation of suspicious access to stored data.

Alerts surface storage and hardware health conditions that threaten data availability, allowing pool degradation, failed scrubs, and disk temperature problems to be tracked alongside the rest of the infrastructure estate.

## What do I need to use this integration?

* A TrueNAS system running a supported version, reachable over HTTPS from the host running Elastic Agent.
* A TrueNAS API key linked to an account with sufficient privileges to read audit records and alerts. See the setup steps below.
* Elastic Agent installed on a host that can reach the TrueNAS web interface port (443 by default).

## How do I deploy this integration?

### Agent-based deployment

Elastic Agent must be installed. For more details, check the Elastic Agent [installation instructions](https://www.elastic.co/guide/en/fleet/current/elastic-agent-installation.html). You can install only one Elastic Agent per host.

Elastic Agent polls the TrueNAS API and ships the data to Elastic, where the events are processed by the integration's ingest pipelines. The agent does not need to run on the TrueNAS system itself; a separate host can collect from many TrueNAS systems, using one integration policy per system.

On TrueNAS Enterprise high-availability pairs, point the integration at the virtual IP rather than an individual controller address so that collection follows failover.

### Set up steps in TrueNAS

1. Create a user for monitoring, or choose an existing one, under **Credentials > Users**.
2. Grant that user the built-in **Read-Only Administrator** privilege under **Credentials > Privileges**. This covers every read the integration performs, including `SYSTEM_AUDIT_READ` for audit records. Full administrator access is not required.
3. Create an API key:
   * On TrueNAS 25.04 and later, open the user menu in the top toolbar and select **My API Keys**, then **Add**. Set a name, select the monitoring user, and choose an expiry.
   * On TrueNAS 24.x, open **Settings** (gear icon) **> API Keys**, then **Add**.
4. Copy the generated key. It is displayed only once. The key has the form `<id>-<secret>`, for example `3-AbCd...`, where the leading number identifies the key. **Copy the entire string including that leading number and hyphen.** A key missing the prefix, or with the secret truncated, is rejected as invalid.
5. Confirm the TrueNAS URL you will use begins with `https://`. TrueNAS automatically revokes any API key that is sent over plain HTTP, so a misconfigured collector destroys the credential rather than simply failing.

#### Vendor resources

- [Managing API keys](https://www.truenas.com/docs/scale/toptoolbar/managingapikeys/)
- [Administrative roles and privileges](https://www.truenas.com/docs/scale/scaletutorials/credentials/adminroles/)
- [Audit logging](https://www.truenas.com/docs/scale/25.04/scaletutorials/systemsettings/auditingscale/)

### Set up steps in Kibana

1. In Kibana, navigate to **Management > Integrations** and search for **TrueNAS**.
2. Select **Add TrueNAS** and configure the connection:
   * **TrueNAS URL**: the base URL of the system including scheme and port, for example `https://truenas.example.com`. Must use HTTPS.
   * **API Key**: the full `<id>-<secret>` key created above.
   * **SSL Configuration**: TrueNAS ships a self-signed certificate by default. Either supply the certificate authority here, or set `verification_mode: none` to skip verification.
3. Enable the data streams you want and adjust their intervals. The `audit` data stream also accepts an initial interval controlling how far back the first poll reaches, and the set of audit services to query.
4. Save and assign the policy to an agent.

### Validation

1. Generate activity on the TrueNAS system: log in to the web interface to produce middleware audit records, or access an SMB share to produce SMB records. Audit records only appear for services with auditing enabled.
2. In Kibana, open **Discover** and query `data_stream.dataset: "truenas.audit"` or `data_stream.dataset: "truenas.alert"`.
3. Confirm that documents are arriving and that `event.action`, `user.name`, and `source.ip` are populated on audit records.

## Troubleshooting

- HTTP 401 "Invalid API key": TrueNAS returns this same message for a malformed key, a revoked key, and a key belonging to another system. Check the key format first. It must be `<id>-<secret>` with a 64-character secret, so a complete key is 66 characters for a single-digit id. A key copied without the leading `<id>-` prefix, truncated, or accidentally pasted twice will always fail here.
- API key stopped working unexpectedly: TrueNAS revokes any key that is sent over an insecure HTTP connection. If the URL was ever configured without TLS, the key is permanently revoked and a new one must be generated. Upgrading to TrueNAS 25.04 also revokes older keys that used method allowlists.
- `x509: certificate signed by unknown authority`: TrueNAS uses a self-signed certificate unless one has been installed, and the SSL Configuration field is empty by default, so this is the expected first failure on a stock system. Set the **SSL Configuration** field to `verification_mode: none`, or supply the TrueNAS certificate authority with `certificate_authorities: ["/path/to/ca.crt"]`. The setting is shared by both data streams. Note the error surfaces as a CEL evaluation failure mentioning `do_request()` rather than as an authentication error.
- No audit records collected: auditing must be enabled for the relevant services on the TrueNAS side, and the account linked to the API key needs the `SYSTEM_AUDIT_READ` role, which the Read-Only Administrator privilege includes. An account without it receives an authorization error rather than an empty result.
- HTTP 404 on all requests: the REST API has been removed in TrueNAS 26 and later. This integration cannot collect from those versions.

## Performance and scaling

The `audit` data stream pages through results using offset and limit, requesting up to the configured batch size per request and stopping when a page returns fewer records than the batch size. On systems with heavy SMB traffic, audit volume can be substantial because file operations such as reads and writes each generate a record; reduce the set of audited services or lengthen the polling interval if volume is a concern.

The `alert` data stream fetches the complete set of active alerts on every poll, but indexes only those whose most recent occurrence is newer than the last one already collected. Steady-state volume is therefore proportional to the rate at which alerts appear or recur, not to the polling frequency. Because the endpoint has no time filter, the interval sets how quickly a new alert is detected rather than how much data is collected; the 15 minute default suits most systems.

For more information on architectures that can be used for scaling this integration, check the [Ingest Architectures](https://www.elastic.co/docs/manage-data/ingest/ingest-reference-architectures) documentation.

## Reference

### Inputs used
{{ inputDocs }}

### API usage

These APIs are used with this integration:

* `POST /api/v2.0/audit/query` — retrieves audit records. Accepts a list of services, query filters, and query options controlling ordering, limit, and offset.
* `GET /api/v2.0/alert/list` — retrieves all currently active alerts.

### Vendor documentation links

- [TrueNAS Documentation Hub](https://www.truenas.com/docs/)
- [TrueNAS API reference](https://api.truenas.com/)
- [TrueNAS 26 deprecations, including REST API removal](https://www.truenas.com/docs/scale/26/gettingstarted/deprecations/)

### Data streams

#### audit

The `audit` data stream provides audit records from the TrueNAS audit framework. Each record identifies the service that produced it (`MIDDLEWARE`, `SMB`, `SUDO`, or `SYSTEM`), the event type within that service, the acting user, the client address, and whether the action succeeded. Event-specific details are carried in a per-event structure that varies by service.

##### audit fields

{{ fields "audit" }}

##### audit sample event

{{ event "audit" }}

#### alert

The `alert` data stream provides the alerts currently active on the TrueNAS system. Each alert carries a class identifying the condition, a severity level, the time of first and most recent occurrence, and rendered alert text.

##### alert fields

{{ fields "alert" }}

##### alert sample event

{{ event "alert" }}

{{ transform }}
