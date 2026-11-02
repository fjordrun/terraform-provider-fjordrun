---
page_title: "Deployments"
subcategory: ""
description: |-
  How changes reach running services: explicit deployments, driven by triggers.
---

# Deployments

On the API, saving a service's settings or variables redeploys it if it is running. An apply often changes several things
for one service, so with the API's default it would deploy several times, each with half of the changes.

This provider therefore saves changes **without deploying**, and you say when to roll out:

```terraform
resource "fjordrun_deployment" "web" {
  service_id     = fjordrun_service.web.id
  environment_id = fjordrun_service.web.environment_id
  wait           = true

  triggers = {
    config    = fjordrun_service.web.config_hash
    variables = fjordrun_service_variables.web.variables_hash
    shared    = fjordrun_shared_variables.production.variables_hash
  }
}
```

Terraform creates this after what it references, so it runs last, and only when a trigger changed. That makes **one
deployment per apply, per service and environment**, with all the changes in it.

| Change | What to put in `triggers` |
|---|---|
| Source, build, deploy, resources | `fjordrun_service.x.config_hash`, or `fjordrun_service_environment.x.config_hash` |
| Service variables | `fjordrun_service_variables.x.variables_hash` |
| Shared variables | `fjordrun_shared_variables.x.variables_hash` |
| A new volume | `fjordrun_volume.x.id` |
| Function code | `fjordrun_function.x.code_sha256` |
| A new commit on the branch | Pass the SHA in, for example `var.commit_sha` |

Growing a volume needs no deployment, and a domain or public TCP port goes live without one.

If a service gets no `fjordrun_deployment`, nothing runs and changes stay unapplied. The API's `status`
(`fjordrun_service.status`) shows `inactive` for a service that was never deployed.

## Letting a resource deploy by itself

`auto_deploy = true` on a resource, or on the provider for all resources, makes it behave like the API's default: it
deploys when it is created and when it changes. That is convenient for a service managed by one resource alone. Across
resources it means one deployment per resource that changed, each with what is saved so far.

`fjordrun_function` is one resource with its code and settings, so `auto_deploy = true` on it makes one deployment per
apply of that resource, and `wait = true` reports whether it worked.

## What still restarts a service

- Destroying a volume, because its workload has to be recreated without it.
- Rotating a function's token with `auth = "token"`, so the old token stops working.
- Deploying a template: `fjordrun_template_deployment` starts its services in `environment_id`.
- Pushes to a GitHub source's branch, if the repository's webhook is connected. That happens outside Terraform.

## Waiting

With `wait = true` the apply waits until the deployment finishes, and fails unless it succeeded. The `timeouts` block sets
how long (30 minutes by default). A deployment that fails during `create` isn't saved, so the next apply deploys again.
During `update`, the old deployment stays in state, the triggers are still different, and the next apply tries again too.
