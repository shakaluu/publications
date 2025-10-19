---
theme: default
title: Helm
info: |
  ## Helm
  A presentation for CNCF Meetup Innsbruck
  by Bernhard Bucher and Michael Wolf
# apply UnoCSS classes to the current slide
class: text-left
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: false
---

# Helm

The package manager for Kubernetes

<div class="abs-br m-6 text-xl">
  <a href="https://miro.com/app/board/uXjVJ3pkFaw=/" target="_blank" class="slidev-icon-btn">
    <img src="/miro.png" class="w-6">
  </a>
  <a href="https://github.com/shakaluu/publications/tree/helm-intro" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

---

# What is Helm?

Helm is a graduated project in the CNCF and is maintained by the Helm community  
It is a package manager for Kubernetes

<div class="abs-tr m-6 w-10">
  <img src="/helm-logo.png">
</div>

<v-click>

## Advantages

<br>

- 📝 **Manage complexity** - Helm Charts describe most complex apps, repeatable installation

</v-click>

<v-clicks>

- 🛠 **Easy Updates** - in-place upgrades and custom hooks
- 📤 **Simple Sharing** - easy to version, share, and host on public or private servers
- 🔥 **Rollbacks** - helm rollback to roll back to an older version of a release with ease

</v-clicks>

<v-click>

## Disadvantages

- well...

</v-click>

<!--
Package manager with format of Helm Charts - compared to .deb or .rpm in parts of the Linux world.

Differences between "real" Package Managers and Helm.
- automated dependency resolution
- Highly customizable during install - unlike other package managers, maybe if you use Gentoo and it's Portage.
-->

---
transition: slide-up
---

# Helm Charts

contain a collection of files defining a set of K8s resources to deploy a specific application

```yaml {none|2|2,5|2,5,9-10|all}
wordpress/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  values.yaml         # The default configuration values for this chart
  values.schema.json  # OPTIONAL: A JSON Schema for imposing a structure on the values.yaml file
  charts/             # A directory containing any charts upon which this chart depends.
  crds/               # Custom Resource Definitions
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
```

<br>

<div v-after>

## Templates and Values

<br>

- 📝 YAML based
- 🐹 Go Templating Language
- 🌻 sprig library

</div>

---

# Let's have a look

```
/templates/service.yaml
```

````md magic-move
```yaml {*|5|8|*}{lines:true}
# prettier-ignore
apiVersion: v1
kind: Service
metadata:
  name: {{ default (printf "%s-svc" .Release.Name | trunc 63 | trimSuffix "-") .Values.service.name }}
spec:
  selector:
    app: {{ .Chart.Name | lower }}
  ports:
    - port: {{ .Values.service.port | default 80 }}
```

```yaml {*}{lines:true}
# prettier-ignore
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: mychart
  ports:
    - port: 8080
```
````

```
/values.yaml
```

```yaml {*}{lines:true}
service:
  name: myapp
  port: 8080
```

<br>
<div v-after>

```bash {*}
$ helm template my-release . -f values.yaml
```

</div>

---

# Dependency Management

```
/Chart.yaml
```

```yaml {*}{lines:true}
apiVersion: v2
name: mychart
version: 1.0.0

dependencies:
  - name: nginx
    version: "1.2.3"
    repository: "https://example.com/charts"
  - name: redis
    version: "3.2.1"
    repository: "https://another.example.com/charts"
```

<br>

<v-click>
```bash {*}
$ helm dependency update
```

- 📥 Resolves the dependency and downloads the chart(s)

</v-click>

<v-clicks>

- 💼 Stored in the `/charts` directory as .tgz archive

</v-clicks>

---

# Subcharts and Global Values

The `values.yaml` file has specific sections for values considered global and additionally specific to subcharts.

<v-clicks>

- 💭 A subchart is considered "stand-alone", it can never explicitly depend on its parent chart
- 🚧 For that reason, a subchart cannot access the values of its parent or siblings
- ✏️ A parent chart can override values for subcharts
- 🌍 Helm has a concept of global values that can be accessed by all charts

</v-clicks>

<br>

<v-click>

````md magic-move
```yaml {*}{lines:true}
apiVersion: v2
name: mychart
version: 1.0.0

dependencies:
  - name: nginx
    version: "1.2.3"
    repository: "https://example.com/charts"
  - name: redis
    version: "3.2.1"
    repository: "https://another.example.com/charts"
```

```yaml {*}{lines:true}
global:
  env: dev
service:
  name: myapp
  port: 8080
nginx:
  enabled: false
redis:
  test: true
```
````

</v-click>

---

# Named Templates

A named template (sometimes called a partial or a subtemplate) is simply a template defined inside of a file, and given a name

```yaml [/templates/_helpers.tpl] {*}{lines:true}
{{- define "mychart.app" -}}
app_name: {{ .Chart.Name }}
app_version: "{{ .Chart.Version }}"
{{- end -}}
```

<br>

<v-click>

````md magic-move
```yaml {*|6,9}{lines:true}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Chart.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
data:
  myvalue: "Hello World"
{{ include "mychart.app" . | indent 2 }}
```

```yaml {*}{lines:true}
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-configmap
  labels:
    app_name: mychart
    app_version: "1.0.0"
data:
  app_name: mychart
  app_version: "1.0.0"
```
````

</v-click>

---

# Using ArgoCD with Helm (our Approach)

## GitOps State Repository

<v-clicks>

- ✂️ Parent chart to decouple state from application chart
- 🔧 Additional configuration for notifications, owning team, etc.

```
path/in/gitops-state/*/Chart.yaml
path/in/gitops-state/*/config.json
```

```yaml {*}{lines:true}
apiVersion: v2
name: my-fancy-application
version: 1.0.0

dependencies:
  - name: my-fancy-application
    repository: oci://registry/my-domain
    version: 2.3.4
```

</v-clicks>

---

# Using ArgoCD with Helm (our Approach)

## ArgoCD Applicationset

<v-clicks>

🔎 Git Generator discovering parent chart(s) folder

```yaml {*}{lines:true}
generators:
  - git:
      files:
        - path: "path/in/gitops-state/*/config.json"
      repoURL: my-repo
      revision: main
```

</v-clicks>

---

## ArgoCD Applicationset

<v-clicks>

🚀 Automatic creation (based on template) of ArgoCD application using Helm chart

```yaml {*}{lines:true}
# prettier-ignore
source:
  repoURL: my-repo
  targetRevision: main
  path: {{ .path.path }}
  helm:
    releaseName: "my-fancy-application"
    ignoreMissingValueFiles: true
    valueFiles:
      - "additional-values.yaml"
```

</v-clicks>

---

# Using Helm CLI VS Helm in ArgoCD

## Different Approaches of Applying

- `helm install` renders and verifies chart and applies it through K8s API
- ArgoCD runs `helm template` and applies the result with `kubectl apply`

## Keep in Mind When Using ArgoCD

- No Helm release handling in ArgoCD
- Template result is not tested for valid resources before applying
- Duplicate resources possible (last rendered will be applied)
- [Helm hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/#helm-hooks) are mapped to ArgoCD hooks

---

# Be Type Aware

## Integer Example

```
/values.yaml
```

```yaml {*}{lines:true}
service:
  port: 8080
  targetPort: 8080
```

```
/templates/deployment.yaml
```

```yaml {*}{lines:true}
# prettier-ignore
ports:
  - port: "{{ .Values.service.port }}"
    targetPort: {{ .Values.service.targetPort }}
```

---

# Be Type Aware

## Env Var Example

```
/values.yaml
```

```yaml {*}{lines:true}
sayHello: true
```

```
/templates/deployment.yaml
```

```yaml {*}{lines:true}
containers:
  - name: hello
    image: "hello"
    env:
      - name: SAY_HELLO
        value: "{{ .Values.sayHello }}"
```

---

# Summary of Recommendations

## Validation of Values

- Be type aware
- Use conversion functions
- Implement schema for values
- Validate values with functions and return errors

## Test Helm Charts

- Render chart (helm template) and verify results
- Dry-run chart installation (helm install –dry-run)
- Not supported by ArgoCD: Implement chart tests and run them after installing (helm test)
- Automate!

---

# Problems with Helm

Till now it was a salespitch - let's explore the dark side...

<v-click>

- Public Charts on `artifacthub.io` can be tricky in various ways

```bash
$ helm search hub redis -o yaml | yq '. | length'
174
```

</v-click>
<br>
<v-click>

- Whitespaces and Indentation errors - no debugging tool

```bash
{{-, -}}, indent, nindent, Tabs/Spaces
```

</v-click>
<br>
<v-click>

- Puzzling Error messages

```bash
$ helm template . -f values.dev.yaml > results.yaml
Error: failed to parse values.dev.yaml: error converting YAML to JSON: yaml: line 15: did not find expected key
```

</v-click>

<v-click>

```yaml {*}{lines:true,startLine:15}
# The init-container is for local development only! In DEV/HARD/PROD a migration-job is used!
```

</v-click>

---

# Problems are here to be solved...

for some of the issues there might be a fix

<v-click>

- Check for Helm Charts not only on Artifacthub

```bash
$ helm repo add <repo-name> https://helm.redis.io/
```

</v-click>
<br>
<v-click>

- `helm template` and the `--debug` flag are your friends - a long distance, see once in a year friends

```bash
$ helm template ./mychart --debug
```

</v-click>

---

# What's next...

... for the Helm project

<br>

### 🎉 Helm 4.0 November Release 🎉

<br>

<v-clicks>

- 🔑 Enhanced OCI Support - install by digest  
  `helm install myapp oci://registry.example.com/charts/app@sha256:abc123`
- 🚀 Performance - Faster dependency resolution and new content-based chart caching
- 📢 Error Messages - Clearer, more helpful error output.
- 🔒 Registry Authentication - Better OAuth and token support for private registries.

</v-clicks>

---

# What's next...

... for the Helm project

<br>

### KYAML with Kubernetes 1.34

<br>

<v-clicks>

- 🎯 Explicitness - The type of every value is obvious, avoiding ambiguity issues
- 🚫 Error Reduction - Avoids common parsing traps in YAML
- 🔄 Version Control Friendly - Cleaner diffs, fewer merge conflicts
- 🔧 Tool Compatibility - Fully compatible with existing YAML processing tools
- 💡 Better IDE Support - JSON-like structure provides better syntax highlighting and validation

</v-clicks>

---

# YAML vs. KYAML

````md magic-move
```yaml
replicaCount: 3
image:
  repository: my-app
  tag: latest
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
env:
  - name: LOG_LEVEL
    value: debug
  - name: FEATURE_FLAG
    value: true
resources:
  limits:
    cpu: "500m"
    memory: "256Mi"
  requests:
    cpu: "250m"
    memory: "128Mi"
```

```yaml
# prettier-ignore
{
  replicaCount: 3,
  image: { repository: "my-app", tag: "latest", pullPolicy: "IfNotPresent" },
  service: { type: "ClusterIP", port: 80 },
  env: [
    { name: "LOG_LEVEL,", value: "debug" },
    { name: "FEATURE_FLAG,", value: true },
  ],
  resources: {
    limits: { cpu: "500m", memory: "256Mi" },
    requests: { cpu: "250m", memory: "128Mi" },
  },
}
```
````

<br>

<div v-after class="text-center">
KYAML, a safer, less ambiguous subset of YAML that maintains full compatibility with existing tooling while eliminating many of the most common YAML pitfalls.
</div>

---
layout: end
---

# Thank You!

<div class="abs-br m-6 text-xl">
  <a href="https://miro.com/app/board/uXjVJ3pkFaw=/" target="_blank" class="slidev-icon-btn">
    <img src="/miro.png" class="w-6">
  </a>
  <a href="https://github.com/shakaluu/publications/tree/helm-intro" target="_blank" class="slidev-icon-btn">
    <carbon:logo-github />
  </a>
</div>

<div class="abs-bl m-6 text-xs text-left">
<div>
  <carbon:logo-slack />&nbsp;
  <a href="https://cloud-native.slack.com/archives/D08QGRD9Z16" target="_blank">Bernhard Bucher</a>
</div>
<div>
  <carbon:logo-slack />&nbsp;
  <a href="https://cloud-native.slack.com/archives/D09QFSQ1M9N" target="_blank">Michael Wolf</a>
</div>
</div>
