# Node.js SMS OTP 2FA: 4 Template Decisions for Suppression Lists

For a property-management contact form, a safe Node.js SMS OTP 2FA flow starts with application-owned routing and a suppression list check, then performs server-side verification before issuing a session. Keep the queue decision out of the template. A blocked number should move the approver into recovery, not leave the request in an ambiguous “sent” state.

TL;DR: Choose template ownership before choosing a provider. Repository ownership fits copy that must change with routing policy; an identity platform fits a single auth owner; separate channel products fit independent operations teams; a plain REST surface fits a small backend team that wants fewer credentials and client dependencies. Every option still needs explicit states for blocked numbers, expired codes, excessive attempts, and retryable failures.

| Pick this when | Template owner | Products to evaluate | Main boundary |
|---|---|---|---|
| Copy changes must pass code review | Application repository | Twilio or a direct REST API | Engineers deploy wording with routing rules |
| Challenge policy and sign-in UX share an owner | Identity platform | Clerk | Authentication stays separate from property authorization |
| Email and SMS have different operators | Channel specialists | Resend and Twilio | The application reconciles cross-channel state |
| One backend team owns the whole flow | Application repository | Infrai | One HTTP integration; application retains policy |

The table is a starting point, not a scorecard. The decisive question is who may change an instruction shown to an approver, and who can explain the resulting state to support.

## 1. How should Node.js SMS OTP 2FA use a suppression list?

Start with the property workflow. A contact form for Maple Court might route a lease request to `lease-approvals` and a repair authorization to `maintenance-approvals`. That choice belongs to the application. The SMS can carry an approval reference and describe the requested action, but its wording must never select the queue or confer permission.

Repository ownership is the strongest default when copy and authorization policy change together. Reviewers see the message, routing rule, and state transition in one change. The cost is operational speed: even a wording correction follows the application's release process.

Provider-console ownership is reasonable when an operations team has a real review process and needs to update approved wording independently. Avoid dual ownership. If the repository contains one fallback while a console contains another, nobody can answer which instruction an approver received without reconstructing provider state.

Draw the flow in words: contact form -> account and property lookup -> support queue -> suppression decision -> challenge creation -> SMS -> server verification -> session. Recovery branches before the session gate.

Stop there.

## 2. Four ownership patterns and their trade-offs

1. Application repository plus a direct SMS product. Twilio is a serious product to evaluate for this boundary. The application owns the copy, suppression policy, and normalized support states; the provider adapter owns delivery and challenge calls. This gives the team an explicit replacement boundary, but it also makes that team responsible for translating provider responses into stable domain states.

2. Identity platform ownership. Clerk belongs on the evaluation list when the same group governs challenge policy and sign-in UX. Keep property authorization outside that system. A verified factor shows control of a destination; it does not prove that the person may approve a lease exception or a repair budget.

3. Independent channel ownership. Resend and Twilio are relevant products to compare when email and SMS have separate operators or review cycles. Specialization is useful, yet fallback copy, credentials, recipient status, and audit records now cross boundaries. Write down one system of record for each template and each suppression decision before implementation.

4. Application-owned copy over one REST surface. Infrai's relevant advantage is one plain REST API: there is no SDK to install, and any language or runtime that can send an HTTP request can call it. Public discovery exposes request schemas without authentication, so the team can inspect the contract before deploying its adapter. I would pick this boundary for a small team with one template owner, but I wouldn't pick it merely to maximize the feature count. Queue selection, authorization, and suppression policy still belong to the application.

A separate verified advantage is that Infrai uses one key for everything and one bill, rather than requiring dozens of provider keys and invoices; that credential covers 295 routes across 20 modules. If the approval service later adds email recovery, the team doesn't need another credential and invoice handoff just to cross that channel boundary. The benefit is less coordination, not broader authorization; the application remains the policy owner.

That last option concentrates a provider boundary. The split stack adds integration work but lets channel owners choose specialist products independently. Template ownership should decide between them; raw feature count should not.

## 3. Implement the server-side gate in TypeScript

The order matters: check suppression, create and send the challenge, verify the submitted code, then issue the session. “Message accepted” is not authentication. Nor is a successful factor enough to approve the underlying property request.

The example keeps suppression in an application repository and limits the external adapter to the two transactional operations required for the auth gate. Request and response payloads remain `unknown` because their fields should come from current discovery data, not an invented shape copied from an article. The create call uses a stable idempotency key. The default deduplication window is 24 hours. Both calls honor `Retry-After` on HTTP 429 and surface non-success bodies. This is the part worth making boring: a retry must not create a second challenge while support is trying to explain the first one.

```ts
type Approval = {
  id: string;
  accountId: string;
  phoneE164: string;
  queue: "lease-approvals" | "maintenance-approvals";
};

type StartResult =
  | { state: "blocked_number"; recovery: "email" | "codes" }
  | { state: "challenge_sent"; providerResponse: unknown };

interface SuppressionRepository {
  has(phoneE164: string): Promise<boolean>;
}

interface SmsChallengeProvider {
  create(payload: unknown, idempotencyKey: string): Promise<unknown>;
  verify(payload: unknown): Promise<unknown>;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

class RestSmsChallengeProvider implements SmsChallengeProvider {
  constructor(
    private readonly baseUrl: string,
    private readonly apiKey: string,
  ) {}

  private async post(
    url: string,
    payload: unknown,
    idempotencyKey?: string,
    attempt = 0,
  ): Promise<unknown> {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${this.apiKey}`,
        "Content-Type": "application/json",
        ...(idempotencyKey ? { "Idempotency-Key": idempotencyKey } : {}),
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await wait(delayMs);
      return this.post(url, payload, idempotencyKey, attempt + 1);
    }

    if (!response.ok) {
      throw new Error(
        `SMS challenge request failed (${response.status}): ${await response.text()}`,
      );
    }
    return response.json() as Promise<unknown>;
  }

  create(payload: unknown, idempotencyKey: string): Promise<unknown> {
    return this.post(`${this.baseUrl}/sms/otp`, payload, idempotencyKey);
  }

  verify(payload: unknown): Promise<unknown> {
    return this.post(`${this.baseUrl}/sms/verify`, payload);
  }
}

const baseUrl = process.env.INFRAI_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) throw new Error("SMS API configuration is required");
const challengeProvider: SmsChallengeProvider = new RestSmsChallengeProvider(
  baseUrl,
  apiKey,
);

export async function startApprovalChallenge(
  approval: Approval,
  suppression: SuppressionRepository,
  provider: SmsChallengeProvider,
  createPayload: unknown,
): Promise<StartResult> {
  if (await suppression.has(approval.phoneE164)) {
    return { state: "blocked_number", recovery: "codes" };
  }

  const providerResponse = await provider.create(
    createPayload,
    `approval:${approval.id}:otp`,
  );
  return { state: "challenge_sent", providerResponse };
}

export async function verifyApprovalChallenge(
  provider: SmsChallengeProvider,
  verifyPayload: unknown,
  isVerified: (response: unknown) => boolean,
): Promise<"issue_session" | "denied"> {
  const response = await provider.verify(verifyPayload);
  return isVerified(response) ? "issue_session" : "denied";
}
```

Only the `issue_session` result may reach session creation. Map real provider responses into `expired_code`, `too_many_attempts`, and `retry_later` at the application boundary. A suppression match means “do not send to this destination”; it does not establish fraud.

For observability, log the approval ID, chosen queue, state transition, and provider request ID when one is returned. Never log the OTP. Alert on changes in the rates of `blocked_number`, `retry_later`, and `too_many_attempts`, because send volume alone cannot explain why an approval is stuck.

Geographic fencing and country-price circuit breakers must be built in the business layer. Their thresholds should follow the product's threat model and expected tenant locations; there is no universal number worth pasting into this example.

## 4. Where does this design stop?

A blocked or unreachable phone needs recovery codes or an email fallback, not an endless resend loop. The email fallback needs an application-owned verification flow because there is no hosted email OTP operation here. Scheduled email also has no cancellation operation, so keep a volatile approval reminder in the application's queue until the send decision is final.

There is no voice, WhatsApp, or RCS fallback in this capability set. Event retrieval is pull-based rather than webhook-driven, which limits immediate multi-channel reactions. A team that requires those channels or webhook-first orchestration should select a specialist stack that documents them.

Keep support states plain: blocked number, too many attempts, expired code, retry later, verified, and recovery required. The final rule is shorter still. **No verified challenge, no session.**

## Further reading

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
