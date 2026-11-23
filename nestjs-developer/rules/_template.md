---
title: Rule Title Here
impact: MEDIUM
impactDescription: One line on what following the rule protects or enables (no unmeasured percentages)
tags: tag1, tag2
---

## Rule Title Here

Brief explanation of the rule and why it matters, in a few sentences.

**Incorrect (description of what's wrong):**

```typescript
// Bad code example here, with the resources of _example-domain.md
const bad = example()
```

**Correct (description of what's right):**

```typescript
const good = example()
```

The good code example, with the resources of `_example-domain.md`. Readers copy a correct example's comments into their code, so it keeps only comments a project would keep under `style-code-comments` (a reason from outside the code, a subtle invariant); what the example teaches goes in the prose around the block, and a bare file path line may name the file. The annotations of an incorrect example explain the mistake and stay in its code.

Reference: [Link to documentation or resource](https://example.com)
