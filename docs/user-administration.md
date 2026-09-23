# User Authn/Authz in phenix

`phenix` provides three separate modes of user authentication (authn) and
authorization (authz).

* disabled
* enabled
* proxy

## `disabled` Mode

When in `disabled` mode, no user authentication or authorization occurs. Users
do not have to authenticate, and all actions are allowed.

To use `disabled` mode, the UI should be built with `VUE_APP_AUTH=disabled` (if
Docker is being used to build the UI, use Docker build arg
`PHENIX_WEB_AUTH=disabled`) and the UI server should be started without the
`-k/--jwt-signing-key` option set.

## `enabled` Mode

When in `enabled` mode, user authentication and authorization occurs within
phenix directly. Users have to authenticate to the phenix UI, and certain
actions are prohibited based on the role assigned to the user.

To use `enabled` mode, the UI should be built with `VUE_APP_AUTH=enabled` (if
Docker is being used to build the UI, use Docker build arg
`PHENIX_WEB_AUTH=enabled`) and the UI server should be started with the
`-k/--jwt-signing-key` (and optionally the `--jwt-lifetime`) option set.

## `proxy` Mode

When in `proxy` mode, user authentication is expected to occur in a reverse
proxy that sits in front of phenix but user authorization still occurs within
phenix directly. Users authenticate to the proxy, and certain actions are
prohibited based on the role assigned to the user.

To use `proxy` mode, the UI should be built with `VUE_APP_AUTH=proxy` (if Docker
is being used to build the UI, use Docker build arg `PHENIX_WEB_AUTH=proxy`) and
the UI server should be started with the `-k/--jwt-signing-key` and
`--proxy-auth-header` (and optionally the `--jwt-lifetime`) options set.

In addition, the reverse proxy should add a header to requests being proxied
that contains the username of the authenticated user, with the name of the
header matching what `--proxy-auth-header` is set to (for example,
`--proxy-auth-header=X-phenix-user`).

If a user is able to authenticate to the proxy but is not yet a user in phenix,
they will be added as a phenix user automatically and assigned the `Disabled`
role that will deny all actions until an admin user can assign them a different
role.

## Create a new user

There are three primary ways to create new users.

1. Choose the `Create Account` link off the login page and complete all fields
   in the `Create a New Account` dialogue. This will initiate a message to an
   administrator's account who can then activate the account, setting the
   role(s) and resource name(s).

    ![screenshot](images/login_create.png){: width=400 .center}

    ![screenshot](images/create_new_account.png){: width=400 .center}

2. From the `Users` tab, click the `+` button to create a new user. Here the
   administrator will add the [role(s) and resource
   name(s)](#user-administration).

    ![screenshot](images/create_a_new_user.png){: width=400 .center}

3. Create a YAML or JSON file at `/etc/phenix/users.[yml|json]` with the
   following structure. When the `phenix` UI starts, it looks for this file and
   adds any users present in the file that are not already present in `phenix`.
   For users in the file that already exist, `phenix` ensures the user role
   matches what's in the file and updates it as necessary. This file is also
   automatically watched, so any users added to the file while `phenix` is
   running will automatically be added to `phenix`.

```yaml
ui:
  users:
    - <username>:<password>:<role name>
    - ...
```

!!! note
    A user must be assigned a role that exists. Creating a user with an
    unknown role from the `Users` tab or the API returns an error, and users
    in the users file whose role doesn't exist are skipped and logged, so
    phēnix never stores a user without a role. Creating a user whose name is
    already taken returns `409 Conflict`.

## Login

The login page is self-descriptive. Using the `Remember me` checkbox will set a
token to local storage so that you can remove the requirement to enter a
`Username` and `Password` each time the page or site is reloaded.

If an administrator starts the UI server with the following command,
authentication is enabled:

```shell
phenix ui -k <some_string>
```

Without the `-k` (or `--jwt-signing-key`), authentication is disabled.

## Generating User Authentication Tokens

From the `Users` tab, click the key icon next to the given user's name. A dialog
box will pop up where you can enter in a description for the token to be created
and an expiration date. This expiration date should be entered in Golang time
duration. For example, `4320h` is valid and represents 4320 hours or about 6
months. After clicking `Create Token` a token should appear with the expiration
date.

This token can be used to authenticate when using the Phenix API. Specifically, you would include the following as a header in HTTP requests.

```http
X-phenix-auth-token: ******
```

## User Administration

### Updating Users

An administrator is able to click on the username on the table in the Users tab
to update a user. They can update `First Name` or `Last Name`, `Role`,
`Experiment Names`, and `Resource Name(s)`.

### Scoping a Role to Experiments

When a role is assigned to a user, the space-separated `Resource Name(s)` are
copied into the role's policies that don't already name their resources, in
order, stopping at the first policy that does. This is how roles such as
Experiment User and Scorch Viewer are limited to the user's experiments.

* Experiment checks use the experiment name, such as `exp-a`.
* VM checks, including port forwards, use `<experiment>/<vm>`, so enter both
  the experiment and a VM pattern, for example `exp-a exp-a/*`.
* Roles whose policies all name their resources, such as Global Admin, Global
  Viewer, and Builder, ignore these names and are not scoped to experiments.
* Builder, Scorch, and Tunneler access (`builder`, `scorch`,
  `scorch/terminals`, and `tunneler`) is checked without a name, so scoping
  does not limit it. Scorch data is still limited to experiments the user can
  read.

### Roles

`Global Admin` is the administrator level account and has access to all
capabilities, to include user management. Global Admins also have access to all
resources. The following table provides a high-level overview of all the
available roles and their access rights.

| Role              | Limits                                                                                                                   | List  |  Get  | Create | Update | Patch | Delete |
|-------------------|:-------------------------------------------------------------------------------------------------------------------------|:-----:|:-----:|:------:|:------:|:-----:|:------:|
| Global Admin      | Can see and control absolutely anything/everything.                                                                      | E V U | E V U | E V U  | E V U  | E V U | E V U  |
| Global Viewer     | Can see absolutely anything/everything, but cannot make any changes.                                                     | E V U | E V U |        |        |       |        |
| Experiment Admin  | Can see and control anything/everything for assigned experiments, including VMs, but cannot create new experiments.      | E V   | E V   |   V    | E V    |   V   |   V    |
| Experiment User   | Can see assigned experiments, and can control VMs within assigned experiments, but cannot modify experiments themselves. | E V   | E V   |        |        |   V   |        |
| Experiment Viewer | Can see assigned experiments and VMs within assigned experiments, but cannot modify or control experiments or VMs.       | E V   | E V   |        |        |       |        |
| VM Admin          | Can see assigned experiments, and has full administrative control over VMs in assigned experiments.                       | E V   | E V   |   V    |   V    |   V   |   V    |
| VM Viewer         | Can only see VM screenshots and access VM VNC, nothing else.                                                             |   V   |       |        |        |       |        |
| Scorch Viewer     | Can see assigned experiments, their Scorch pipelines, component output, read-only Scorch terminals, and run files.       |   E   |   E   |        |        |       |        |
| Scorch Admin      | Everything Scorch Viewer can do, plus start and cancel Scorch runs, type into Scorch terminals, and see VMs.            |  E V  |  E V  |        |        |       |        |
| Builder           | Can design topologies and scenarios, and create and update experiments from them. Not scoped to experiments.             |  E C  |  E C  |  E C   |  E C   |       |        |

Key: E - experiment resource, V - VM resource, U - user resource, C - Topology, Scenario, and Experiment configs

#### Builder, Scorch, and Tunneler Access

The Builder, Scorch, and Tunneler have their own permissions, described in
[Resources](#resources). The built-in roles grant the following. Custom roles
must add these permissions explicitly.

| Role              | Builder                        | Scorch runs                  | Scorch terminals | Tunneler download | Port forwards |
|-------------------|:-------------------------------|:-----------------------------|:----------------:|:-----------------:|:-------------:|
| Global Admin      | open, save, create, update     | view, start, cancel          | type, exit       | yes               | yes           |
| Global Viewer     | open, save                     | view                         |                  | yes               | view          |
| Experiment Admin  | open, save, create, update [^1] | view, start, cancel          |                  | yes               | yes           |
| Experiment User   | open, save, create [^1]        | view, start, cancel          |                  | yes               | yes           |
| Experiment Viewer | open, save                     | view                         |                  | yes               | view          |
| VM Admin          |                                | view, start, cancel          |                  | yes               | yes           |
| VM Viewer         | open, save                     |                              |                  |                   |               |
| Scorch Viewer     | open, save                     | view                         |                  |                   |               |
| Scorch Admin      |                                | view, start, cancel          | type, exit       |                   |               |
| Builder           | open, save, create, update     |                              |                  |                   |               |

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

#### Upgrading Existing Installs

The first time phēnix starts after upgrading to a release with Builder,
Scorch, and Tunneler permissions, it keeps existing users working:

* Built-in roles, and the users assigned to them, get the access in the table
  above. Each role is updated once and then annotated with
  `phenix.rbac/service-permissions`, so an administrator can remove this
  access later without a restart adding it back.
* The Builder, Scorch Viewer, and Scorch Admin roles are created once. If an
  administrator deletes one, it is not created again.
* Custom roles are not changed.

### Resources

#### Resource: `experiments`

|      |      |
|------|------|
| Verb | list |
| Desc | get a list of all experiments |
| Exp. Scoped | yes (list is filtered to only include experiments in scope) |
| Res. Scoped | no |

|      |      |
|------|------|
| Verb | get
| Desc | get a specific experiment
| Exp. Scoped | yes
| Res. Scoped | no

|      |      |
|------|------|
| Verb | create
| Desc | create a new experiment
| Exp. Scoped | no
| Res. Scoped | no

|      |      |
|------|------|
| Verb | update
| Desc | update an experiment's topology from the Builder (also see [`builder`](#resource-builder))
| Exp. Scoped | yes
| Res. Scoped | no

|      |      |
|------|------|
| Verb | delete
| Desc | delete a specific experiment
| Exp. Scoped | yes
| Res. Scoped | no

#### Resource: `experiments/start`

|      |      |
|------|------|
| Verb | update
| Desc | start an experiment
| Exp. Scoped | yes
| Res. Scoped | no

#### Resource: `experiments/stop`

|      |      |
|------|------|
| Verb | update
| Desc | stop an experiment
| Exp. Scoped | yes
| Res. Scoped | no

#### Resource: `experiments/schedule`

|      |      |
|------|------|
| Verb | get
| Desc | get current schedule for an experiment
| Exp. Scoped | yes
| Res. Scoped | no

|      |      |
|------|------|
| Verb | create
| Desc | schedule an experiment using schedule algorithm
| Exp. Scoped | yes
| Res. Scoped | no

#### Resource: `experiments/trigger`

|      |      |
|------|------|
| Verb | create
| Desc | trigger the running stage of an experiment
| Exp. Scoped | yes
| Res. Scoped | no

|      |      |
|------|------|
| Verb | delete
| Desc | cancel triggered apps for an experiment
| Exp. Scoped | yes
| Res. Scoped | no

Triggering or canceling the Scorch app this way also needs
[`scorch`](#resource-scorch) `post` or `delete`. Starting and canceling Scorch
runs from the Scorch table does not need `experiments/trigger`.

#### Resource: `experiments/captures`

|      |      |
|------|------|
| Verb | list
| Desc | get list of packet captures for an experiment
| Exp. Scoped | yes (list is filtered to only include experiments in scope)
| Res. Scoped | yes (list is filtered to only include VMs in scope)

#### Resource: `experiments/files`

|      |      |
|------|------|
| Verb | list
| Desc | get list of files for an experiment
| Exp. Scoped | yes (list is filtered to only include experiments in scope)
| Res. Scoped | no

|      |      |
|------|------|
| Verb | get
| Desc | get specific experiment file
| Exp. Scoped | yes
| Res. Scoped | no

#### Resource: `vms`

|      |      |
|------|------|
| Verb | list
| Desc | get list of VMs for an experiment
| Exp. Scoped | yes (list is filtered to only include experiments in scope)
| Res. Scoped | yes (list is filtered to only include VMs in scope)

|      |      |
|------|------|
| Verb | get
| Desc | get a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | patch
| Desc | update a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | delete
| Desc | delete a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/start`

|      |      |
|------|------|
| Verb | update
| Desc | start a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/stop`

|      |      |
|------|------|
| Verb | update
| Desc | stop a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/redeploy`

|      |      |
|------|------|
| Verb | update
| Desc | redeploy a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/screenshot`

|      |      |
|------|------|
| Verb | get
| Desc | get screenshot for a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/vnc`

|      |      |
|------|------|
| Verb | get
| Desc | get VNC address for a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/captures`

|      |      |
|------|------|
| Verb | list
| Desc | get list of packet captures for a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | create
| Desc | start a packet capture on a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | delete
| Desc | stop all packet captures on a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/snapshots`

|      |      |
|------|------|
| Verb | list
| Desc | get list of snapshots for a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | create
| Desc | create a snapshot of a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | update
| Desc | restore a specific experiment VM to a previous snapshot
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/commit`

|      |      |
|------|------|
| Verb | create
| Desc | create a new backing image from a specific experiment VM
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `applications`

|      |      |
|------|------|
| Verb | list
| Desc | get list of user applications
| Exp. Scoped | no
| Res. Scoped | yes (list is filtered to only include applications in scope)

#### Resource: `topologies`

|      |      |
|------|------|
| Verb | list
| Desc | get list of available topologies
| Exp. Scoped | no
| Res. Scoped | yes (list is filtered to only include topologies in scope)

#### Resource: `disks`

|      |      |
|------|------|
| Verb | list
| Desc | get list of available backing images
| Exp. Scoped | no
| Res. Scoped | yes (list is filtered to only include backing images in scope)

|      |      |
|------|------|
| Verb | get
| Desc | download a specific backing image
| Exp. Scoped | no
| Res. Scoped | yes

#### Resource: `hosts`

|      |      |
|------|------|
| Verb | list
| Desc | get list of minimega cluster hosts
| Exp. Scoped | no
| Res. Scoped | yes (list is filtered to only include hosts in scope)

#### Resource: `users`

|      |      |
|------|------|
| Verb | list
| Desc | get list of users
| Exp. Scoped | no
| Res. Scoped | yes (list is filtered to only include users in scope)

|      |      |
|------|------|
| Verb | get
| Desc | get a specific user
| Exp. Scoped | no
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | create
| Desc | create a new user
| Exp. Scoped | no
| Res. Scoped | no

|      |      |
|------|------|
| Verb | patch
| Desc | update an existing user
| Exp. Scoped | no
| Res. Scoped | yes

|      |      |
|------|------|
| Verb | delete
| Desc | delete an existing user
| Exp. Scoped | no
| Res. Scoped | yes

#### Resource: `users/roles`

|      |      |
|------|------|
| Verb | patch
| Desc | update user role assignments
| Exp. Scoped | no
| Res. Scoped | yes

#### Resource: `configs`

|      |      |
|------|------|
| Verb | list, get, create, update, delete
| Desc | manage store configurations (topologies, scenarios, etc.)
| Exp. Scoped | no
| Res. Scoped | yes (checked against `Kind/name`, such as `Topology/foo`)

Config permissions are checked against the config's kind and name, such as
`Topology/foo`. A resource name of `*` does not match any config; use `*/*`
for all configs or a kind pattern such as `Topology/*`. Creating a config is
checked against the new config's `Kind/name`, and renaming a config or
changing its kind needs `create` for the new name.

!!! warning
    `configs` access to `User/*` or `Role/*` lets a user create or change
    users and roles, and so grant themselves more access. The Builder role
    only covers `Topology/*`, `Scenario/*`, and `Experiment/*`.

#### Resource: `schemas`

|      |      |
|------|------|
| Verb | get
| Desc | get config schemas, used to validate configs in the UI
| Exp. Scoped | no
| Res. Scoped | yes (by config kind)

#### Resource: `scenarios`

|      |      |
|------|------|
| Verb | list
| Desc | get list of scenarios for a topology
| Exp. Scoped | no
| Res. Scoped | yes (list is filtered to only include scenarios in scope)

#### Resource: `options`

|      |      |
|------|------|
| Verb | list
| Desc | get server-side defaults used when creating experiments
| Exp. Scoped | no
| Res. Scoped | no

#### Resource: `builder`

|      |      |
|------|------|
| Verb | get
| Desc | open the Builder, list Builder topologies, and save topology files locally
| Exp. Scoped | no
| Res. Scoped | no

|      |      |
|------|------|
| Verb | post
| Desc | create an experiment from the Builder (also needs `experiments` `create`)
| Exp. Scoped | no
| Res. Scoped | no

|      |      |
|------|------|
| Verb | put
| Desc | update an experiment from the Builder (also needs `experiments` `update` for the experiment, and `create` if it doesn't exist)
| Exp. Scoped | no
| Res. Scoped | no

#### Resource: `scorch`

|      |      |
|------|------|
| Verb | get
| Desc | view Scorch pipelines, component output, and read-only Scorch terminals, and receive Scorch updates
| Exp. Scoped | yes (also needs `experiments` `get` for the experiment)
| Res. Scoped | no

|      |      |
|------|------|
| Verb | post
| Desc | start a Scorch run
| Exp. Scoped | yes (also needs `experiments` `get` for the experiment)
| Res. Scoped | no

|      |      |
|------|------|
| Verb | delete
| Desc | cancel a Scorch run
| Exp. Scoped | yes (also needs `experiments` `get` for the experiment)
| Res. Scoped | no

#### Resource: `scorch/terminals`

|      |      |
|------|------|
| Verb | write
| Desc | type into and exit Scorch terminals
| Exp. Scoped | yes (also needs `scorch` `get` and `experiments` `get` for the experiment)
| Res. Scoped | no

!!! warning
    A Scorch terminal is a shell on the phēnix server, so this permission
    gives control of the phēnix server. It is separate from
    [`scorch`](#resource-scorch) so that users can run Scorch without it. See
    [Builder, Scorch, and Tunneler Access](#builder-scorch-and-tunneler-access).
    A resource pattern of `scorch` does not match `scorch/terminals`, so
    `scorch` with verb `*` does not grant it.

#### Resource: `tunneler`

|      |      |
|------|------|
| Verb | get
| Desc | download the phēnix tunneler (also see [`vms/forwards`](#resource-vmsforwards))
| Exp. Scoped | no
| Res. Scoped | no

#### Resource: `settings`

|      |      |
|------|------|
| Verb | update
| Desc | update phēnix system settings
| Exp. Scoped | no
| Res. Scoped | no

#### Resource: `vms/mount`

|      |      |
|------|------|
| Verb | list, get, post, patch, delete
| Desc | manage VM filesystem mounts on headnode
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/forwards`

|      |      |
|------|------|
| Verb | list, get, create, delete
| Desc | manage port forwarding rules for experiment VMs, used by the [tunneler](tunneler.md)
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/cdrom`

|      |      |
|------|------|
| Verb | update, delete
| Desc | mount or eject CD-ROM ISO images on experiment VMs
| Exp. Scoped | yes
| Res. Scoped | yes

#### Resource: `vms/memorySnapshot`

|      |      |
|------|------|
| Verb | create
| Desc | create ELF memory dumps of experiment VMs
| Exp. Scoped | yes
| Res. Scoped | yes

### Built-In Roles

The following default roles are defined in phēnix as YAML resource specifications:

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

View Scorch for assigned experiments without controlling runs or typing into terminals.

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

Run Scorch for assigned experiments, including typing into Scorch terminals. See the warning in [Builder, Scorch, and Tunneler Access](#builder-scorch-and-tunneler-access) before assigning this role.

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

Design topologies and scenarios and create or update experiments from them. It is not scoped to experiments and cannot read or change User, Role, or Image configs.

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

Denies everything. New self sign-up and proxy users get this role until an administrator assigns another.

```yaml
apiVersion: phenix.sandia.gov/v1
kind: Role
metadata:
  name: disabled
spec:
  roleName: Disabled
  policies: []
```
