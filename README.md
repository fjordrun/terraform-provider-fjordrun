# Terraform provider for Fjordrun

The official [Terraform](https://www.terraform.io) and [OpenTofu](https://opentofu.org) provider for
[Fjordrun](https://fjordrun.com). It manages teams, projects, environments, services, functions, domains, volumes,
private networks, agents and workflows.

This repository holds the signed release binaries and the documentation. See [docs/](docs/index.md) and
[Releases](../../releases).

```terraform
terraform {
  required_providers {
    fjordrun = {
      source = "fjordrun/fjordrun"
    }
  }
}

# The token comes from FJORDRUN_TOKEN: a team API token (fjr_...) or a personal access token (fjr_pat_...).
provider "fjordrun" {}
```

Questions: support@fjordrun.com. Licence: [LICENSE](LICENSE).
