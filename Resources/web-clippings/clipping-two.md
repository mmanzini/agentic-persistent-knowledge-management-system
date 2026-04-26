# Notes on the case against premature abstraction

**Source:** https://example.com/premature-abstraction
**Author:** A. Engineer
**Captured:** 2026-04-23

Argues that the cost of a bad abstraction is paid every time a reader
has to unwind it, while the cost of three similar lines of code is
paid once when someone eventually unifies them. The post leans on
Sandi Metz's "duplication is far cheaper than the wrong abstraction"
to argue against DRY-at-all-costs.

The piece notes that the inverse failure — refusing to abstract
genuinely repeated logic — also exists, but is rarer in codebases
written by experienced engineers.
