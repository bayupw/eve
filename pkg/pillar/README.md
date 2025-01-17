# EVE microservices for device and app instance management

For information about EVE see <https://www.lfedge.org/projects/eve/>

The onboarding of devices is done by `scripts/device-steps.sh`, and after onboarding that script proceeds to start the agents implementing the various microservices. That script runs on each boot. Once the agents are running they are operated from the controller using the API specified in <https://github.com/lf-edge/eve/api>

The agents are:

- ledmanager - make LEDs light/blink for feedback to the person installing the hardware
- [nim](./docs/nim.md) - network interface manager (ensures that there is connectivity to the controller); manages [radio-silence](./docs/radio-silence.md)
- waitforaddr - merely waiting for a few minutes max for IP address(es) for a more orderly boot in the normal case
- [vaultmgr](./docs/vaultmgr.md) - responsible of creation and operations over, encrypted vault(s) for mutable application images and device configuration data
- nodeagent - montior the device health, while node is in baseos upgrade or normal operation mode. Also orchestrates baseos installation and upgrade validation by interacting with baseosmgr and zedagent.
- zedagent - communicate using the device API to the controller to retrieve configuration and send status and metrics
- loguploader - send gzip logs to the controller for debugging of these agents
- baseosmgr - handle updates of the base OS (hypervisors plus all of the services which make up EVE) using dual partitions for fallback
- [volumemgr](./docs/volumemgr.md) - create volumes based on downloads or from scratch
- downloader - download objects like images and certificates
- verifier - verify cryptographic checksums and signatures on downloaded objects
- [zedmanager](./docs/zedmanager.md) - drive the application instance lifecycle
- [zedrouter](./docs/zedrouter.md) - drive the lifecycle for the connectivity for the instances. Includes services like DHCP, DNS, and Access Control Lists. Provides different connectivity like local, switch, cloud, and mesh networks
- [domainmgr](./docs/domainmgr.md) - interface with the hypervisor to start and stop application images. Includes performing device assignment
- identitymgr - used when mesh networks desire locally created key pairs for the cryptographic application instance identities
- zfsmanager - handle zfs devices managed by mdev
- [tpmmgr](./docs/tpmmgr.md) - manages the Trusted Platform Module

In addition there are debugging tools like:

- diag - prints the state of the connectivity on the console each time there is a change
- ipcmonitor - subscribes to the agents/collections passed between the different microservices

In order to conserve filesystem space, all of the agents above are built into a single executable (zedbox) and are differentiated based on the symbolic link (very similar to how BusyBox does it with traditional UNIX utilities).

## Building

Generally, the build is done inside a Docker container to ensure environment consistency. This container is `FROM golang:${GOVER}-alpine`. When running via docker-for-mac or docker-for-windows, you can run in a container that is derived from the library `golang` image. However, when building on Linux, the output artifacts will have the wrong ownership. Thus, we build a simple special image that contains and writes to the correct user.

All output binaries and links are in `dist/`.

Make targets of note:

- `make build`: build pillar containerized
- `make build BUILD=local`: build pillar via your local golang installation
- `make builder-image`: build the builder image for your user
- `make shell`: launch a shell inside the builder image, whence you can run all commands. It also sets `BUILD=local`, so that you can run just `make build` inside.

## States and their transitions

Object inside agents (more precisely inside [pubsub](./pubsub/doc.go)) may have two descriptions:

1. ObjectConfig - description of object comes directly from config or generated during parsing of config
1. ObjectStatus - description of object generated by agent as a current state of processing of ObjectConfig

Those structures are described in [types](./types). Agents have publishers and subscribers via pubsub to types from other agents. ObjectStatus may have several states inside, generally, they have `SwState`, but may have additional booleans or `SubStates` described current state of object.
Agents obtain ObjectConfig and do their work by creating ObjectStatus in Initial state and moving it to another values. ObjectStatus is sending through pubsub to another agents, which have subscribes, and actions are fired in them depending on the State.
State transitions of the particular ObjectStatus should be written in one place (in one function, you can find several with names starts with doUpdate).

Actions may be sync (they must be executable in limited time and not depends on another, for example, check file or directory existence, touch file, make directory and so on) and async. The second variant is done by [worker](./worker/doc.go) package. Worker instance has limited job queue and parallel workers in pool.
Package worker provides utilities for creating background workers to perform tasks, that need to be queued up or will take a long time to run. In both cases, upon completion, actions call function with state transitions to toggle states if required.
