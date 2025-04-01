# Logging with Grafana, Alloy, and Loki

This repository includes config and `docker-compose.yml` files to set up a remote logging server with Grafana.

## Supported logging sources

New sources can be added as long as Alloy / Loki supports them, but as of now, raw http (like [curtin](https://curtin.readthedocs.io/en/latest/topics/reporting.html#example-http-request) uses during ubuntu-server installation) and syslog is built in to the configuration files.

- Point syslog to this machine's IP, port :5140, with TCP protocol.
    - By default, `config.alloy` is set up to injest syslog messages with **RFC 3164** format to accomodate ChirpStackOS and OpenWRT (this is not documented anywhere, but testing shows that it uses the older RFC 3164 format)

TODO: Injest docker logs

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

## Development

### Quick Start and Testing

Running `docker compose up -d` in this repository should spin up the required containers.

TODO: Grafana Provisioning with files

### Naming

- To allow for other deployment topologies, and potentially segmenting different Alloy instances, the Alloy service is deliberately named `alloy-logging`.
    - If scaling up in the future, refer to <https://grafana.com/docs/alloy/latest/set-up/deploy/>
- The Grafana service in this repository is named `grafana-logging` for separate testing in machines with multiple Grafana instances
    - NOTE! `grafana-logging` is intentionally set up with port `:13000` instead of the normal `:3000` for this purpose.
