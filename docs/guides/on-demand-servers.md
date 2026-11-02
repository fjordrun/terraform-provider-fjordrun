---
page_title: "On-demand servers"
subcategory: ""
description: |-
  How fjordrun_sftp_server and fjordrun_ssh_server treat servers that stop by themselves.
---

# On-demand servers

`fjordrun_sftp_server` and `fjordrun_ssh_server` start servers that aren't meant to last. A server:

- stops after **15 minutes without a connection**, and after **8 hours** in any case;
- stops when an SSH key of the user who started it is removed, and, for SFTP, when the service is deployed, because the
  volume attaches to one pod at a time;
- gets a **new port** with every start;
- accepts the SSH keys its user had at the moment it started.

So these resources can't mean "this server is running", as a resource normally would. They mean "Terraform started this
server", and say honestly what happened since:

| Situation | Plan | State |
|---|---|---|
| Server running | Empty | `running = true`, connection details set |
| Server stopped by itself | **Empty**: this is normal, not drift | `running = false`, `status = "stopped"`, connection details null |
| Same, with `restart_when_stopped = true` | Create: the next apply starts a new one | Gone from state |
| `triggers` changed | Update: starts it if it isn't running | Connection details of the running server |
| Service deleted | Create, which then fails as the service is gone | Gone from state |

Treating a stop as drift by default would make every plan an hour later show a change, and every apply start a server
nobody asked for, billed as long as it runs. Hence `restart_when_stopped = false` by default.

Consequences:

- **Don't feed `port` or `command` into long-lived configuration.** They become null when the server stops, and change with
  every start. These resources suit a short task: start, use, destroy.
- Reading the current state without starting anything is what the data sources `fjordrun_sftp_server` and
  `fjordrun_ssh_server` do.
- The SSH server needs a running replica, so create it after the `fjordrun_deployment` (`depends_on`). The SFTP server needs a
  volume, and should come after any deployment in the same apply, which would stop it.
- Both need the Hobby plan or higher, and an SSH key on the account of the token's user (`fjordrun_ssh_key`).
