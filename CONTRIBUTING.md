# 🤝 Contributing

This repository is still taking shape. PRs are welcome, from small documentation improvements to entirely new Gem proposals.

## Adding a Gem

1. Extract engineering decisions from an implementation that has actually worked or has been sufficiently validated.
2. Remove project-specific information and refine the result into a reusable design.
3. Use the formats under `templates/` and add the Gem under `gems/origin` or `gems/mixed`.
4. Declare `status`, `domains`, `targets`, and `parents` where applicable in `metadata.yaml`.
5. Prioritize contracts, constraints, failure policies, and validation rules over implementation details.

## 🔒 Redaction Rules

Before opening a PR, make sure none of the following remains in the contribution:

- private project, customer, company, or organization names
- personal names, email addresses, phone numbers, bank details, or other identifiers
- private/personal domains or real production API URLs
- absolute local paths such as `C:\\Users\\...`, `/home/...`, or `/srv/...`
- real repository names or internal workspace paths
- secrets, API keys, tokens, private keys, or real cookie names/values
- internal database tables, queues, buckets, hosts, or resource IDs
- detailed file trees that would allow the original project to be reconstructed
- real user data from production logs, prompts, or payloads

Use generalized examples such as `example.com`, `/api/resource`, `example_session`, and `resourceId` when an example is needed.

Names of public libraries, standards, browsers, and frameworks may remain when they are technically relevant.

## Status

- `draft`: the structure is still being defined
- `experimental`: actively used, but likely to change
- `validated`: verified in at least one real implementation
- `stable`: repeatedly applied and the contract is relatively mature
- `deprecated`: no longer recommended

## PR Checklist

- [ ] Project-specific information and personal data have been removed.
- [ ] The problem and engineering intent are clear.
- [ ] Contracts and constraints are explained before implementation details.
- [ ] Failure and recovery policies are included where relevant.
- [ ] Validation rules describe what an LLM-generated implementation must preserve.
- [ ] Framework-specific details are separated as examples rather than treated as the contract.
- [ ] Mixed Gems declare their parent Gems.
