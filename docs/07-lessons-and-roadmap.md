[Back to README](../README.md)

# 07. Lessons Learned and Roadmap

## Problems hit and how they were solved

These come from building the lab. Each one cost real time, so they are worth keeping.

| Problem | Cause | Fix |
| :--- | :--- | :--- |
| TheHive install failed with DNS errors on the old repo | StrangeBee retired `archives.strangebee.com` and the old apt repo | Install the `.deb` directly from `thehive.download.strangebee.com` ([03](03-thehive-server.md)) |
| Server froze or services were OOM-killed when TheHive started | Elasticsearch defaults to a 4 GB heap on an 8 GB host that also runs Cassandra and TheHive | Cap Elasticsearch at 1 GB and Cassandra at 512 MB ([03, Step 5.5](03-thehive-server.md)) |
| TheHive could not connect to the database | Cassandra's default cluster name `Test Cluster` does not match TheHive's expected `thp` | Rename the cluster and clear `/var/lib/cassandra/data/*` before first TheHive start |
| Cassandra repo not found | Guides still referenced the `40x` series | Use the `41x` repository for Cassandra 4.1 |
| VirusTotal node failed on the extracted hash | Passing the whole regex result object instead of the `group` list | Select only the `list` value ([06, Phase 4](06-shuffle-soar-workflow.md)) |

<!--
ADD YOUR OWN: two or three more problems you personally hit, with the cause and the fix.
Examples worth including if they happened to you: agent would not enroll (firewall 1514/1515),
Defender deleting Mimikatz, Shuffle could not reach TheHive, webhook returned no data.
-->

## What this project demonstrates

- Building and sizing a multi-service SIEM and case-management stack in the cloud.
- Telemetry engineering: Sysmon with a community config, forwarded through the Wazuh agent.
- Detection engineering: turning a raw event into a precise, MITRE-mapped rule.
- SOAR: webhook-driven enrichment and ticketing with VirusTotal and TheHive.

## Known limitations

- **Single detection.** Only Mimikatz execution is covered. A real SOC needs breadth.
- **Signature-based.** The rule matches `originalFileName`. A recompiled or modified binary, or in-memory
  execution, will not match.
- **Telemetry-dependent.** If Sysmon is stopped or tampered with, the rule is blind (test 4 in
  [05](05-custom-detection-rules.md)).
- **Lab-grade exposure.** TheHive on plain HTTP and a widened firewall are acceptable for a short lab, not for
  production.

## Roadmap

- [ ] Detect LSASS access (Sysmon Event ID 10) so renamed or modified credential dumpers are still caught.
- [ ] Add detections for other techniques (PowerShell encoded commands, new local admin, service creation).
- [ ] Automated response: Wazuh active response to isolate the host after analyst approval in Shuffle.
- [ ] Enrich with more than a file hash (IP and domain reputation).
- [ ] Put TheHive behind TLS and tighten access ([06, Phase 3](06-shuffle-soar-workflow.md)).
- [ ] Export the Shuffle workflow JSON into this repo.
