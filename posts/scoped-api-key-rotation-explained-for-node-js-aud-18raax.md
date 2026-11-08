# Scoped API Key Rotation Explained for Node.js (Auditing CI Pipeline Access)

TL;DR: Give a CI pipeline a scoped API key with least privilege for its test operation and customer boundary. Keep the key out of build logs, plan its rotation, and record authorization decisions using a separate credential identifier. For an e-commerce service that meters per-customer usage for an invoice, the critical question is whether you can reconstruct which identity could submit usage for which customer. A masked log alone cannot answer that.

A scoped API key is a credential whose authority the receiving service restricts. Putting a customer identifier in a request does not establish that restriction. The service must check the authenticated caller's allowed customer and operation before accepting the request.

The OWASP Secrets Management Cheat Sheet supports least privilege, secret lifecycle management, auditing, and rotation. The design below applies those principles to a concrete billing test. Its identifiers and permissions are illustrative application choices, not a commercial API contract.

## Start with the access trail

Before: a build holds a shared credential, submits usage, and prints the request when a test fails. An invoice investigation finds a customer identifier and a timestamp. That leaves an attribution gap: a customer field describes the target, while a shared secret does little to distinguish the callers that possessed it.

After: a dedicated CI identity can submit synthetic usage for one test customer. The service records its credential identifier, the requested operation, the target customer, the run identifier, and the authorization result. Production invoice usage belongs to a separately authorized runtime identity.

Keep those identities apart.

Diagram in words: build job → secret injection → authenticated request → authorization check → usage acceptance. A separate arrow leaves the authorization check for the audit sink. The secret stops at authentication; its nonsecret identifier continues into the audit event. A run identifier supplies correlation, but the authenticated identity supplies authority. Treat run metadata sent by the caller as caller-supplied context.

This separation adds policy and audit plumbing. It also lets an invoice reviewer ask a useful question: did the service accept this customer's usage under the expected runtime identity, or did a test identity attempt to cross its boundary? You cannot recover that distinction from a total request counter.

## What should a scoped API key allow in a CI pipeline?

For this example, allow one operation, `usage:submit`, against customer `test-shop-042` in the test environment. Deny invoice finalization and credential administration. Those are proposed permission names; define their meaning in the receiving service and test the enforcement there. Merely naming a key "test-only" supplies no protection.

In a GitHub Actions workflow, distinguish the credential used to access repository resources from the credential your usage service accepts. Review both permission boundaries. Do not assume a workflow permission setting constrains an unrelated service's API key. The authorization decision in this example lives at that service.

Build logs deserve their own boundary.

Avoid dumping environment variables, authorization headers, or complete request objects. A failure message should expose the operation, outcome, and a safe correlation identifier. Logging a rejected request verbatim can turn a successful authorization denial into a secret disclosure. That is the trap.

For auditability, choose explicit event fields over a general-purpose object serializer. Do not put customer names or invoice contents in the event when an internal identifier is sufficient. Limit who can read the audit store and who can change its records; useful attribution depends on the integrity of the evidence as well as its presence.

## Make the boundary executable in TypeScript

Here is a small authorization example for a Node.js service. It uses no network endpoints and accepts no raw secret. Authentication must first resolve a verified credential to a trusted principal; never construct that principal from customer or environment fields supplied in the request.

```ts
type Principal = {
  credentialId: string;
  environment: "test" | "production";
  customerId: string;
  permissions: readonly string[];
};

type Attempt = {
  operation: string;
  customerId: string;
  environment: "test" | "production";
};

function authorize(principal: Principal, attempt: Attempt) {
  const allowed =
    principal.environment === attempt.environment &&
    principal.customerId === attempt.customerId &&
    principal.permissions.includes(attempt.operation);

  return {
    allowed,
    audit: {
      credentialId: principal.credentialId,
      operation: attempt.operation,
      customerId: attempt.customerId,
      environment: attempt.environment,
      outcome: allowed ? "allow" : "deny",
    },
  };
}

const ci: Principal = {
  credentialId: "ci-metering-test-a",
  environment: "test",
  customerId: "test-shop-042",
  permissions: ["usage:submit"],
};

const cases: Array<[Attempt, boolean]> = [
  [{ operation: "usage:submit", customerId: "test-shop-042",
     environment: "test" }, true],
  [{ operation: "usage:submit", customerId: "test-shop-043",
     environment: "test" }, false],
  [{ operation: "usage:submit", customerId: "test-shop-042",
     environment: "production" }, false],
  [{ operation: "invoice:finalize", customerId: "test-shop-042",
     environment: "test" }, false],
];

for (const [attempt, expected] of cases) {
  const decision = authorize(ci, attempt);
  if (decision.allowed !== expected) {
    throw new Error("Authorization boundary regression");
  }
}
```

The 4 cases give the permission a falsifiable meaning: 1 permitted submission and 3 denied boundary crossings. The surrounding request handler must enforce the returned decision before changing usage, then attach a server timestamp and request identifier to its audit record. This function is a policy example, not a complete metering handler. Quantity validation, duplicate submission handling, and durable storage still need their own contracts. Consider the second case during review: the permission string matches, and the environment matches, but the customer changes from `test-shop-042` to `test-shop-043`. The entire authorization decision must fail. If a request handler checks only the operation, that test would expose the missing customer boundary; if it writes usage before checking the decision, an HTTP rejection would conceal an invoice mutation. Assert on stored usage as well as the response.

Order matters here.

Test that full path too. A rejected cross-customer request must leave invoice usage unchanged. A permitted submission must carry the authenticated credential identifier into the audit trail. During deployment, exercise these cases against the same authorization path used by real requests so a test-only policy cannot create false confidence.

## Isn't masking enough, and won't rotation break the build?

Masking helps reduce accidental disclosure in displayed output. It does not establish an authorization boundary or invalidate a credential that has escaped. Keep output narrow even when masking is configured. If a build log exposes a usable secret, treat that credential as compromised: revoke it, issue a replacement with the intended scope, update its consumers, and examine its access history. Deleting the visible line cannot establish that nobody copied it.

Rotation has a real coordination trade-off. For routine replacement, a bounded overlap can let consumers move before the old credential is revoked, if the service supports that lifecycle. The trade-off is availability versus the time both credentials remain usable. For a known disclosure, overlap preserves the exposed credential's authority; containment takes priority over keeping the next build green. Make that distinction explicit in the runbook.

Verify retirement.

A successful request using the replacement proves only that the replacement works. Also verify that the old credential is rejected. Preserve the old nonsecret identifier in audit history so the rotation does not sever the link between earlier access and the identity that performed it. Never retain the raw secret merely to make an investigation easier.

For observability, count authorization denials by operation and environment, and investigate unexpected attempts to reach production from the test identity. Keep individual customer and credential identifiers in access-controlled audit events instead of turning every identifier into a metric dimension. Alert thresholds and retention periods are local decisions: choose them from expected build activity, investigation needs, and storage constraints. There is no measured threshold established by this example.

The acceptance rule is concrete: a test build can submit only its synthetic customer's usage; a rejected request cannot alter a metered invoice; and the audit trail can connect each accepted operation to an authenticated credential identifier. That is a useful review standard for least privilege, log safety, and rotation together.

## References

The OWASP Secrets Management Cheat Sheet is the source for the secret-management principles discussed here. The customer identifiers, policy function, test cases, and operational acceptance rule are an illustrative design applying those principles.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
