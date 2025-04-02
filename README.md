# Logging with Grafana, Alloy, and Loki

This repository includes config and `docker-compose.yml` files to set up a remote logging server with Grafana.

## Supported logging sources

New sources can be added as long as Alloy / Loki supports them, but as of now, raw http (like [curtin](https://curtin.readthedocs.io/en/latest/topics/reporting.html#example-http-request) uses during ubuntu-server installation), syslog and docker are built in to the configuration files.

- Point syslog to this machine's IP, port :5140, with TCP protocol.
    - By default, `syslog.alloy` is set up to injest syslog messages with **RFC 3164** format to accomodate ChirpStackOS and OpenWRT (this is not documented anywhere, but testing shows that it uses the older RFC 3164 format)
- Point webhook JSON log reporting to this machine's IP, port :5555, **with the endpoint `/loki/api/v1/raw`**

## Integration

This repository should ideally be included as a git submodule in a larger Grafana stack, integrated with other metrics / dashboards.

Example integration with a larger `docker-compose.yml`:

```yaml
services:

  # ... other services

  alloy-logging:
    extends:
      file: grafana-logging/docker-compose.yml
      service: alloy-logging
    networks:
      - grafana-network # remember to change this if another network name is defined

  loki:
    extends:
      file: grafana-logging/docker-compose.yml
      service: loki
    networks:
      - grafana-network # remember to change this if another network name is defined

  networks:
    grafana-network:
```

To provision Loki by default without clicking through the webui, mount [`grafana/provisioning/datasources/datasource-loki.yml`](./config/grafana/provisioning/datasources/datasource-loki.yml) in the Grafana container. Example:

```yaml
services:
  grafana:
    # ... other config
    volumes:
      - ./grafana-logging/config/grafana/provisioning/datasource-loki.yml:/etc/grafana/provisioning/datasource-loki.yml:ro
```

## Development

### Quick Start and Testing

Running `docker compose up -d` in this repository should spin up the required containers. Access the webui at port `:13000`, then you can explore logs at the `Explore > Logs` tab in the sidebar

### Data flow

Syslog / JSON over HTTP / Docker / ... --> Alloy (Agent) --> Loki (Server) --> Grafana (Display)

- Alloy is a OTel collector, it manages how data flows into the system, thus, inputs and ports are all defined with the `alloy` docker service and in the config files in `config/alloy/`
    - All config files in the directory will be [ran as one config file](https://grafana.com/docs/alloy/v1.7/reference/cli/run/#usage), there is no concept of importing as of now
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
