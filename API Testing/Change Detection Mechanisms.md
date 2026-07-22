Here are the 3 mechanisms in plain terms:

**1. Scheduled re-validation** Just re-run all the tests regularly (e.g. every night) and see what breaks.

- ✅ Simple, works even without a spec
- ❌ Tells you _something_ broke, not _what_ changed or _why_

**2. Spec-diff-triggered regeneration** Compare the client's new API spec against the old one, and only update the tests for whatever actually changed.

- ✅ Precise — knows exactly what changed (new field, removed endpoint, etc.), only touches affected tests
- ❌ Only works if the client keeps their spec accurate — useless if it's outdated or missing

**3. Continuous monitoring** Constantly watch the live API against a reference baseline, instead of checking only at release time.

- ✅ Catches changes the moment they happen, not just at releases
- ❌ Vaguest and most expensive to build — unclear what "reference" even means yet

**Simplest way to think about it:**

- Mechanism 1 = "check everything, see what breaks"
- Mechanism 2 = "check what changed, update only that"
- Mechanism 3 = "watch constantly, don't wait for a release"

Spec-diff (#2) fits the requirements best _if_ the client has a spec — otherwise re-validation (#1) is the fallback. Continuous monitoring (#3) is likely too heavy for now.