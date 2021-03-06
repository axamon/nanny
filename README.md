
# Nanny
[![Build Status](https://travis-ci.org/lunemec/nanny.svg?branch=master)](https://travis-ci.org/lunemec/nanny) [![Go Report Card](https://goreportcard.com/badge/github.com/lunemec/nanny)](https://goreportcard.com/report/github.com/lunemec/nanny) [![Maintainability](https://api.codeclimate.com/v1/badges/224b9390145c2e5a8046/maintainability)](https://codeclimate.com/github/lunemec/nanny/maintainability) [![codecov](https://codecov.io/gh/lunemec/nanny/branch/master/graph/badge.svg)](https://codecov.io/gh/lunemec/nanny)

Nanny is a monitoring tool that monitors the **absence of activity**.

Nanny runs an API server, which expects to be called every N seconds, and if no such call is made, Nanny notifies you.

Nanny can notify you via these channels (for now):
* print text to stderr
* email
* sentry

## Example
Run API server:
```bash
$ LOGXI=* ./nanny
14:21:07.969059 INF ~ Using config file
   path: nanny.toml
14:21:07.977322 INF ~ Nanny listening addr: localhost:8080
```
Call it via curl:
```bash
curl http://localhost:8080/api/v1/signal --data '{ "name": "my awesome program", "notifier": "stderr", "next_signal": "5s" }'
```
With this call, you tell nanny that if program named `my awesome program` does not call again within `next_signal` (5s), it should notify you using `stderr` notifier.

After 5s pass, nanny prints to *stderr*:
```bash
2018-06-26T14:24:29+02:00: Nanny: I haven't heard from "my awesome program@localhost:44554" in the last 5s! (Meta: map[])
```

## Installation
The easiest way is to download .tar.gz from **releases** section, edit `nanny.toml` and run it.

Or you can clone this repository and compile it yourself:
```bash
git clone https://github.com/lunemec/nanny.git
cd nanny
make build
```

## Configuration
See nanny.toml for a configuration example. The fields are self-explanatory (I think). Please create an issue if anything does not make sense!

All enabled notifiers can be used via API, so enable only those you wish to allow.

### ENV variables
ENV variables can be used to override the config file settings. They should be prefixed with `NANNY_`.

Example:
```
NANNY_NAME="custom name" NANNY_ADDR="localhost:9090" LOGXI=* ./nanny
```

## Monitoring nanny
You can use one Nanny to monitor another Nanny or create a monitored Nanny-pair.

Run 1st nanny, on port 8080 that will use nanny at port 9090 as its monitor:
```bash
NANNY_ADDR="localhost:8080" LOGXI=* ./nanny --nanny "http://localhost:9090/api/v1/signal" --nanny-notifier "stderr"
```

Run 2nd nanny, on port 9090 that will use 1st nanny on port 8080:
```bash
NANNY_ADDR="localhost:9090" LOGXI=* ./nanny --nanny "http://localhost:8080/api/v1/signal" --nanny-notifier "stderr"
```

You may get some warnings until both Nannies are listening, but they will recover. If you stop one of them, the other will notify you.

Be sure to change nanny SQLite DB location! They would share the same DB and it could cause strange behavior.
This can be done in the config file or by setting `NANNY_STORAGE_DSN` ENV variable.

## Logging
By default, nanny logs only errors. To enable more verbose logging, use `LOGXI=*` environment variable.

## Adding custom data (tags) to notifications
You can add extra meta-data to the API calls, which will be passed to all the notifiers. Metadata must conform to type `map[string]string`.

```bash
curl http://localhost:8080/api/v1/signal --data '{ "name": "my program", "notifier": "stderr", "next_signal": "5"s "meta":{"custom": "metadata"} }'
```

These metadata will be displayed in the messages for stderr and email, and in tags for sentry.

## Contributing
Contributions welcome! Just be sure you run tests and lints.

```bash
$ make
  Build                          
make build                            Build production binary.                           
  Dev                            
make run                              Run Nanny in dev mode, all logging and race detector ON. 
make test                             Run tests.                                         
make vet                              Run go vet.                                        
make lint                             Run gometalinter (you have to install it). 
```

## FAQ
> Why write such a tool?

Sometimes you expect some job to run, say cron. But when someone messes up your crontab, or the machine is offline, you might not be notified.

Also often programs just log errors and fail silently, with nanny they fail loudly.

> How do I secure my nanny?

To use HTTPS, or authentication you should use a reverse proxy like Apache or Nginx.