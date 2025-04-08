# Logging with Grafana, Alloy, and Loki

This repository includes config and `docker-compose.yml` files to set up a remote logging server with Grafana.

## Supported logging sources

New sources can be added as long as Alloy / Loki supports them, but as of now, raw http (like [curtin](https://curtin.readthedocs.io/en/latest/topics/reporting.html#example-http-request) uses during ubuntu-server installation), syslog and docker are built in to the configuration files.

- Point syslog to this machine's IP, port :5140, with TCP protocol.
    - By default, `syslog.alloy` is set up to injest syslog messages with **RFC 3164** format to accomodate ChirpStackOS and OpenWRT (this is not documented anywhere, but testing shows that it uses the older RFC 3164 format)
- Point webhook JSON log reporting to this machine's IP, port :5555, **with the endpoint `/loki/api/v1/raw`**
- Other machines' docker containers can forward their logs to the host machine's Loki instance, see the [forward docker logs](#forward-docker-logs) section below.

## Integration

This repository should ideally be included as a git submodule in a larger Grafana stack, integrated with other metrics / dashboards.

Example integration with a larger `docker-compose.yml`:

```yaml
services:

  # ... other services

  alloy-logging:
    extends:
      file: "./grafana-logging/docker-compose.yml"
      service: alloy-logging
    networks:
      - grafana-network # remember to change this if another network name is defined

  loki:
    extends:
      file: "./grafana-logging/docker-compose.yml"
      service: loki
    networks:
      - grafana-network # remember to change this if another network name is defined

  grafana:
    # ... other config
    volumes:
      # To provision Loki by default
      - ./grafana-logging/config/grafana/provisioning/datasource-loki.yml:/etc/grafana/provisioning/datasource-loki.yml:ro

  networks:
    grafana-network:
```

### Remote machine integration

If we wish to centralise log collection across many remote machines into one centralised logging server, we can use Syslog, HTTP, or forward the docker container logs.

#### Syslog

- Point syslog to this machine's IP, port :5140, with TCP protocol.
    - By default, `syslog.alloy` is set up to injest syslog messages with **RFC 3164** format to accomodate ChirpStackOS and OpenWRT (this is not documented anywhere, but testing shows that it uses the older RFC 3164 format)

#### HTTP

Point webhook JSON log reporting to this machine's IP, port :5555, **with the endpoint `/loki/api/v1/raw`**

#### Forward docker logs

In the remote machine, clone this repository as per normal, but the target `docker-compose.yml` is in the [`alloy-docker-forwarder`](./alloy-docker-forwarder/) folder.

- `cd` to the `alloy-docker-forwarder` folder
- Copy `.env.example` to `.env` (`cp .env.example .env`) (NOTE: This file does not persist in source control!)
- Edit the `.env` file to point the IP to the remote logging server's loki port, and change the machine name
- Run `docker compose up -d` in the folder

## Development

### Quick Start and Testing

Running `docker compose up -d` in this repository should spin up the required containers. Access the webui at port `:13000`, then you can explore logs at the `Explore > Logs` tab in the sidebar

### Data flow

Syslog / JSON over HTTP / Docker / ... --> Alloy (Agent) --> Loki (Server) --> Grafana (Display)

- Alloy is a OTel collector, it manages how data flows into the system, thus, inputs and ports are all defined with the `alloy` docker service and in the config files in `config/alloy/`
    - All config files in the directory will be [ran as one config file](https://grafana.com/docs/alloy/v1.7/reference/cli/run/#usage) for simplicity sake
- Loki is a Log Aggregator, it injests and stores logs, and manages the labelling / indexing / querying / storage of all the logs
    - Loki is currently set up in a Monolithic mode, however if there are too many logs to manage, refer to [the other possible deployment modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/)
    - Logs are currently persisted in the `loki-data` docker volume
- Grafana just reads and displays the logs from Loki. All queries keyed into Grafana are passed to Loki, and processed in Loki.

### Naming

- To allow for other deployment topologies, and potentially segmenting different Alloy instances, the Alloy service is deliberately named `alloy-logging`.
    - If scaling up in the future, refer to <https://grafana.com/docs/alloy/latest/set-up/deploy/>
- The Grafana service in this repository is named `grafana-logging` for separate testing in machines with multiple Grafana instances
    - NOTE! `grafana-logging` is intentionally set up with port `:13000` instead of the normal `:3000` for this purpose.

### Provisioning

To ensure that the Grafana setup is repeatable, we should always provision data sources and dashboards with an appropriate configuration file, instead of using the webui.

- Datasource YAMLs should be in `provisioning/datasources` directory, while Dashboard JSONs should be in `provisioning/dashboards` directory.
    - A [Loki datasource is included in this repository](./config/grafana/provisioning/datasources/datasource-loki.yml), however no dashboard JSONs have been created yet.
- Reference: <https://grafana.com/docs/grafana/latest/administration/provisioning>

### Labels

Loki has a [minimal indexing approach](https://grafana.com/oss/loki/), and only indexes the labels + timestamp.

[This](https://nsrc.org/activities/agendas/en/nmm-4-days/netmgmt/en/log-management/log-management-loki-overview.pdf) is a good slide deck and resource for understanding Loki, log streams, cardinality, and best practices.

#### RFC3164 Syslog

The RFC3164 syslog messages are not processed ideally by default. This also applies to RFC5424 messages, however there are more fields to be filled.

- Under the hood, Alloy uses the [go-syslog](https://github.com/leodido/go-syslog) parser (see [here](https://github.com/grafana/alloy/blob/ca20237442991ed314c69c8a83ab04c3277be9fb/internal/component/loki/source/syslog/internal/syslogtarget/syslogtarget.go#L18)) when injesting syslogs.
    - Here is an example output of a [parsed RFC3164 message](https://github.com/leodido/go-syslog/blob/e1d78c258095eb935654186707af3bf3c1cafea2/rfc3164/example_test.go#L26-L31)
- However, the [processed labels](https://github.com/grafana/alloy/blob/ca20237442991ed314c69c8a83ab04c3277be9fb/internal/component/loki/source/syslog/internal/syslogtarget/syslogtarget.go#L189-L206) are all [internal labels](https://grafana.com/docs/alloy/v1.7/tutorials/logs-and-relabeling-basics/#add-a-prometheusrelabel-component-to-your-pipeline) (scroll down to the warning from this link, labels starting with double underscores are dropped).
    - This is likely to prevent too many labels from being created.
    - This is documented (very poorly) in the [alloy `loki.source.syslog` docs](https://grafana.com/docs/alloy/v1.7/reference/components/loki/loki.source.syslog/#listener) ("All header fields ... are ... internal labels ...")
    - For both RFC3164 and RFC5424 messages, the following internal labels are applied:
        - `__syslog_message_severity`
        - `__syslog_message_facility`
        - `__syslog_message_hostname`
        - `__syslog_message_app_name`
        - `__syslog_message_proc_id`
        - `__syslog_message_msg_id` (not used in RFC3164)
        - However, `HOSTNAME`, `APP-NAME`, `PROCID` and `MSGID` is only explicity defined in [RFC5424](https://datatracker.ietf.org/doc/html/rfc5424#section-6). For RFC3164 messages, they are [derived in the parser](https://github.com/leodido/go-syslog/blob/e1d78c258095eb935654186707af3bf3c1cafea2/rfc3164/syslog_message.go#L34-L44).
    - For RFC5424 messages with `label_structured_data` set to `true`, more internal labels will then be created in the form of `__syslog_message_sd_<ID>_<KEY>` ([source](https://grafana.com/docs/alloy/v1.7/reference/components/loki/loki.source.syslog/#listener))
- Hence, in this repository, all of the internal labels have been relabelled to normal labels.
    - Even the [source's test file](https://github.com/grafana/alloy/blob/ca20237442991ed314c69c8a83ab04c3277be9fb/internal/component/loki/source/syslog/internal/syslogtarget/syslogtarget_test.go#L445-L458) relabels these
- To prevent too many labels from being created, only `hostname` is retained as a normal label, since it should make sense for each individual device to have its own log stream. Everything else is transformed by a `loki.process` into a structured metadata field.

#### Docker

`loki.source.docker` obtains all its information and labels from `discovery.docker` through the `targets` field, and the labels are documented [here](https://grafana.com/docs/alloy/v1.7/reference/components/discovery/discovery.docker/#exported-fields).

- However, since all the labels are internal labels, the default output of `loki.source.docker` is quite useless, with no information about which container is sending which log.
- In this repository, we have chosen to split `compose` and normal `docker run` container logs into their own different `service_name`.
    - We assume that most of the docker containers should be spun up via `docker compose` rather than individual `docker run`. Individual `docker run` containers are probably small tests or development containers, hence it will be useful to view those logs separately from the (presumably) production containers in `docker compose`
    - Logs from `compose` projects will be labelled `PREFIX.docker_compose`:
        - For these containers, only `docker_compose_project` will be labelled
        - The following other fields will be structured metadata:
            - `docker_compose_service` (this was chosen over container name as the container name is not always specified in the compose file)
            - `docker_compose_project_working_dir`
            - `docker_compose_network_name`
        - The assumption here is that there will be many services, and putting services as a label might increase the cardinality too much. The other structured metadata fields (`working_dir` and `network_name`) were exposed for ease of debugging.
    - Logs from `docker run` containers will be labelled `PREFIX.docker_run`
        - Only `docker_container_name` will be labelled.
    - Other labels and fields not mentioned here are dropped.

### Docker specific information

Exposing the docker daemon directly to Alloy was chosen over setting up another [logging driver](https://docs.docker.com/engine/logging/configure/) in docker / docker compose itself because it was easy to plug and play into existing containers and docker compose projects.

`gelf` and `syslog` logging drivers could be compatible with Alloy and Loki, however they are not tested. If mounting the docker daemon directly to the Alloy container is too big of a security vulnerability, consider using those logging drivers to push logs, and set up the corresponding `loki.source.xxx` to receive those logs.

#### File structure

For remote machines, we want to have an independent docker container running that is able to scrape logs from all docker containers and forward it to a centralised Loki server. That is contained in the `alloy-docker-forwarder` folder.

- To be able to reuse the relabelling rules and processing stages, all the crucial docker processing components are contained in a separate file, at [`./config/alloy/docker-processor.alloy`](./config/alloy/docker-processor.alloy).
- This file is mounted and imported into `alloy-docker-forwarder` as a [module](https://grafana.com/docs/alloy/v1.7/get-started/modules/)
- This same file is also concatenated into the combined Alloy config for use with the main logging server.

#### HTTP 400 "entry too far behind" error

- When Alloy attempts to send logs that are older than 1 hour, a HTTP 400 "entry too far behind" error will occur, and completely block any further logs from being sent through that Alloy pipeline.
    - So far, this is only observed in `loki.source.docker` since Alloy consumes all docker logs, even older ones from containers that have been running for >1h
        - The timestamp for all docker containers are processed [here in the source code](https://github.com/grafana/alloy/blob/537e036e73cbc0b62ae0dd6b82e72f5fffb4b646/internal/component/loki/source/docker/internal/dockertarget/target.go#L156-L166)
        - However, this timestamp isn't set to a label or structured metadata. Hence, when viewing on Grafana, the timestamp looks "invisible" and you can scratch your head for two hours figuring out how Loki even "detects" that the log is old.
        - Instead, it is included as part of the `logproto` message, which is the message format when sending from Alloy -> Loki (see [source code](https://github.com/grafana/alloy/blob/537e036e73cbc0b62ae0dd6b82e72f5fffb4b646/internal/component/loki/source/docker/internal/dockertarget/target.go#L207-L213) and [Alloy documentation for `loki.write` using `logproto`](https://grafana.com/docs/alloy/v1.7/reference/components/loki/loki.write/))
- To prevent this, a `stage.drop` is added to the `loki.process` block in `docker.alloy`, that drops all logs older than 30m.
- If this error appears for any other pipeline, consider implementing a similar fix.

#### Docker remote daemon access

- Remote access to remote machines' docker daemons are supported, hence bypassing the need to run another container on the remote machine. However **this is not recommended** since docker daemons usually have root access to the machine. Any networking security leak would result in very bad consequences, since the remote agent could also obtain root access to the machine.
- If this is still desired, it has been tested to work with the following steps:
    - Follow the instructions [here](https://docs.docker.com/engine/daemon/remote-access/#configuring-remote-access-with-systemd-unit-file)
    - Change `host = "unix:///var/run/docker.sock"` to `host = "tcp://<replace-with-ip-addr>:2375"` (two lines to change!) in [`./config/alloy-docker/config.alloy`](./config/alloy-docker/config.alloy)
    - **Definitely** access the docker daemon only through the local network or a VPN
    - Consider using [TLS (HTTPS) to protect the docker daemon](https://docs.docker.com/engine/security/protect-access/) (not tested)

### Prometheus

A Prometheus service is included in this project's `docker-compose.yml` and is tested to be working and able to injest metrics from Alloy, Loki, and Grafana. However, dashboards and automatic provisioning are not set up yet (since the Alloy and Loki metrics honestly look quite useless as of now).
