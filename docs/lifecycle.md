# 🔄 Gem Lifecycle

```text
Mine → Refine → Stash → Socket → Craft → Validate → Upgrade
 ↑                                                        │
 └────────────────────────────────────────────────────────┘
```

## Mine

Find responsibilities, contracts, failure policies, security decisions, and other engineering choices worth reusing in a system that actually works.

## Refine

Remove project names, internal paths, domain-specific naming, and accidental implementation details. Preserve the engineering intent and constraints needed to reproduce the design.

## Stash

Store one Gem per directory and attach searchable metadata.

## Socket

Check whether the Gem's responsibility matches a responsibility in the new project. The framework or language does not need to be the same.

## Craft

When the contracts of multiple Gems do not align, avoid forcing the Gems together. Create an explicit adapter or glue design for the gap.

## Validate

Use schemas, contracts, security rules, automated tests, and other checks to define the acceptance boundary of the generated implementation.

## Upgrade

Feed new decisions discovered in real-world use back into the existing Gem, or extract them into a new Mixed Gem when they represent integration-specific knowledge.
