# Common: The Helm Base Chart

One little-known feature of [Helm](http://helm.sh) charts is the ability to share chart definitions
among all templates in a chart, including any of the subchart templates.

The `common` chart is a chart that defines commonly used Chart primitives that
can be used in all of your charts.

See the [Documentation](docs/index.md) for complete API documentation and examples.

## Goals

The common chart is being developed by members of the Helm and [Kubernetes Charts](https://github.com/kubernetes/charts) community, and stems from the
problems we've encountered managing a large repository of charts. The goals of
the common chart are:

- Reduce boilerplate in charts by providing a base to build from
- Provide a set of helper functions to express common patterns
- Develop and be the source of best practices for common Kubernetes patterns
- Facilitate structural updates for all charts as the Kubernetes API changes

### Non-goals

- Create an abstraction on top of the Kubernetes API

## Example Usage

Create a new chart:

```
$ helm create mychart
```

Include the common chart as a subchart:

```console
$ cd mychart/charts
$ helm fetch common
```

Use the `common.*` definitions in your code.

The Common chart has many other utilities.

## Developers

If you are developing on this project, you can use `make build` to build the
charts. Note that the makefile requires signing your chart.
