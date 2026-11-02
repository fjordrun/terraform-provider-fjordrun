---
page_title: "Secrets and state"
subcategory: ""
description: |-
  Which secrets end up in the Terraform state, and how to keep it safe.
---

# Secrets and state

Attributes holding secrets are marked sensitive, so plans and logs don't show them. They are **still written to the state
file, in clear text**. Keep the state in a remote backend that encrypts it and restricts who can read it, such as an
encrypted S3 bucket, Terraform Cloud, or OpenTofu's state encryption. Don't commit it.

| Attribute | In state because | Notes |
|---|---|---|
| `fjordrun_team_secret.value` | You set it, and the API never returns it | A change made elsewhere moves `updated_at`, which the provider treats as drift: the next apply writes the configured value again. It can't tell what the other value was |
| `fjordrun_team_token.token` | The API returns it only when the token is created | Lost if not in state: replace the token to get a new one. Empty after an import |
| `fjordrun_service_variables.variables`, `fjordrun_shared_variables.variables` | You set them | |
| `fjordrun_function.token`, `fjordrun_function_environment.token` | The API returns it | Rotate with `rotate_token_trigger` |
| `fjordrun_workflow.webhook_url` | It contains a secret | |
| `fjordrun_team_invite.accept_url` | It contains the invite's token | Only known to the run that created the invite |
| Data sources `fjordrun_service_variables`, `fjordrun_shared_variables`, `fjordrun_service_access`, `fjordrun_function_token` | They read secrets | Read them only where needed |

## Variables

`variables` holds what you write, not what it resolves to:

- A **reference** such as `${{Postgres.DATABASE_URL}}` stays a reference, so a changing database password is no diff.
  Note that in HCL a literal `${` is written `$${`: `"$${{Postgres.DATABASE_URL}}"`.
- A **generator** such as `$${{secret(32)}}` is expanded by the API, once. The state keeps the generator as you wrote it, and
  as long as the stored value is one it could have produced, it is left alone, also when other variables change. Change
  the generator (`secret(48)`) to rotate it.
- To read the resolved values, use the data sources. That puts them in the state too.

`variables_hash`, for `fjordrun_deployment` triggers, is sensitive as well, because it is derived from secret values: a plan
shows `triggers` as `(sensitive value)` when it contains it.
