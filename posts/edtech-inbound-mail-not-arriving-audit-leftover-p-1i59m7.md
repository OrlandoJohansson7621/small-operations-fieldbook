# Edtech Inbound Mail Not Arriving — Audit Leftover Provider MX Priorities

Short answer: treat inbound-mail readiness as an evidence chain, not as a green check beside an MX lookup. For an edtech tenant, accept ownership first, then prove that the public DNS view names the intended receivers, and finally record a real SMTP delivery outcome. If an unexpected receiver remains anywhere in that chain, stop onboarding and reconcile the zone.

The order matters. A low MX preference number is tried before a higher one, but multiple records are not a migration history. They are a live routing set. An old provider record can still receive traffic when it remains published, so a successful ownership token does not prove that password resets, guardian invitations, or enrollment notices will reach the new mailbox system.

| Evidence available | What it proves | Pick it when | What it cannot prove |
| --- | --- | --- | --- |
| Ownership TXT challenge | The operator can publish a value under the domain | Onboarding begins before mail moves | Which server receives mail |
| Authoritative MX snapshot | The delegated zone publishes the expected routing set | A cutover is planned or completed | That SMTP accepts a recipient |
| Recursive DNS observations | Resolver-facing answers and TTL behavior | Results differ across networks | End-to-end delivery |
| SMTP probe plus correlation | A receiver accepted a controlled message | Readiness needs delivery evidence | Delivery to every recipient |

This is the field guide: choose the weakest test that answers the current question, then require stronger evidence as the tenant approaches activation. Fast checks help. False confidence hurts.

## Why is inbound mail not arriving after provider MX priorities change?

Use the TXT challenge to gate domain control. Use the normalized MX set to gate routing configuration. Use a correlated SMTP result to gate mail readiness. Those are three different claims, so one boolean should not stand in for all of them.

For an early edtech setup screen, the ownership challenge is the right pick. It lets an administrator prove control without moving production mail. Keep its state separate: `ownership_verified` can become true while `inbound_ready` remains false. That distinction makes the UI honest and gives support engineers a useful timeline.

At cutover, pick authoritative DNS evidence. Query the nameservers responsible for the zone, collect every MX record, normalize hostnames, and compare the resulting set with the declared target. Do not inspect only the first answer. A leftover record with preference 20 is still published routing information, even when the intended record has preference 10.

Pause there.

Consider a school that verifies control on Monday and schedules its mailbox cutover for Friday. The zone initially sends mail to a provider at preference 10. During the change, the administrator adds the new exchange at preference 10 and moves the former exchange to 20, intending that lower-ranked record to be a temporary fallback. The verifier must report both records because both are part of the live answer; showing only the first would erase the exact evidence needed to explain split delivery. If the declared target contains only the new exchange, the record at 20 is unexpected, not healthy redundancy. If the declared target intentionally contains both, it is a match, but the team still needs to prove that each published path has the expected behavior. This is the trade-off: strict set comparison can delay activation after an intentional change, yet accepting an undeclared fallback can send student-facing mail down a path nobody is observing. For onboarding, I would choose the delay and show the administrator the complete difference. That is a policy judgment, made visible rather than buried in lookup code.

Near activation, pick end-to-end evidence. Send a uniquely identified test message to a controlled recipient, correlate that identifier with the receiving event, and store the outcome beside the DNS snapshot that preceded it. Now an operator can answer a practical question: what did DNS say when this test was attempted, and which receiver accepted it?

## Read the MX set as a routing graph

Here is the diagram in words. Start at the tenant's domain. Follow its delegation to the authoritative nameservers. Read the complete MX answer. For each exchange, resolve its address records, then observe the SMTP connection and message event. Each arrow produces evidence; each missing arrow narrows the investigation.

MX preference is ordering, not a health score. Lower values are preferred. Equal values represent equally preferred exchangers, and senders can choose among them. If delivery to a preferred host fails, a sender may try another published exchanger. This is why a stale high-numbered MX record is not inert documentation. It remains a possible branch in the graph.

A crisp before/after makes the error visible. Before reconciliation, imagine the answer contains `10 mx1.school-mail.example` and `20 mx.legacy-mail.example`. After reconciliation, it contains only the exchangers the tenant deliberately declared. The important change is not that one number won. The unintended branch disappeared.

There is another edge worth checking: an MX exchange ultimately needs usable address resolution. A tidy MX answer does not help if its target cannot be reached. Capture resolution and connection outcomes separately so an address failure is not reported as an MX mismatch.

## Build a verifier that returns evidence

The implementation below focuses on reconciliation. It accepts observed records from a DNS adapter, compares them with an expected set, and emits a result that can be logged or attached to an onboarding decision. The adapter boundary is deliberate: production code can query authoritative servers and recursive resolvers separately without coupling policy to a particular DNS library.

```ts
type MxRecord = {
  exchange: string;
  priority: number;
};

type MxEvidence = {
  checkedAt: string;
  expected: MxRecord[];
  observed: MxRecord[];
  missing: MxRecord[];
  unexpected: MxRecord[];
  ready: boolean;
};

const normalizeHost = (host: string): string =>
  host.trim().toLowerCase().replace(/\.$/, "");

const normalizeMx = (record: MxRecord): MxRecord => ({
  exchange: normalizeHost(record.exchange),
  priority: record.priority,
});

const key = (record: MxRecord): string =>
  `${record.priority} ${record.exchange}`;

export function reconcileMx(
  expectedInput: MxRecord[],
  observedInput: MxRecord[],
  checkedAt = new Date().toISOString(),
): MxEvidence {
  const expected = expectedInput.map(normalizeMx);
  const observed = observedInput.map(normalizeMx);
  const expectedKeys = new Set(expected.map(key));
  const observedKeys = new Set(observed.map(key));

  const missing = expected.filter((record) => !observedKeys.has(key(record)));
  const unexpected = observed.filter((record) => !expectedKeys.has(key(record)));

  return {
    checkedAt,
    expected,
    observed,
    missing,
    unexpected,
    ready: missing.length === 0 && unexpected.length === 0,
  };
}
```

The comparison includes both exchange and preference. That is strict on purpose. A priority change alters sender behavior and deserves review, even if the hostnames are identical. If policy intentionally ignores priority, encode that choice explicitly and retain the raw observation; silently dropping it makes later debugging harder.

Test the awkward cases. Trailing dots and hostname case should normalize cleanly. An empty answer, one missing intended record, one unexpected legacy record, and equal-priority exchangers all deserve fixtures. Keep timestamps from the system clock rather than inventing them in tests.

The verifier should not mutate DNS. It reports a difference. An administrator can inspect the authoritative zone, confirm the intended migration state, and make the change through the system that owns that zone. This separation preserves an audit trail and avoids turning a diagnostic worker into a privileged control plane.

## Make the failure observable

A useful log event contains the tenant identifier, domain, query vantage point, authoritative nameserver, complete normalized MX set, expected set, check time, and reconciliation outcome. Avoid logging message bodies or student addresses. A generated probe identifier is enough to join the DNS check, SMTP attempt, and receiving event.

Metrics should describe states and transitions rather than individual domains as labels. Count outcomes such as `match`, `missing`, `unexpected`, `dns_error`, and `smtp_rejected`. Measure the age of the latest successful check and the time from ownership verification to delivery proof. Domain names in metric labels create unbounded cardinality; keep that detail in logs or traces.

Alert on sustained conditions. A single recursive lookup can reflect cached data during a planned DNS change, while repeated authoritative mismatches indicate that the published zone still differs from intent. Page only when a user-facing promise is threatened. A tenant that has not scheduled activation belongs in a workflow queue; an activated tenant whose controlled probes stop arriving needs urgent attention.

Use this debugging sequence:

1. Confirm that the ownership token and MX check refer to the same normalized domain.
2. Compare authoritative and recursive MX snapshots, including every preference value.
3. Resolve each intended exchange and record connection outcomes.
4. Correlate the controlled message identifier with the receiving event.
5. Classify the failure at the earliest broken edge, then repeat the evidence chain after remediation.

Notice what is absent: guessing from a dashboard badge. The evidence says where to look.

## Deployment and operational trade-offs

Run verification asynchronously. DNS and SMTP involve remote systems, variable latency, retries, and caching; holding an onboarding request open for the entire chain produces a brittle user interaction. Return the current state, enqueue a check, and stream or poll for the resulting evidence. Make jobs idempotent by keying them to the domain, intended configuration revision, and probe identifier.

Retries need boundaries. Retry timeouts and temporary lookup failures with backoff, but do not translate a deterministic mismatch into network noise. Preserve the last good evidence while showing that it is aging. That gives operators continuity without pretending an old success describes the current zone.

There is a real cost trade-off. Authoritative queries, recursive observations from several vantage points, and SMTP probes provide progressively stronger evidence, but they also consume more time and infrastructure. Use ownership proof early, perform DNS reconciliation when configuration changes, and reserve controlled delivery probes for activation and ongoing health checks. The schedule should follow risk: a domain carrying authentication and enrollment mail needs stronger evidence than a dormant sandbox.

DMARC is related but answers a different question. Its policy and reporting model concern authentication alignment and handling of messages that claim to come from a domain. It does not replace checking where inbound mail for that domain is routed. Record DMARC evidence in an outbound-authentication track instead of treating it as proof that an inbound receiver works.

## Limits

No finite probe proves delivery to every mailbox. Recipient policy, filtering, mailbox state, and later configuration changes remain outside a single successful test. The practical goal is narrower: prove control, show the intended public route, demonstrate one correlated delivery, and retain enough evidence to explain the next failure.

Do not auto-delete an unexpected record, and do not declare readiness from priority alone. Stop at the mismatch. Let the zone owner reconcile intent with what is publicly delegated, then collect fresh evidence.

## References

- https://datatracker.ietf.org/doc/html/rfc1034
- https://datatracker.ietf.org/doc/html/rfc1035
- https://datatracker.ietf.org/doc/html/rfc5321
- https://datatracker.ietf.org/doc/html/rfc7489
