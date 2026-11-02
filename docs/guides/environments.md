---
page_title: "Environments"
subcategory: ""
description: |-
  How services, functions and their settings relate to a project's environments.
---

# Environments

A Fjordrun project has environments, and **every service exists in all of them**, as an instance with its own settings,
variables, domains, volume and deployments. The service itself (its name, kind and icon) is shared. This shapes the
resources:

| What | Resource | Scope |
|---|---|---|
| The service, and its settings in one environment | `fjordrun_service` | Creates and destroys the service everywhere; settings in `environment_id` |
| Its settings in another environment | `fjordrun_service_environment` | One environment; creates nothing |
| Variables | `fjordrun_service_variables` | One environment |
| Domain, public TCP port, volume, deployment | `fjordrun_domain`, `fjordrun_tcp_proxy`, `fjordrun_volume`, `fjordrun_deployment` | One environment |
| The function, its code, and its settings in one environment | `fjordrun_function` | Code is shared by all environments |
| Its settings in another environment | `fjordrun_function_environment` | One environment |

```terraform
resource "fjordrun_service" "web" {
  project_id     = fjordrun_project.shop.id
  environment_id = fjordrun_project.shop.default_environment_id # production
  name           = "web"
  source         = { type = "github", repo = "acme/shop", branch = "main" }
  deploy         = { replicas = 3 }
  depends_on     = [fjordrun_environment.staging]
}

resource "fjordrun_service_environment" "web_staging" {
  service_id     = fjordrun_service.web.id
  environment_id = fjordrun_environment.staging.id
  source         = { type = "github", repo = "acme/shop", branch = "develop" }
  deploy         = { replicas = 1 }
}
```

Rules that follow from the API:

- **Create environments first.** When a service or template is created, all existing environments get its settings,
  variables and volumes. An environment created later gets inactive instances with default settings and no variables,
  unless it forks from another with `fork_from_environment_id`. Hence the `depends_on` above.
- **After creation, a change applies to one environment.** Changing `fjordrun_service.deploy` doesn't touch staging.
- **One resource per environment.** Don't point `fjordrun_service_environment` at the environment `fjordrun_service` manages:
  they would overwrite each other.
- **Destroying `fjordrun_service_environment` can't reset the instance.** The API has no such operation, so the settings stay
  as they are. `on_destroy` says whether the instance also stops (`stop`, the default) or keeps running (`keep`).
- **`production` needs no resource.** It comes with the project, as `fjordrun_project.default_environment_id`. The API can't
  rename environments, so changing `fjordrun_environment.name` replaces the environment, with its services' data.
