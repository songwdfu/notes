# Decrusting Tokio Crate

## Tokio Runtime

### Scheduler

- Multi-thread scheduler

By default 1 OS thread per CPU core

Future trait's `Poll` method returns either `READY` or `PENDING`

`block_on` interface: takes future and return result of future


