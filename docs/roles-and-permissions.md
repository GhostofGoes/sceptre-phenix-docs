# Roles and Permissions

When [authentication](user-administration.md#authentication-modes) is
enabled, phēnix uses role-based access control (RBAC) to decide what each
user can do. Every user has exactly one role, and a role is a list of
policies that grant permissions on resources. This page explains how
permissions are checked, describes the built-in roles, and lists every
resource and verb phēnix checks.

## How Permissions Are Checked

A role is a list of policies. Each policy has:

* `resources`: the resources it applies to, such as `experiments` or
  `vms/snapshots`.
* `verbs`: the actions it allows, such as `list`, `get`, or `update`, or `*`
  for all verbs.
* `resourceNames` (optional): the objects it applies to, such as experiment
  names.

```yaml
policies:
- resources:
  - experiments
  - "experiments/*"
  resourceNames:
  - exp-a
  verbs:
  - list
  - get
```

phēnix applies these rules:

* **Wildcards.** Resources and resource names use shell-style patterns (`*`,
  `?`, `[...]`), where `*` never matches `/`. So `experiments/*` matches
  `experiments/start` but not `experiments`, and `*` matches `experiments` but
  not `experiments/start`. Global Admin uses both `*` and `*/*` to match
  everything. Likewise, `scorch` does not match `scorch/terminals`.
* **Named and unnamed checks.** Many checks name the object being accessed,
  such as the experiment, the VM as `<experiment>/<vm>`, or a config as
  `<Kind>/<name>`. A named check passes only if a matching policy lists a
  matching resource name; a policy without `resourceNames` never passes a
  named check. A name prefixed with `!` excludes matching objects. Checks that
  don't name an object, such as creating an experiment or opening the Builder,
  ignore `resourceNames`. The [Resources](#resources) tables show what each
  check is named with.
* **Lists.** Listing needs the permission without a name, and then filters
  the list by name, as described for each resource below.
* **Live updates.** The web UI receives live updates, such as VM state
  changes, over a websocket, filtered by the permissions described below.
* **Users hold a copy of their role.** Assigning a role copies its policies
  into the user, scoped to the user's resource names. Editing a role config
  afterward does not change users who already have the role; reassign the
  role on the `Users` tab to apply the changes. Self sign-up users are the
  exception: until an administrator assigns them a role, they use the
  `disabled` role config directly.
* **Role changes and the UI.** The web UI keeps the role it received at sign
  in, so after a user's role changes, the user should sign out and back in.
* **The UI only hides controls.** The web UI hides controls a user's role
  doesn't allow, but what a user can do is decided by the server's permission
  checks, not by what the UI shows.

## Scoping a Role to Experiments

When a role is assigned to a user, the space-separated `Resource Name(s)` from
the `Users` tab (or the extra fields of a [`ui.users`](user-administration.md#from-configuration-uiusers)
entry) are copied into the role's policies that don't already name their
resources, in order, stopping at the first policy that does. This is how roles
such as Experiment User and Scorch Viewer are limited to the user's
experiments.

* Experiment checks use the experiment name, such as `exp-a`.
* VM checks, including port forwards, use `<experiment>/<vm>`, so enter both
  the experiment and a VM pattern, for example `exp-a exp-a/*`.
* Names are patterns, so `exp-*` matches every experiment whose name starts
  with `exp-`. A name prefixed with `!` excludes matches, so
  `exp-* exp-*/* !exp-secret !exp-secret/*` covers every `exp-` experiment and
  its VMs except `exp-secret`.
* Leaving `Resource Name(s)` empty is the same as `*`, which matches every
  experiment but no VM, because `*` doesn't match `/`. For access to every
  experiment and VM, enter `* */*`. The same applies to
  [`ui.users`](user-administration.md#from-configuration-uiusers) entries
  without resource names.
* Roles whose policies all name their resources, such as Global Admin, Global
  Viewer, and Builder, ignore these names and are not scoped to experiments.
* Builder, Scorch, and Tunneler access (`builder`, `scorch`,
  `scorch/terminals`, and `tunneler`) is checked without a name, so scoping
  does not limit it. Scorch data is still limited to experiments the user can
  read.

## Built-In Roles

`Global Admin` is the administrator role and has access to everything,
including user management. The following table gives a high-level overview of
the built-in roles.

| Role              | Limits                                                                                                                   | List  |  Get  | Create | Update | Patch | Delete |
|-------------------|:-------------------------------------------------------------------------------------------------------------------------|:-----:|:-----:|:------:|:------:|:-----:|:------:|
| Global Admin      | Can see and control absolutely anything/everything.                                                                      | E V U | E V U | E V U  | E V U  | E V U | E V U  |
| Global Viewer     | Can see everything, use VM consoles and port forwards, and mount VM disks, but can't change experiments or users.         | E V U | E V U |        |        |       |   V    |
| Experiment Admin  | Can see, start, and stop assigned experiments and fully control their VMs, but can't create or delete experiments.       | E V   | E V   |   V    | E V    |   V   |   V    |
| Experiment User   | Can see assigned experiments and change, redeploy, snapshot, capture, and port-forward their VMs, but can't start, stop, or kill VMs. | E V   | E V   |   V    |   V    |   V   |   V    |
| Experiment Viewer | Can see assigned experiments and their VMs, and use VM consoles and port forwards, but can't change experiments or VMs.   | E V   | E V   |        |        |       |        |
| VM Admin          | Can see assigned experiments, and has full administrative control over VMs in assigned experiments.                       | E V   | E V   |   V    |   V    |   V   |   V    |
| VM Viewer         | Can see VM screenshots, use VM consoles, and mount VM disks.                                                             |   V   |   V   |        |        |       |   V    |
| Scorch Viewer     | Can see assigned experiments, their Scorch pipelines, component output, read-only Scorch terminals, and run files.       |   E   |   E   |        |        |       |        |
| Scorch Admin      | Everything Scorch Viewer can do, plus start and cancel Scorch runs, type into Scorch terminals, and see VMs.            |  E V  |  E V  |        |        |       |        |
| Builder           | Can design topologies and scenarios, and create and update experiments from them. Not scoped to experiments.             |  E C  |  E C  |  E C   |  E C   |       |        |
| Disabled          | Has no permissions. New self sign-up users get this role until an administrator assigns another.                         |       |       |        |        |       |        |

Key: E - experiment resource, V - VM resource, U - user resource, C - Topology, Scenario, and Experiment configs

### Builder, Scorch, and Tunneler Access

The Builder, Scorch, and Tunneler have their own permissions, described in
[Resources](#builder-scorch-and-tunneler). The built-in roles grant the
following.

| Role              | Builder                         | Scorch runs         | Scorch terminals | Tunneler download | Port forwards |
|-------------------|:--------------------------------|:--------------------|:----------------:|:-----------------:|:-------------:|
| Global Admin      | open, save, create, update      | view, start, cancel | type, exit       | yes               | yes           |
| Global Viewer     | open, save                      | view                |                  | yes               | list, connect |
| Experiment Admin  | open, save, create, update [^1] | view, start, cancel |                  | yes               | yes           |
| Experiment User   | open, save, create [^1]         | view, start, cancel |                  | yes               | yes           |
| Experiment Viewer | open, save                      | view                |                  | yes               | list, connect |
| VM Admin          |                                 | view, start, cancel |                  | yes               | yes           |
| VM Viewer         | open, save                      |                     |                  |                   |               |
| Scorch Viewer     | open, save                      | view                |                  |                   |               |
| Scorch Admin      |                                 | view, start, cancel | type, exit       |                   |               |
| Builder           | open, save, create, update      |                     |                  |                   |               |

[^1]: Creating experiments from the Builder also needs `experiments`
    `create`, and importing topologies from phēnix needs `configs` access.
    These roles don't have either by default, so in practice they can open
    the Builder and save files locally.

!!! warning "Scorch terminals are shells on the phēnix server"
    A Scorch terminal, such as the one a [`break`](scorch.md#break-component)
    component opens, is a shell running as the phēnix server process. In a
    typical Docker deployment, that is root in a privileged container that
    shares the host's process namespace. Anyone who can type into a Scorch
    terminal can control the phēnix server, and so can bypass all of phēnix's
    access control. For this reason, typing into and exiting Scorch terminals
    is a separate permission, [`scorch/terminals`](#resource-scorchterminals)
    `write`, from starting and canceling Scorch runs. By default, only Global
    Admin and Scorch Admin have it. Only assign it to users trusted with that
    access.

### Upgrading Existing Installs

The first time `phenix ui` starts after upgrading to a release with Builder,
Scorch, and Tunneler permissions, it keeps existing users working:

* Built-in roles, and the users assigned to them, get the access in the table
  above. Each role is updated once and then annotated with
  `phenix.rbac/service-permissions`, so an administrator can remove this
  access later without a restart adding it back.
* The Builder, Scorch Viewer, and Scorch Admin roles are created once. If an
  administrator deletes one, it is not created again.
* Custom roles are not changed.

### Built-In Role Definitions

phēnix creates these role configs when it first starts. They can be viewed
and edited like any other config, for example with
`phenix config get role/experiment-user`.

#### Global Admin (`global-admin`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: global-admin
spec:
  roleName: Global Admin
  policies:
  - resources:
    - "*"
    - "*/*"
    resourceNames:
    - "*"
    - "*/*"
    verbs:
    - "*"
```

#### Global Viewer (`global-viewer`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: global-viewer
spec:
  roleName: Global Viewer
  policies:
  - resources:
    - "*"
    - "*/*"
    resourceNames:
    - "*"
    - "*/*"
    verbs:
    - list
    - get
  - resources:
    - "vms/mount"
    resourceNames:
    - "*"
    - "*/*"
    verbs:
    - post
    - delete
```

#### Experiment Admin (`experiment-admin`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: experiment-admin
spec:
  roleName: Experiment Admin
  policies:
  - resources:
    - experiments
    - "experiments/*"
    verbs:
    - list
    - get
    - update
  - resources:
    - vms
    - "vms/*"
    verbs:
    - list
    - get
    - create
    - update
    - patch
    - delete
  - resources:
    - disks
    resourceNames:
    - "*"
    verbs:
    - list
  - resources:
    - "experiments/files"
    verbs:
    - create
  - resources:
    - hosts
    resourceNames:
    - "*"
    verbs:
    - list
  - resources:
    - builder
    verbs:
    - get
    - post
    - put
  # Start and cancel Scorch runs. Typing into Scorch terminals needs the
  # separate scorch/terminals write permission, which this role does not have.
  - resources:
    - scorch
    verbs:
    - get
    - post
    - delete
  - resources:
    - tunneler
    verbs:
    - get
```

#### Experiment User (`experiment-user`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: experiment-user
spec:
  roleName: Experiment User
  policies:
  - resources:
    - experiments
    - "experiments/*"
    verbs:
    - list
    - get
  - resources:
    - vms
    - "vms/*"
    verbs:
    - list
    - get
    - patch
  - resources:
    - "vms/redeploy"
    verbs:
    - update
  - resources:
    - "vms/captures"
    verbs:
    - create
    - delete
  - resources:
    - "vms/snapshots"
    verbs:
    - list
    - create
    - update
  # Tunneler port forwards. Keep this before the first policy with
  # resourceNames so assigning the role scopes it to the user's VMs.
  - resources:
    - "vms/forwards"
    verbs:
    - create
    - delete
  - resources:
    - "experiments/files"
    verbs:
    - create
  - resources:
    - hosts
    resourceNames:
    - "*"
    verbs:
    - list
  - resources:
    - builder
    verbs:
    - get
    - post
  # Start and cancel Scorch runs. Typing into Scorch terminals needs the
  # separate scorch/terminals write permission, which this role does not have.
  - resources:
    - scorch
    verbs:
    - get
    - post
    - delete
  - resources:
    - tunneler
    verbs:
    - get
```

#### Experiment Viewer (`experiment-viewer`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: experiment-viewer
spec:
  roleName: Experiment Viewer
  policies:
  - resources:
    - experiments
    - "experiments/*"
    - vms
    - "vms/*"
    verbs:
    - list
    - get
  - resources:
    - hosts
    resourceNames:
    - "*"
    verbs:
    - list
  - resources:
    - "vms/mount"
    verbs:
    - post
    - delete
  - resources:
    - builder
    verbs:
    - get
  - resources:
    - scorch
    verbs:
    - get
  - resources:
    - tunneler
    verbs:
    - get
```

#### VM Admin (`vm-admin`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: vm-admin
spec:
  roleName: VM Admin
  policies:
    - resources:
        - experiments
        - experiments/*
      verbs:
        - list
        - get
    - resources:
        - vms
        - vms/*
      verbs:
        - '*'
    # Start and cancel Scorch runs. Typing into Scorch terminals needs the
    # separate scorch/terminals write permission, which this role does not have.
    - resources:
        - scorch
      verbs:
        - get
        - post
        - delete
    - resources:
        - tunneler
      verbs:
        - get
```

#### VM Viewer (`vm-viewer`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: vm-viewer
spec:
  roleName: VM Viewer
  policies:
  - resources:
    - vms
    verbs:
    - list
  - resources:
    - "vms/screenshot"
    - "vms/vnc"
    verbs:
    - get
  - resources:
    - "vms/mount"
    verbs:
    - post
    - list
    - delete
    - get
  - resources:
    - builder
    verbs:
    - get
```

#### Scorch Viewer (`scorch-viewer`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: scorch-viewer
spec:
  roleName: Scorch Viewer
  # View Scorch pipelines, component output, read-only Scorch terminals, and
  # the run files Scorch writes to the experiment files directory. Assigning
  # the role scopes it to the user's experiments.
  policies:
  - resources:
    - experiments
    verbs:
    - list
    - get
  - resources:
    - "experiments/apps"
    - "experiments/files"
    verbs:
    - list
    - get
  - resources:
    - scorch
    verbs:
    - get
  - resources:
    - builder
    verbs:
    - get
```

#### Scorch Admin (`scorch-admin`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: scorch-admin
spec:
  roleName: Scorch Admin
  # Everything Scorch Viewer can do, plus start and cancel Scorch runs, type
  # into and exit Scorch terminals, and watch the VMs Scorch components act on.
  # Assigning the role scopes it to the user's experiments.
  policies:
  - resources:
    - experiments
    verbs:
    - list
    - get
  - resources:
    - "experiments/apps"
    - "experiments/files"
    verbs:
    - list
    - get
  - resources:
    - vms
    verbs:
    - list
    - get
  - resources:
    - "vms/screenshot"
    verbs:
    - get
  - resources:
    - scorch
    verbs:
    - "*"
  # Type into and exit Scorch terminals. This is separate from scorch because a
  # Scorch terminal, such as the one a break component opens, is a shell running
  # as the phenix server process, usually as root in a privileged container.
  # Anyone with this permission controls the phenix server and can bypass RBAC,
  # so only assign this role to users trusted with that access.
  - resources:
    - "scorch/terminals"
    verbs:
    - write
```

#### Builder (`builder`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: builder
spec:
  roleName: Builder
  # Design topologies and scenarios in the Builder or on the Configs page, and
  # create or update experiments from them. Every policy names its resources,
  # so, like the global roles, assigning this role does not scope it to
  # experiments. Config access excludes User and Role configs, which would let
  # a user grant themselves more access, and Image configs, whose build scripts
  # administrators run as root.
  policies:
  - resources:
    - builder
    resourceNames:
    - "*"
    verbs:
    - get
    - post
    - put
  - resources:
    - configs
    resourceNames:
    - "Topology/*"
    - "Scenario/*"
    - "Experiment/*"
    verbs:
    - list
    - get
    - create
    - update
  # create and update are what the Builder needs to create or update an
  # experiment; they do not allow starting, stopping, or deleting experiments.
  - resources:
    - experiments
    resourceNames:
    - "*"
    verbs:
    - list
    - get
    - create
    - update
  - resources:
    - disks
    resourceNames:
    - "*"
    verbs:
    - list
    - get
  - resources:
    - topologies
    - scenarios
    - applications
    - hosts
    - options
    resourceNames:
    - "*"
    verbs:
    - list
  - resources:
    - schemas
    resourceNames:
    - "*"
    verbs:
    - get
```

#### Disabled (`disabled`)

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: disabled
spec:
  roleName: Disabled
  policies: []
```

## Custom Roles

A custom role is a `Role` config. Create it from the `Configs` tab, or from a
file with `phenix config create <file>`, and then assign it to users on the
`Users` tab. `roleName` is the name shown in the UI; `metadata.name` can also
be used to refer to the role, for example in [`ui.users`](user-administration.md#from-configuration-uiusers).

This example lets a user run Scorch for their experiments without typing into
Scorch terminals:

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: scorch-operator
spec:
  roleName: Scorch Operator
  policies:
  - resources:
    - experiments
    - "experiments/apps"
    - "experiments/files"
    verbs:
    - list
    - get
  - resources:
    - scorch
    verbs:
    - get
    - post
    - delete
```

When writing a custom role:

* Put policies that should be limited to the user's experiments first, before
  any policy that sets `resourceNames`. See
  [Scoping a Role to Experiments](#scoping-a-role-to-experiments).
* `configs` names are `<Kind>/<name>`, so use patterns such as `Topology/*`,
  not `*`.
* Only grant these to roles meant to administer phēnix, because each lets a
  user gain more access:
    * `configs` `get`, `create`, or `update` on `User/*` or `Role/*`. User
      configs hold password hashes and API tokens.
    * `users` `create`, `users` `patch` on other users (which can create API
      tokens for them), and `users/roles` `patch`.
    * [`scorch/terminals`](#resource-scorchterminals) and
      [`miniconsole`](#resource-miniconsole).
* Existing users keep their copy of a role until it is reassigned; see
  [How Permissions Are Checked](#how-permissions-are-checked).

## Resources

Each table lists a resource's verbs, what they allow, and what the check is
named with (see [Named and unnamed checks](#how-permissions-are-checked)).
"—" means the check is not named, so `resourceNames` don't limit it.

### Experiments

#### Resource: `experiments`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list experiments; the list shows only experiments the user can access by name, and includes each experiment's VMs | experiment |
| get    | view an experiment and its VMs (VMs are filtered by [`vms`](#resource-vms) `list`), see live updates when an experiment is created, and read its [Scorch](#resource-scorch) data | experiment |
| create | create an experiment, including from the Builder | — |
| update | update an experiment's topology from the Builder (also needs [`builder`](#resource-builder) `put`) | experiment |
| patch  | change the VLAN aliases of a stopped experiment | experiment |
| delete | delete an experiment | experiment |

#### Resource: `experiments/apps`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | see an experiment's apps and which are running; the Scorch table needs this | experiment |

#### Resource: `experiments/captures`

| Verb | Allows | Named with |
|------|--------|------------|
| list | list an experiment's packet captures; the list shows only captures whose VM name (without the experiment) matches | experiment, then VM name |

#### Resource: `experiments/files`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list an experiment's files, including Scorch run output | experiment |
| get    | download an experiment file, or copy it into a mounted VM disk (also needs [`vms/mount`](#resource-vmsmount) `patch`) | experiment |
| create | upload experiment files through the file server | experiment |

#### Resource: `experiments/netflow`

| Verb   | Allows | Named with |
|--------|--------|------------|
| get    | check whether netflow is capturing, and stream netflow data | experiment |
| create | start a netflow capture | experiment |
| delete | stop a netflow capture | experiment |

#### Resource: `experiments/schedule`

| Verb   | Allows | Named with |
|--------|--------|------------|
| get    | view the experiment's VM-to-host schedule | experiment |
| create | schedule an experiment with a scheduling algorithm | experiment |

#### Resource: `experiments/start`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | start an experiment, and see live start progress and errors | experiment |

#### Resource: `experiments/stop`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | stop an experiment, and see live stop progress and errors | experiment |

#### Resource: `experiments/topology`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | view and search the experiment's topology graph | experiment |

#### Resource: `experiments/trigger`

| Verb   | Allows | Named with |
|--------|--------|------------|
| create | trigger the running stage of an experiment's apps, and see live app trigger updates; triggering the Scorch app also needs [`scorch`](#resource-scorch) `post` | experiment |
| delete | cancel triggered apps; canceling the Scorch app also needs [`scorch`](#resource-scorch) `delete` | experiment |

Starting and canceling Scorch runs from the Scorch table does not need
`experiments/trigger`.

#### Resource: `exp/captureSubnet`

| Verb   | Allows | Named with |
|--------|--------|------------|
| create | start or stop packet captures on every VM interface in a subnet | experiment |

This resource starts with `exp/`, so `experiments/*` does not grant it. Of the
built-in roles, only Global Admin has it.

### VMs

VM checks are named `<experiment>/<vm>`.

#### Resource: `vms`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list VMs, including their screenshots and live screenshot updates; the list shows only VMs the user can access by name | experiment/VM |
| list   | view an experiment's state of health | — |
| get    | view a VM | experiment/VM |
| patch  | change a VM's settings, such as CPUs, memory, disk, VLANs, tags, boot, host, and snapshot; bulk edits skip VMs the user can't access | experiment/VM |
| delete | kill a VM in a running experiment | experiment/VM |

#### Resource: `vms/captures`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list a VM's packet captures | experiment/VM |
| create | start a packet capture on a VM interface | experiment/VM |
| delete | stop a VM's packet captures | experiment/VM |

#### Resource: `vms/cdrom`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | insert an ISO into a VM's CD-ROM drive | experiment/VM |
| delete | eject a VM's CD-ROM | experiment/VM |

#### Resource: `vms/commit`

| Verb   | Allows | Named with |
|--------|--------|------------|
| create | create a new backing image from a VM | experiment/VM |

#### Resource: `vms/forwards`

Port forwards are used by the [tunneler](tunneler.md).

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list a VM's port forwards, including other users' | experiment/VM |
| get    | connect through any user's port forward | experiment/VM |
| create | create a port forward | experiment/VM |
| delete | delete your own port forward | experiment/VM |

#### Resource: `vms/memorySnapshot`

| Verb   | Allows | Named with |
|--------|--------|------------|
| create | create an ELF memory dump of a VM | experiment/VM |

#### Resource: `vms/mount`

Only available when phēnix is started with the `vm-mount` feature.

| Verb   | Allows | Named with |
|--------|--------|------------|
| post   | mount a VM's disk on the headnode | experiment/VM |
| delete | unmount it | experiment/VM |
| list   | browse the mounted files | experiment/VM |
| get    | download a mounted file | experiment/VM |
| patch  | upload a file into the mount, or copy an experiment file into it (also needs [`experiments/files`](#resource-experimentsfiles) `get`) | experiment/VM |

#### Resource: `vms/redeploy`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | redeploy a VM | experiment/VM |

#### Resource: `vms/reset`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | reset a VM's disk state | experiment/VM |

#### Resource: `vms/restart`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | restart a VM | experiment/VM |

#### Resource: `vms/screenshot`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | get a VM's screenshot | experiment/VM |

VM lists that include screenshots only need [`vms`](#resource-vms) `list`.

#### Resource: `vms/shutdown`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | shut down a VM | experiment/VM |

#### Resource: `vms/snapshots`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list a VM's snapshots | experiment/VM |
| create | snapshot a VM, and see live snapshot and restore progress | experiment/VM |
| update | restore a VM to a snapshot | experiment/VM |

#### Resource: `vms/start`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | start or resume a VM | experiment/VM |

#### Resource: `vms/stop`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | pause (stop) a VM | experiment/VM |

#### Resource: `vms/vnc`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | open a VM's VNC console | experiment/VM |

### Configs and Experiment Design

#### Resource: `configs`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list configs, and see live config changes | `<Kind>/<name>` |
| list   | load a Builder topology into the Builder (also needs [`builder`](#resource-builder) `get`) | `Topology/<name>` |
| get    | view or download a config | `<Kind>/<name>` |
| create | create a config from the `Configs` tab, the API, or the workflow API, or rename a config or change its kind | `<Kind>/<name>` of the new config |
| update | edit a config, including through the workflow API | `<Kind>/<name>` |
| delete | delete a config | `<Kind>/<name>` |

Config names are the config's kind and name, such as `Topology/foo`. A
resource name of `*` does not match any config; use `*/*` for all configs or a
kind pattern such as `Topology/*`.

!!! warning
    `configs` access to `User/*` or `Role/*` lets a user read password hashes
    and API tokens, or create and change users and roles, and so gain more
    access. The Builder role only covers `Topology/*`, `Scenario/*`, and
    `Experiment/*`.

#### Resource: `topologies`

| Verb | Allows | Named with |
|------|--------|------------|
| list | list topologies, including the Builder's topology list | topology |

#### Resource: `scenarios`

| Verb | Allows | Named with |
|------|--------|------------|
| list | list the scenarios for a topology | scenario |

#### Resource: `applications`

| Verb | Allows | Named with |
|------|--------|------------|
| list | list the available phēnix apps | app |

#### Resource: `schemas`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | get config schemas, used to validate configs in the UI | config kind, when getting one kind |

#### Resource: `options`

| Verb | Allows | Named with |
|------|--------|------------|
| list | get server defaults used when creating experiments, such as bridge mode | — |

#### Resource: `disks`

Disk checks are named with the image's file name, such as `ubuntu.qc2`.

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list disk images (VM, ISO, and container images) | file name |
| get    | download a disk image | file name |
| create | snapshot a disk into a new image | new file name |
| create | clone a disk | — |
| update | commit a disk into its backing image (checked on both), or rebase, resize, or rename a disk | file name |
| delete | delete a disk image | file name |
| upload | upload a disk image | — |

#### Resource: `workflow`

| Verb   | Allows | Named with |
|--------|--------|------------|
| create | apply a [git workflow](git-workflow.md), which creates, updates, and restarts experiments | — |

The workflow config API uses [`configs`](#resource-configs) permissions
instead.

### Builder, Scorch, and Tunneler

#### Resource: `builder`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | open the Builder, list and load Builder topologies (also needs [`configs`](#resource-configs) `list`), and save topology files locally | — |
| post | create an experiment and its topology from the Builder (also needs [`experiments`](#resource-experiments) `create`) | — |
| put  | update an experiment's topology from the Builder (also needs [`experiments`](#resource-experiments) `update` for the experiment, and `create` if it doesn't exist) | — |

#### Resource: `scorch`

Every Scorch route also needs [`experiments`](#resource-experiments) `get`
for the experiment. Triggering the Scorch app through
[`experiments/trigger`](#resource-experimentstrigger) needs `scorch` `post` or
`delete` instead.

| Verb   | Allows | Named with |
|--------|--------|------------|
| get    | view Scorch pipelines, component output, and read-only Scorch terminals, and see live Scorch updates | — |
| post   | start a Scorch run | — |
| delete | cancel a Scorch run | — |

#### Resource: `scorch/terminals`

| Verb  | Allows | Named with |
|-------|--------|------------|
| write | type into and exit Scorch terminals (also needs [`scorch`](#resource-scorch) `get`) | — |

!!! warning
    A Scorch terminal is a shell on the phēnix server, so this permission
    gives control of the phēnix server. It is separate from
    [`scorch`](#resource-scorch) so that users can run Scorch without it. See
    [Builder, Scorch, and Tunneler Access](#builder-scorch-and-tunneler-access).
    `*` and `scorch` don't match `scorch/terminals`, so `scorch` with verb
    `*` does not grant it; patterns such as `*/*` and `scorch/*` do.

#### Resource: `tunneler`

Only available when phēnix serves tunneler downloads, which it does when a
`downloads/tunneler` directory exists in its working directory, as in the
Docker image.

| Verb | Allows | Named with |
|------|--------|------------|
| get  | download the phēnix [tunneler](tunneler.md) (creating port forwards needs [`vms/forwards`](#resource-vmsforwards)) | — |

### Users and System

#### Resource: `users`

| Verb   | Allows | Named with |
|--------|--------|------------|
| list   | list users; without it, the list shows only the requester (with `get`) | username |
| get    | view a user | username |
| create | create users, and see live updates when users are created | — |
| patch  | change a user's name and password, and create API tokens for the user | username |
| delete | delete a user (users can't delete themselves) | username |

Users created from the `Users` tab or API get `get` and `patch` for
themselves, so they can change their own password and create their own API
tokens. Users created from [`ui.users`](user-administration.md#from-configuration-uiusers)
only get `get`.

#### Resource: `users/roles`

| Verb  | Allows | Named with |
|-------|--------|------------|
| patch | change a user's role and resource names (also needs [`users`](#resource-users) `patch`); without it, role changes are ignored | username |

#### Resource: `roles`

| Verb | Allows | Named with |
|------|--------|------------|
| list | list roles, such as for the role menu on the `Users` tab | — |

#### Resource: `hosts`

| Verb | Allows | Named with |
|------|--------|------------|
| list | list minimega cluster hosts | host |

#### Resource: `logs`

| Verb | Allows | Named with |
|------|--------|------------|
| get  | view phēnix and minimega logs on the `Logs` tab | — |

#### Resource: `miniconsole`

Only available when phēnix is started with `--minimega-console`.

| Verb | Allows | Named with |
|------|--------|------------|
| post | open and resize a minimega console | — |
| get  | attach to a minimega console, with read and write access | — |

!!! warning
    A minimega console can run any minimega command on the cluster, so this
    permission gives control of every experiment and cluster host. Treat it
    like [`scorch/terminals`](#resource-scorchterminals).

#### Resource: `settings`

| Verb   | Allows | Named with |
|--------|--------|------------|
| update | change phēnix settings on the `Settings` tab; viewing settings needs no permission | — |
