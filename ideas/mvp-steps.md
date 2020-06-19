# MVP user journey
Including an overview of differences between local and m3o env implementations where applicable.

The MVP supports the following commands:

- micro new
- micro server
- micro run <local or public github>
- micro status
- micro logs 
- micro call
- micro config
- micro update
- micro kill 
- micro env (set / add)
- micro login


Assumptions:
- the user is already familiar with [basic Micro commands](https://dev.m3o.com/getting-started).
- micro cli is installed on the users machine

Note `proxy.micro.mu` should likely become `proxy.m3o.com` or similar (leaving proxy out is likely more elegant).

```sh
$ micro env
  local                             none
* server                            127.0.0.1:8081
  platform                          proxy.micro.mu
```

## New User Journey

We assume they have followed the tutorial steps which should provide them with `micro` which already has knowledge of `platform` env baked in.

### 1. `micro env set platform`

With this step the user switches to using our platform as opposed to her `micro server`. But at this point she does not have an account, so  commands should return "Not logged in" or a similar error message:
```
$ micro run github.com/micro/examples/helloworld
Not logged in to env 'server' hosted at 'proxy.micro.mu'.
Please see `micro login help`
```

### 2. Introduce `micro login --new`

- In this step we will create an account for the user.
- We ask for their email address, when they submit it we
  1. Create an account in `unverified` status
  1. Send an email with details on how to pay for their subscription - likely this is a link to a single page which allows them to pay via stripe
- When they pay we'll ask them to submit their payment confirmation code which will put their status in `complete`. Either we pause the `micro login --new` call here and expect that the user keeps the terminal window running while they do all the payment stuff OR we have return from that command and have a second command `micro login --confirm|verify` which takes the confirmation code (this is a little bit nicer because it means the user can carry on where they left off in the event of an issue) 
- Only once in `complete` status will we create a namespace for them on the backend and return the generated namespace name and that should be written to the local Micro CLI user config file (found at path `~/.micro` by default currently). This can be in any format in that file, it is unimportant and can be changed later. This value will be embedded on each call to the platform in the `Micro-Namespace` header.
- If an account is `unverified` they shouldn't be able to do anything on m3o

Note: For the MVP we decided to hide the concept of namespaces from the users to leave us more time to work out the details.

**Things to work out**:
- ~~"Right now `micro login` uses the auth of the namespace you're currently in. If we're changing this to always talk to the platform it'll mean the users auth account isn't valid when making a call via the CLI."~~ EDIT: We'll give them a single global account for m3o that they login with and has access to do "platform" things. Since we only give them one namespace right now it shouldn't matter.

```
$ micro run github.com/micro/examples/helloworld
Payment is required. Please see `micro account help` for payment information. 
```

### 3. The `platform` environment is ready to be used

```
$ micro env
  local                             none
  server                            127.0.0.1:8081
* platform                          proxy.micro.mu
```

## Existing User Journey
The existing user journey is simple and requires little additional work
```
micro env set platform
micro login
```

## Overview of major differences between local/default implementations and ones used by m3o

Focusing on initial commands during the MVP user journey.

### `micro run [url]`

```
micro run github.com/micro/examples/helloworld
```

#### Default implementation

Checks out git repositories to a local folder (eg. `$TMPDIR/github-com-micro-examples`) and runs the code from there.

#### K8s implementation

A generic Go image downloads checks out the git repositories and runs the code from there. **We need to modify how `micro server` is ran in live, see [discussion here](https://github.com/micro/development/pull/221)**

For MVP we will not do anything with private repos since that needs ability to pass secrets etc. The workaround is for the user to clone the private repo to their local machine and use the `micro run [local path]` command below.

### `micro run [local path]`

#### Default implementation

The local code gets gzipped and uploaded to the server through the `server.Upload` endpoint, which uncompresses the zip and writes the folder into `$TMPDIR/uploads/[service name]` then does a `go run [folder path]`.

#### K8s implementation

**This implementation is nonexistent, ie. entirely broken.**

What should happen is the local code gets gzipped and uploaded to the server and saved to store or written straight to the store to begin with. After this we need to get the code into the pod/container. This step is to be defined and likely involves architecture work.

### `micro logs -f [runtime name]`

#### Default implementation

The logs come from the server's file system and get streamed to the CLI command by the runtime service logs endpoint. (Note: multi node setup might be borked, does not matter for MVP.). The logs, coming from the filesystem are unstructured and it's not trivial to seek exact lines (https://github.com/micro/go-micro/blob/master/runtime/default.go#L432)

#### K8s implementation

The logs come from K8s in a structured and easy to query way and then get streamed to the client by the runtime service logs endpoint. **This is broken at the moment and needs to be fixed for MVP.**