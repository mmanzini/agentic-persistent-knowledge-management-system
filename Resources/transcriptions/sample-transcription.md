# Sample transcription — interview with a platform engineer

**Recorded:** 2026-04-15
**Participants:** Interviewer (I), Platform engineer (P)
**Duration:** 12 minutes (excerpt)

---

**I:** What's the biggest source of toil for your platform team right now?

**P:** Honestly, it's the same thing it was two years ago — secret
rotation. We have a sane secret store now, but the rotation cadence
isn't enforced anywhere. Teams set up a credential, ship the feature,
and the credential lives forever. We caught one last quarter that had
been valid since 2021.

**I:** Have you tried automating expiry?

**P:** We've talked about it. The blocker isn't technical, it's that
half the consumers of these credentials are services we don't own —
legacy contractors mostly — and we don't have a clean rollover story
for them. So we end up rotating manually, which means we mostly don't.

**I:** What would unblock it?

**P:** Honestly, a six-month deprecation window with loud warnings,
and exec air cover when the contractor relationships start screaming.
We've got the tooling. We don't have the political cover to break
things on purpose.
