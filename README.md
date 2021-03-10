# hellyna/docker-socket-proxy

This fork tries to provide an even more granular approach to the [original](tecnativa/docker-socket-proxy) and [fluencelab's fork](https://github.com/fluencelabs/docker-socket-proxy).

## What?

This is a security-enhanced proxy for the Docker Socket.

## Why?

Giving access to your Docker socket could mean giving root access to your host,
or even to your whole swarm, but some services require hooking into that socket
to react to events, etc. Using this proxy lets you block anything you consider
those services should not do.

## How?

We use the official [Alpine][]-based [HAProxy][] image with a small
configuration file.

It blocks access to the Docker socket API according to the environment
variables you set. It returns a `HTTP 403 Forbidden` status for those dangerous
requests that should never happen.

## Security recommendations

- Never expose this container's port to a public network. Only to a Docker
  networks where only reside the proxy itself and the service that uses it.
- Revoke access to any API section that you consider your service should not
  need.
- This image does not include TLS support, just plain HTTP proxy to the host
  Docker Unix socket (which is not TLS protected even if you configured your
  host for TLS protection). This is by design because you are supposed to
  restrict access to it through Docker's built-in firewall.
- [Read the docs](#suppported-api-versions) for the API version you are using,
  and **know what you are doing**.

## Usage

1.  Run the API proxy

        $ docker container run \
            -d \
            --name dockerproxy \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -p 127.0.0.1:2375:2375 \
            tecnativa/docker-socket-proxy

    The `--privileged` flag may be required here if you use SELinux or AppArmor.

2.  Connect your local docker client to that socket:

        $ export DOCKER_HOST=tcp://localhost

3.  You can see the docker version:

        $ docker version
        Client:
         Version:      17.03.1-ce
         API version:  1.27
         Go version:   go1.7.5
         Git commit:   c6d412e
         Built:        Mon Mar 27 17:14:43 2017
         OS/Arch:      linux/amd64

        Server:
         Version:      17.03.1-ce
         API version:  1.27 (minimum version 1.12)
         Go version:   go1.7.5
         Git commit:   c6d412e
         Built:        Mon Mar 27 17:14:43 2017
         OS/Arch:      linux/amd64
         Experimental: false

4.  You cannot see running containers:

        $ docker container ls
        Error response from daemon: <html><body><h1>403 Forbidden</h1>
        Request forbidden by administrative rules.
        </body></html>

The same will happen to any containers that use this proxy's `2375` port to
access the Docker socket API.

## Grant or revoke access to certain API sections

You grant and revoke access to certain features of the Docker API through
environment variables.

Normally the variables match the URL prefix (i.e. `AUTH` blocks access to
`/auth/*` parts of the API, etc.).

Possible values for these variables:

- `0` to **revoke** access.
- `1` to **grant** access.

### Access granted by default

These API sections are mostly harmless and almost required for any service that
uses the API, so they are granted by default.

- `HEAD_PING`
- `GET_PING`
- `GET_EVENTS`
- `GET_VERSION`

### Access revoked by default

#### Security-critical

These API sections are considered security-critical, and thus access is revoked
by default. Maximum caution when enabling these.

- `AUTH`
- `SECRETS`
- `POST_ALL`: Enables all POST operations
- `DELETE_ALL`: Enables all DELETE operations

#### Not always needed

You will possibly need to grant access to some of these API sections, which
can expose some information that your service does not need.

A good way is to run your application that you want to filter with with `POST_ALL`
and `DELETE_ALL` set to `1` connected to this proxy, then check the logs after to know
which are the ones you need to allow.

| HEAD | GET            | POST                  | DELETE              |
|:-----|:---------------|:----------------------|:--------------------|
| `HEAD_PING` | `GET_BUILD`        |  `POST_CONTAINERS_PRUNE`   | `NETWORKS_DELETE`   |
|             | `GET_COMMIT`       |  `POST_CONTAINERS_CREATE`  | `CONTAINERS_DELETE` |
|             | `GET_CONFIGS`      |  `POST_CONTAINERS_RESIZE`  | `IMAGES_DELETE`     |
|             | `GET_CONTAINERS`   |  `POST_CONTAINERS_START`   | `VOLUMES_DELETE`    |
|             | `GET_DISTRIBUTION` |  `POST_CONTAINERS_STOP`    |                     |
|             | `GET_EXEC`         |  `POST_CONTAINERS_RESTART` |                     |
|             | `GET_IMAGES`       |  `POST_CONTAINERS_KILL`    |                     |
|             | `GET_INFO`         |  `POST_CONTAINERS_UPDATE`  |                     |
|             | `GET_NETWORKS`     |  `POST_CONTAINERS_RENAME`  |                     |
|             | `GET_NODES`        |  `POST_CONTAINERS_PAUSE`   |                     |
|             | `GET_PLUGINS`      |  `POST_CONTAINERS_UNPAUSE` |                     |
|             | `GET_SERVICES`     |  `POST_CONTAINERS_ATTACH`  |                     |
|             | `GET_SESSION`      |  `POST_CONTAINERS_WAIT`    |                     |
|             | `GET_SWARM`        |  `POST_CONTAINERS_EXEC`    |                     |
|             | `GET_SYSTEM`       |  `POST_VOLUMES_CREATE`     |                     |
|             | `GET_TASKS`        |  `POST_VOLUMES_PRUNE`      |                     |
|             | `GET_VOLUMES`      |  `POST_NETWORKS_CREATE`    |                     |
|             |                    |  `POST_NETWORKS_PRUNE`     |                     |
|             |                    |  `POST_NETWORKS_CONNECT`   |                     |
|             |                    |  `POST_NETWORKS_DISCONNECT`|                     |
|             |                    |  `POST_IMAGES_CREATE`      |                     |
|             |                    |  `POST_IMAGES_PRUNE`       |                     |

## Logging

You can set the logging level or severity level of the messages to be logged with the
 environment variable `LOG_LEVEL`. Defaul value is info. Possible values are: debug,
 info, notice, warning, err, crit, alert and emerg.

## Supported API versions

- [1.27](https://docs.docker.com/engine/api/v1.27/)
- [1.28](https://docs.docker.com/engine/api/v1.28/)
- [1.29](https://docs.docker.com/engine/api/v1.29/)
- [1.30](https://docs.docker.com/engine/api/v1.30/)
- [1.37](https://docs.docker.com/engine/api/v1.37/)

Please open any PR if you find an error or require support to a specific version of API.

Useful link: [API version history](https://docs.docker.com/engine/api/version-history/)
