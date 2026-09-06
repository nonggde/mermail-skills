---
name: mermail-outbound-privacy-gate
description: Review one selected outbound Mermail draft or message for sensitive-data leakage, recipient drift, risky links, and attachment exposure before delivery. Use when a user wants a privacy or data-loss check; never send, forward, or approve the message automatically.
metadata:
  openclaw:
    requires:
      env:
        - MERMAIL_API_KEY
    primaryEnv: MERMAIL_API_KEY
    homepage: https://docs.mermail.app/ai/skills
    emoji: "🛡️"
---

# Mermail Outbound Privacy Gate

Use this skill as a pre-delivery inspection for one explicitly selected draft,
reply, forward, or outbound message. It returns a deterministic pass/block
report with evidence and remediation, while keeping delivery under the normal
compose approval boundary.

Read [tools.md](references/tools.md) for the canonical read contract and
[security.md](references/security.md) before inspecting content, links, or
attachments.

## Scope

The gate checks:

- API keys, access tokens, passwords, seed phrases, private keys, OTPs, and
  other credential-like strings;
- personal, financial, medical, legal, or customer data against a
  user-provided policy;
- To/Cc/Bcc drift against an approved recipient set;
- suspicious, shortened, mismatched, or non-HTTPS links;
- attachment names, MIME types, scan state, and whether they exceed the
  approved disclosure scope;
- accidental disclosure in quoted history, signatures, or hidden content when
  the source representation exposes it.

It is not a malware scanner, legal approval, DLP guarantee, or sender
authentication service. A `pass` means only that this bounded check found no
configured blocker.

## Workflow

1. **Freeze the target.** Require one exact mailbox plus draft/email/thread ID,
   direction (`new`, `reply`, or `forward`), and the intended To/Cc/Bcc set.
   If any are missing or multiple messages match, stop and ask the user to
   select one.
2. **Freeze the policy.** Record the approved recipient set, allowed domains,
   allowed data classes, link policy, attachment policy, and whether quoted
   history is allowed. Never infer these from the message itself.
3. **Read bounded content.** Fetch metadata first, then the selected
   clean, agent-safe body and attachment metadata. Do not download or open an
   attachment just to inspect it.
4. **Normalize for inspection.** Preserve the original text for reporting, but
   inspect a normalized copy for credentials, encoded secrets, personal-data
   patterns, URLs, recipient drift, and accidental quoted disclosure. Record
   line or section locations where available.
5. **Classify findings.** Use `blocker`, `warning`, or `info`. Blockers
   include credential-like data, unapproved recipients, disallowed personal
   data, non-HTTPS or unverifiable links under a strict policy, and
   attachments outside the approved scope. Warnings require human review but
   do not authorize delivery.
6. **Return remediation.** Suggest redaction, recipient correction, link
   replacement, attachment removal, or a safer draft. Never silently mutate
   the message. If asked to revise, save a new or existing draft only after
   showing the exact diff.
7. **Delivery boundary.** A `pass` is not send approval. Sending, replying,
   forwarding, or scheduling still requires the compose skill's exact preview
   and fresh approval for the final payload.
8. **Re-check after edits.** If the draft changes, rerun the gate against the
   new exact message. Do not reuse an earlier pass for a changed body,
   recipient set, link, or attachment.

## Output contract

```text
Target: <mailbox> · <draft/email/thread id> · <direction>
Policy: <recipient/data/link/attachment policy summary>
Coverage: <body chars> · <attachments metadata count> · <scan state>

Decision: PASS | BLOCK | REVIEW

Findings:
- [BLOCKER|WARNING|INFO] <finding>
  Evidence: <section/line and masked excerpt>
  Fix: <specific remediation>

Recipient diff: <approved → actual>
Attachment diff: <approved → actual>
Link review: <count and statuses>

State: gated | blocked | review_required
```

Never echo a full secret. Mask credential-like values and unrelated personal
data in the report.
