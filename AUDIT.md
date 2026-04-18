• Given a true single-user, self-hosted deployment on a personal Cloudflare account, I’d downgrade several of the earlier findings. The app is closer to
  a personal tool than a multi-tenant product, so “any Access-authenticated user can do everything” is mostly the intended trust model.

  Still Important

  - Inbound email can trigger AI spend and work. This remains the biggest practical risk. Anyone who can email your routed address can cause parsing, R2
    writes, DO writes, prompt-injection scanning, and auto-draft generation via workers/index.ts:405 and workers/agent/index.ts:465. For a personal CF
    account, this is more of a cost/abuse issue than a privilege issue. I’d add an autoDraft.enabled setting, per-mailbox inbound rate limiting, and
    perhaps skip auto-draft for unknown senders.
  - Preview/dev can touch real resources. remote: true for email and the same preview/production R2 bucket in wrangler.jsonc:21 and wrangler.jsonc:28 are
    still risky for personal use because local testing can affect real mail/data. Use a separate preview bucket and avoid real send bindings in dev
    unless explicitly needed.
  - Mailbox deletion leaves data behind. In workers/index.ts:139, deleting a mailbox removes only the R2 settings file. For single-user use this is less
    of an access-control issue, but it is still a data-retention and surprise-recovery issue. Recreating the same mailbox can expose old DO state. I’d
    either rename the action to “remove mailbox from list” or implement full purge.
  - Sent emails are recorded before delivery succeeds. The API writes to Sent before async delivery in workers/index.ts:187 and workers/index.ts:204.
    Same pattern exists for reply/forward. This is mainly correctness now: your UI can say “sent” when delivery failed. Use an outbox/pending state or
    await the send before writing Sent.
  - Dependency advisories should be fixed. This does not change with single-user scope. npm audit --omit=dev reported production advisories including
    direct drizzle-orm, hono, and dompurify dependencies in package.json:39. Update dependencies and rerun audit/typecheck.

  Lower Priority In Single-User Mode

  - No per-mailbox authorization. Acceptable if Cloudflare Access allows only you and you trust all connected MCP/agent clients. This would become high
    severity only if you invite other users or share the Access policy.
  - MCP can read/send/delete across all mailboxes. For a personal tool, this is a conscious trust boundary: any MCP client you connect is effectively an
    operator for the inbox. The risk is not “another app user can access mail”; it is “a compromised or over-permissioned local AI/MCP client can act as
    you.” Reasonable mitigation: keep MCP disabled unless needed, or gate MCP behind a separate Access policy/token.
  - Arbitrary mailbox creation when EMAIL_ADDRESSES is empty. In a single-user deployment this is mostly convenience. I would still validate against
    DOMAINS so accidental typo/spoof-like mailboxes do not get created, but it is not a high-severity issue in your stated model.
  - Custom system prompt replaces the default prompt. Since only you can edit settings, this is not an authorization problem. It is still fragile
    architecture: the safety/tool-use policy and writing-style customization should ideally be separate, but it can be backlog.
  - Outgoing attachment limits. Since only you can call the authenticated send endpoint, unbounded outbound attachments are mainly accidental OOM/cost
    risk. Add limits eventually, but it is not urgent unless MCP or browser access is exposed beyond you.

  Revised Priority

  1. Add auto-draft opt-in/rate limits for inbound mail.
  2. Separate preview/dev resources from production R2/email.
  3. Fix mailbox deletion semantics or implement full purge.
  4. Change send flow so Sent reflects actual delivery success.
  5. Update vulnerable dependencies.
  6. Optionally enforce DOMAINS and add a separate MCP gate for defense in depth.

  So the architecture is much more defensible as a personal app. The remaining serious risks are mostly cost, accidental production data mutation,
  misleading state, and dependency hygiene rather than multi-user data exposure.
  
