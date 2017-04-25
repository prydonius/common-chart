# Common: The Helm Helper Chart

This chart is designed to make it easier for you to build and maintain Helm
charts.

It provides utilities that reflect best practices of Kubernetes chart development,
making it faster for you to write charts.

## Tips

A few tips for working with Common:

- Use `{{ include "some.template" | indent $number }}` to produce formatted output.
- Be careful when using functions that generate random data (like `common.fullname.unique`).
  They may trigger unwanted upgrades or have other side effects.

In this document, we use `RELEASE-NAME` as the name of the release.

## Resource Kinds

Kubernetes defines a variety of resource kinds, from `Secret` to `StatefulSet`.
We define some of the most common kinds in a way that lets you easily work with
them.

The resource kind templates are designed to make it much faster for you to
define _basic_ versions of these resources. They allow you to extend and modify
just what you need, without having to copy around lots of boilerplate.

To make use of these templates you must define a template that will extend the
base template. The name of this template is then passed to the base template
along with the root context, for example:

```yaml
{{- template "common.service" (list . "mychart.service") -}}
{{- define "mychart.service" }}
## Define overrides for your Service resource here, e.g.
# metadata:
#   labels:
#     custom: label
# spec:
#   ports:
#   - port: 8080
{{- end -}}
```

The `common.service` template is responsible for rendering the templates with
the root context and merging any overrides. As you can see, this makes it very
easy to create a basic `Service` resource without having to copy around the
standard metadata and labels.

Each implemented base resource is described in greater detail below.

### `common.service`

The `common.service` template creates a basic `Service` resource with the
following defaults:

- Service type (ClusterIP, NodePort, LoadBalancer) made configurable by `.Values.service.type`
- Named port `http` configured on port 80
- Selector set to `app: {{ template "common.fullname" }}` to match the default used in the `Deployment` resource

Example template:

```yaml
{{- template "common.service" (list . "mychart.mail.service") -}}
{{- define "mychart.mail.service" -}}
metadata:
  name: {{ template "common.fullname" . }}-mail # overrides the default name to add a suffix
  labels:                                       # appended to the labels section
    protocol: mail
spec:
  ports:                                        # composes the `ports` section of the service definition.
  - name: smtp
    port: 22
    targetPort: 22
  - name: imaps
    port: 993
    targetPort: 993
  selector:                                     # this is appended to the default selector
    protocol: mail
{{- end -}}
---
{{ template "common.service" (list . "mychart.web.service") -}}
{{- define "mychart.web.service" -}}
metadata:
  name: {{ template "common.fullname" . }}-www  # overrides the default name to add a suffix
  labels:                                       # appended to the labels section
    protocol: www
spec:
  ports:                                        # composes the `ports` section of the service definition.
  ports:
  - name: www
    port: 80
    targetPort: 8080
  selector:                                     # this is overrides the value of the default app selector
    app: {{ template "common.fullname" . }}-www
{{- end -}}
```

The above template defines _two_ services: a web service and a mail service. Note
that the `common.service` template defines two parameters:

  - The global context (usually `.`)
  - A template name containing the service definition overrides

The most important part of a service definition is the `ports` object, which
defines the ports that this service will listen on. Most of the time,
`selector` is computed for you. But you can replace it or add to it.

The output of running the above values through the earlier template is:

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: release-name-service
    chart: examples-0.1.0
    heritage: Tiller
    protocol: mail
    release: release-name
  name: release-name-service-mail
spec:
  ports:
  - name: smtp
    port: 22
    targetPort: 22
  - name: imaps
    port: 993
    targetPort: 993
  selector:
    app: release-name-service
    protocol: mail
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: release-name-service
    chart: examples-0.1.0
    heritage: Tiller
    protocol: www
    release: release-name
  name: release-name-service-www
spec:
  ports:
  - name: www
    port: 80
    targetPort: 8080
  selector:
    app: release-name-service-www
  type: ClusterIP
```

## `common.deployment`

The `common.deployment` template defines a basic `Deployment`. Underneath the
hood, it uses `common.container`.

By default, the pod template within the deployment only defines a single `app`
label as this is also used as the selector. The standard set of labels are not
used as some of these can change during upgrades, which causes the replica sets
and pods to not correctly match.

Example use:

```yaml
{{- template "common.deployment" (list . "mychart.deployment") -}}
{{- define "mychart.deployment" }}
## Define overrides for your Deployment resource here, e.g.
spec:
  replicas: {{ .Values.replicaCount }}
{{- end -}}
```

## `common.container`

The `common.container` template creates a basic `Container` spec to be used
within a `Deployment` or `ReplicaSet`. It holds the following defaults:

- The name is set to the chart name
- Uses `.Values.image` to describe the image to run, with the following spec:
  ```yaml
  image:
    repository: nginx
    tag: stable
    pullPolicy: IfNotPresent
  ```
- Exposes the named port `http` as port 80
- Lays out the resources spec using `.Values.resources`

Example use:

```yaml
{{- template "common.deployment" (list . "mychart.deployment") -}}
{{- define "mychart.deployment" }}
## Define overrides for your Deployment resource here, e.g.
spec:
  template:
    spec:
      containers:
      - {{ template "common.container" (list . "mychart.deployment.container") }}
{{- end -}}
{{- define "mychart.deployment.container" -}}
## Define overrides for your Container here, e.g.
# ports:
# - name: http
#   containerPort: 80
livenessProbe:
  httpGet:
    path: /
    port: 80
readinessProbe:
  httpGet:
    path: /
    port: 80
{{- end -}}
```

The above declares a `Deployment` with a single `Container`.
```
## Utilities

### `common.fullname`

The `common.fullname` template generates a name suitable for the `name:` field
in Kubernetes metadata. It is used like this:

```yaml
name: {{ template "common.fullname" . }}
```

The following different values can influence it:

```yaml
# By default, fullname uses '{{ .Release.Name }}-{{.Chart.Name}}. This
# overrides that and uses the given string instead.
fullnameOverride: "some-name"

# This adds a prefix
fullnamePrefix: "pre-"
# This appends a suffix
fullnameSuffix: "-suf"

# Global versions of the above
global:
  fullnamePrefix: "pp-"
  fullnameSuffix: "-ps"
```

Example output:

```yaml
---
# with the values above
name: pp-pre-some-name-suf-ps

---
# the default, for release "happy-panda" and chart "wordpress"
name: happy-panda-wordpress
```

Output of this function is truncated at 54 characters, which leaves 9 additional
characters for customized overriding. Thus you can easily extend this name
in your own charts:

```yaml
{{- define "my.fullname" -}}
  {{ template "common.fullname" . }}-my-stuff
{{- end -}}
```

### `common.fullname.unique`

The `common.fullname.unique` variant of fullname appends a unique seven-character
sequence to the end of the common name field.

This takes all of the same parameters as `common.fullname`

Example template:

```yaml
uniqueName: {{ template "common.fullname.unique" . }}
```

Example output:

```yaml
uniqueName: release-name-fullname-jl0dbwx
```

It is also impacted by the prefix and suffix definitions, as well as by
`.Values.fullnameOverride`

Note that the effective maximum length of this function is 63 characters, not 54.

### `common.metadata`

The `common.metadata` helper generates the `metadata:` section of a Kubernetes
resource.

This takes three objects:
  - .top: top context
  - .nameOverride: override the fullname with this name
  - .metadata
    - .labels: key/value list of labels
    - .annotations: key/value list of annotations
    - .hook: name(s) of hook(s)

It generates standard labels, annotations, hooks, and a name field.

Example template:

```yaml
{{ template "common.metadata" (dict "top" . "metadata" .Values.bio) }}
---
{{ template "common.metadata" (dict "top" . "metadata" .Values.pet "nameOverride" .Values.pet.nameOverride) }}
```

Example values:

```yaml
bio:
  name: example
  labels:
    first: matt
    last: butcher
    nick: technosophos
  annotations:
    format: bio
    destination: archive
  hook: pre-install

pet:
  nameOverride: Zeus

```

Example output:

```yaml
metadata:
  name: release-name-metadata
  labels:
    app: release-name-metadata
    heritage: "Tiller"
    release: "RELEASE-NAME"
    chart: metadata-0.1.0
    first: "matt"
    last: "butcher"
    nick: "technosophos"
  annotations:
    "destination": "archive"
    "format": "bio"
    "helm.sh/hook": "pre-install"
---
metadata:
  name: Zeus
  labels:
    app: release-name-metadata
    heritage: "Tiller"
    release: "RELEASE-NAME"
    chart: metadata-0.1.0
  annotations:
```

Most of the common templates that define a resource type (e.g. `common.configmap`
or `common.job`) use this to generate the metadata, which means they inherit
the same `labels`, `annotations`, `nameOverride`, and `hook` fields.

### `common.labelize`

`common.labelize` turns a map into a set of labels.

Example template:

```yaml
{{- $map := dict "first" "1" "second" "2" "third" "3" -}}
{{- template "common.labelize" $map -}}
```

Example output:

```yaml
first: "1"
second: "2"
third: "3"
```

### `common.labels.standard`

`common.labels.standard` prints the standard set of labels.

Example usage:

```
{{ template "common.labels.standard" . }}
```

Example output:

```yaml
app: release-name-labelizer
heritage: "Tiller"
release: "RELEASE-NAME"
chart: labelizer-0.1.0
```

### `common.hook`

The `common.hook` template is a convenience for defining hooks.

Example template:

```yaml
{{ template "common.hook" "pre-install,post-install" }}
```

Example output:

```yaml
"helm.sh/hook": "pre-install,post-install"
```

### `common.chartref`

The `common.chartref` helper prints the chart name and version, escaped to be
legal in a Kubernetes label field.

Example template:

```yaml
chartref: {{ template "common.chartref" . }}
```

For the chart `foo` with version `1.2.3-beta.55+1234`, this will render:

```yaml
chartref: foo-1.2.3-beta.55_1234
```

(Note that `+` is an illegal character in label values)

### `common.port` and `common.port.string`

`common.port` takes a port in either numeric or colon-numeric (":8080") syntax
and converts it to an integer.

`common.port.string` does the same, but formats the result as a string (in quotes)
to satisfy a few places in Kubernetes where ports are passed as strings.

Example template:

```yaml
port1: {{ template "common.port" 1234 }}
port2: {{ template "common.port" "4321" }}
port3: {{ template "common.port" ":8080" }}
portString: {{ template "common.port.string" 1234 }}
```

Example output:
```yaml
port1: 1234
port2: 4321
port3: 8080
portString: "1234"
```
