# service-framework

`service-framework` is a Helm chart of extensible kubernetes resources with common metadata format that are enabled and customized by the values passed into the **service-framework** Helm `.Values` dictionary.  The kubernetes resources currently created by **service-framework** based on values are 
- Deployments - see [ability to disable](#ability-to-disable), [common map attributes](#common-map-attributes) and [deployments: map](#deployments-map-item-specific-attributes)
  - note: a PodDisruptionBudget is created for each Deployment
- Jobs - see [ability to disable](#ability-to-disable), [common map attributes](#common-map-attributes) and [jobs: map](#jobs-map-item-specific-attributes)
- Services - see [ability to disable](#ability-to-disable) and [deployment.service map](#deployment.service-map)
- Ingresses - see [ability to disable](#ability-to-disable) and [deployment.service.ingress map](#deployment.service.ingress-map)
- PersistentVolumes and associated PersistentVolumeClaims
- ServiceAccount, Role, and RoleBinding


**service-framework** is meant to be used as a "helper" chart brought in through a "parent" chart's requirements.yaml file - typically using `alias` like this:

```
dependencies:
- name: service-framework
  alias: serviceFramework
  version: 1.0.4
  repository: *TBD REPLACE THIS*
```

The **service-framework** Values are documented [below](#service-framework-Values).  The parent chart can pass values to **service-framework** be declaring values namespaced with serviceFramework.  For example:
```
serviceFramework:
  nameOverride: some-chart # this is typically the partent chart name as defined in its Chart.yaml
  profile: none
  deployments:
    "nginx":
      service:
        tbd...
      enabled: Values.nginx.enabled # if a path in dictionary, do not start with leading '.'
      containers:
        "main":
          REVISIT

nginx:
  enabled: true
  foo: bar
```

A powerful aspect of the **service-framework** is its ability to be customized by defining [global maps](#service-framework-global-values).  These global maps can define configuration  based on values set into the serviceFramework dictionary such as `serviceFramework.nameOverride` or `serviceFramework.profile`.

Note, since **service-framework** was created to support clusters hosted in AWS it has a few AWS-specific functionalities that should not cause a problem in non AWS environments.

-----

## service-framework Values

## `.Values.commonEnv:` array

`.Values.commonEnv:` is an array that each **deployment.container** or **job.container** may request be added to its **.env:** array definition.  **commonEnv:** array items may include **service-framework** custom items.  For more information, please refer to the [container.env array definition](#container.env-array).

## `.Values.jobs:` and `.Values.deployments:` maps

`.Values.jobs:` is a map that defines kubernetes **Job** resources; and `.Values.deployments:` is a map that defines kubernetes **Deployment** and **Service** resources.  If these maps are not in the dictionary, **service-framework** will render no resources.

### ability to disable

It is possible to explicitly enable/disable the resources depending on the following **.enabled** attribute on the following map items:

| **deployment** or **job** `.enabled` | kubernetes `Deployment` or `Job` rendered |
| :----------------------------------- | :---------------------- |
| not defined        | yes                                       |
| `false`            | no                                        |
| path.in.dictionary | yes iff **path.in.dictionary** is defined and not `false` |
| *any other value*  | yes                                       |

| deployment.service `.enabled` | kubernetes `Service` rendered |
| :---------------------------- | ------------------ |
| parent deployment is disabled | no                 |
| not defined                   | no                 |
| `false`                       | no                 |
| *any other value*             | yes                |

| deployment.service.ingress `.enabled` | kubernetes `Service` rendered |
| :---------------------------- | ------------------ |
| parent deployment is disabled | no                 |
| parent service is disabled    | no                 |
| not defined                   | no                 |
| `false`                       | no                 |
| *any other value*             | yes                |



### common map attributes
| attribute                | optional | default | kubernetes `Job` or `Deployment` resource result                   |
| :------------------------------------- | :------: | ------- | ------------------------------------------------------------------ |
| map name                 | n | | used as part of `.metadata.name` as described below (more below *here - tbd, make link* ) |
| `.nameOverride`          | n | | `.metadata.labels.app` set to this value, also used as part of `.metadata.name` as described below |
| `.releaseType`           | y | "release" | used as part of `.metadata.name` as described below |
| *above three*                  |   | | `.metadata.name` set to `.nameOverride`-{ map name }-`.releaseType` |
| `.assumedIamRole`        | y | | `spec.template.metadata.annotations.iam.amazonaws.com/role` added with this value |
| `.imagePullSecrets`      | y | | if defined, these array elements are added to `spec.template.spec.imagePullSecrets:` (see also [global.imagePullSecrets](#imagePullSecrets)) |
| `.automountServiceAccountToken` | y | `false` |`spec.template.spec.automountServiceAccountToken` set to this value |
| `.volumes:`              | y | | array of optional `spec.template.spec.volumes` items (more below in [volumes map](#volumes-map)) |
| `.defaultImage:`         | y | | see [image map](#defaultImage-or-container.image-map) |
| `.containers.image:`     | y | | see [image map](#defaultImage-or-container.image-map) |
| *above two*              |   | | one of the above two must be defined |
| `.commonContainers:`     | y | | map of [commonContainers](#global-commonContainers-map) to include with containers: map (below).  each map item [customize](#commonContainers-map) its matching commonContainer |
| `.containers:`           | n | | array of `spec.template.spec.containers` items (more below [containers map](#containers-map)) |

#### volumes map

Each job or deployment **.volumes:** array item results in a array entry in the corresponding kubernetes **Job** or **Deployment** `spec.template.spec.volumes` map.

| attribute                | optional | default | kubernetes `Job` or `Deployment` volumes array result                |
| :------------------------- | :------: | ------- | ------------------------------------------------------------------ |
| `.name`                  | y | | volume .name: value  |
| `.configMap`             | y | | volume .configMap: value  |
| `.secret`                | y | | volume .secret: value  |
| `.persistentVolumeClaim` | y | | volume .persistentVolumeClaim: value  |
| `.releaseRevisionConfigMap`| y | | volume is a configMap volume where CM name is constructed to include the Helm release revision [like this](#releaseRevisionConfigMap-rendered) |

##### releaseRevisionConfigMap rendered
```
  - name: {{ .releaseRevisionConfigMap }}
    configMap:
      name: {{ .Values.nameOverride }}-{{ .releaseRevisionConfigMap }}-{{ .Values.releaseType }}-{{ .Release.Revision }}
```

#### defaultImage or container.image map

| attribute                | optional | default | kubernetes `Job` or `Deployment` **.containers:** array result             |
| :----------------------- | :------: | ------- | ------------------------------------------------------------------ |
| `.repository`    | n | | used as part of container `.image:` value as described below |
| `.tag`           | n | | used as part of container `.image:` value as described below |
| *above two*      |   | | container `.image:` set to {{ .repository }}:{{ .tag }}
| `.pullPolicy`    | n | | container `.imagePullPolicy:` value |

#### commonContainers map

Each **jobs:** or *deployments:** map item may add and specialize "common containers" to its [containers map](#containers-map) by defining a **commonContainers:** map

| attribute                | optional | default | kubernetes `Job` or `Deployment` **.containers:** array result             |
| :----------------------- | :------: | ------- | ------------------------------------------------------------------ |
| map name                 | n | | the name of the [commonContainers](#global-commonContainers-map) that should be included |
| `.enabled:`              | y | false | whether this commonContainer should be included in this Job or Deployment's containers: array |
| `.env:`                  | y | | map of env array name / value items to override or add to this container |
| *others...*              | y | | see [containers map](#containers-map) for other valid attributes |


#### containers map

Each job or deployment **.containers:** map item results in a array entry in the corresponding kubernetes **Job** or **Deployment** `spec.template.spec.containers` map.

| attribute                | optional | default | kubernetes `Job` or `Deployment` **.containers:** array result             |
| :----------------------- | :------: | ------- | -------------------------------------------------------------------- |
| map name                 | n | | container .name: value  |
| `.image:`                | y | | see [image map](#defaultImage-or-container.image-map) |
| `.livenessProbe:`        | y | | if defined, .livenessProbe: is set to this map's value |
| `.readinessProbe:`       | y | | if defined, .readinessProbe: is set to this map's value |
| `.lifecycle:`            | y | | if defined, .lifecycle: is set to this value |
| `.ports:`                | y | | if defined, .ports: is set to matching values (see [ports array](#container.ports-array)) |
| `.workingDir`            | y | specified by image: | if defined, .workingDir: is set to this value |
| `.command`               | y | specified by image: | if defined, .command: is set to this value |
| `.arg`                   | y | | .args: is set to value `["-x", {{ $arg | quote }}]` |
| `.carg`                  | y | | .args: is set to value `["-c", {{ $carg | quote }}]` |
| `.args:`                 | y | | same as .arg above, but specified .args: is assumed to be an array |
| `.volumeMounts:`         | y | | if defined, .volumeMounts: is set to this value after .volumeMounts.fromTemplate processed (see [volumeMounts array](#container.volumeMounts-array)) |
| `.resources:`            | y | | if defined, .resources: is set to this map's value |
| `.includeCommonEnv`      | y | | if true, add [.Values.commonEnv:](#.Values.commonEnv:-array) items to [container.env: array](#container.env:-array) |
| `.env:`                  | y | | |

REVISIT: i believe both job.container and deployment.container may have initJobs: array... REMOVE THIS???

##### container.ports array

| attribute                | optional | default | kubernetes `Job` or `Deployment` **.conatiners.ports:** array result                 |
| :----------------------- | :------: | ------- | ------------------------------------------------------------------ |
| `.name`                  | y | | port .name: value  |
| `.containerPort`         | n | | port .name: value  |
| `.bindToService`         | y | | if defined and not "false" this deployment's kubernetes `Service` sets a ports: port.targetPort to this port |

##### container.volumeMounts array

| attribute                | optional | default | kubernetes `Job` or `Deployment` **.conatiners.volumeMounts:** array result       |
| :----------------------- | :------: | ------- | ------------------------------------------------------------------ |
| `.name`                  | n | | volumeMount .name: value      |
| `.scriptVolume`          | y | | set name to helm-scripts-volume - REVISIT: get rid of this?????   |
| `.mountPath`             | n | | volumeMount .mountPath: value |
| `.readOnly`              | y | | volumeMount .readOnly: value |
| `.fromTemplate`          | y | | if specified, include this template name via templating:  `{{ include $fromTemplate $ | indent 12 }}` |

##### container.env: array

| array item               | optional | default | kubernetes `Job` or `Deployment` **.conatiners.env:** array result       |
| :----------------------- | :------: | ------- | ------------------------------------------------------------------ |
| `- fromTemplate`         | y | | if specified, include this template name via templating: `{{ include $fromTemplate $ | indent 12 }}` |
| `- name`                 | y | | env array item `-name:` value      |
| *- name requires one of following 4 attributes* |
| `value`                  | n | | env array item `value:` value |
| `valueFrom:`             | n | | env array item `valueFrom:` map value (from Secret or ConfigMap) |
| `templateGetValue`       | n | | env array item `value:` is set to the value specified by templateGetValue |
|                          |   | | e.g., `templateGetValue: Values.some.value` results in `value: {{ .Values.some.value }}` |
| `valueFromRevision:`     | n | | map to define an env var from a **ConfigMap** or **Secret** where its **name** has .Release.Revision added.  [more info](#container.env-.valueFromRevision-map) |

##### container.env .valueFromRevision map

**service-framework** **container.env:** item **.valueFromRevision:** is a map that defines the creation of an environment variable whose value comes from a **ConfigMap** or **Secret** where "-{{ .Release.Revision }}" is appended to the specified name.

| attribute                | optional | default | kubernetes `Job` or `Deployment` **.conatiners.env:** array item **value:** map |
| :----------------------- | :------: | ------- | -------------------------------------------------------------------- |
| `.configMapKeyRef:`      | y | | map that defines creating an env variable **value** from a ConfigMap | 
| `.configMapKeyRef.name:` | n | | `-name:` is set to {{ $configMapKeyRef.name }}-{{ .Release.Revision }} | 
| `.configMapKeyRef.key:`  | n | | `  key:` is set to this value | 
| `.secretKeyRef:`         | y | | map that defines creating an env variable **value** from a Secret | 
| `.secretKeyRef.name:`    | n | | `-name:` is set to {{ $secretKeyRef.name }}-{{ .Release.Revision }} | 
| `.secretKeyRef.key:`     | n | | `  key:` is set to this value | 




### jobs map item specific attributes
| attribute                | optional | default | kubernetes `Job` resource result                                     |
| :--------------------------- | :------: | ------- | -------------------------------------------------------------------- |
| `.annotations:`          | y | | map of attributes that will be added to `Job`'s `metadata.annotaions:` | 
| `.restartPolicy:`        | y | `OnFailure` | `spec.template.spec.restartPolicy:` | 

### deployments map item specific attributes
| attribute                | optional | default | kubernetes `Deployment` resource result                              |
| :----------------------------- | :------: | ------- | -------------------------------------------------------------------- |
| `.nameOverride`      | n | `.spec.selector.matchLabels.app` set to this value |
| `.annotations:`      | n | | map of annotations that will be added to `Deployment`'s `spec.template.metadata.annotations:` | 
| `.service:`          | y | | map to render a `Service` associated with this `Deployment` (more below in [services map](#deployment.service-map)) |
| `.service.selector:` | y | | map of key-value pairs that are added to this `Deployment`'s `spec.template.metadata.labels:` and to the |
| `.labels:`           | y | | map of key-value pairs that are added to this `Deployment`'s `spec.template.metadata.labels:` |
| `.replicaCount:`     | y | | `spec.template.spec.replicaCount:` set to this value |
| `.ports.protocol`    | y | TCP | resulting port array entry set to this value (see [ports array](#ports-array)) |
| `.volumes.releaseRevisionConfigMap` | y | |  special volume type for **Deployment** that means to create a volume with given-name from a ConfigMap with name `{{ ..nameOverride }}-given-name-{{ .serviceFramework.releaseType }}-{{ .Release.Revision }}` |
| `.commonContainers:` | y | | map of commonContainers to include in the `.containers:` map if each commonContainer has a matching `.Values.global.commonContainers:` map entry  (see more below [commonContainers](#commonContainers)) |

#### deployment.service map

Each **service-framework** **deployments:**  map entry can specify a kubernetes **Service** should also be created.  This **Deployment's** containers may associate themselves with this **Service** using .bindToService in their [ports array](#ports-array).  When a container sets .bindToService, this **Service** has a `spec.ports:` array entry corresponding to that container port.  Remember, defined **service** [can be disabled](#ability-to-disable).

| attribute                | optional | default | kubernetes `Service` resource result                              |
| :--------------------------------- | :------: | ------- | -------------------------------------------------------------------- |
| `.service.name` | y | `.nameOverride` | `metadata.name` |
| `.nameOverride` | n | | app: {{ .nameOverride }} added to `spec.selector:` |
| `.service.annotations:` | y | | array of entries added to `metadata.annotations:` |
| `.service.type` | n | | `spec.type` - if **LoadBalancer** the [following](#service-LoadBalancer-annotations) `metadata.annotations:` are also added  |
| `.service.loadBalancerSourceRanges:` | y | | array of entries added to `spec.loadBalancerSourceRanges:` |
| `.service.sessionAffinity` | y | | `spec.sessionAffinity:` |
| `.service.selector:` | y | | map of key-value pairs that are added to `spec.selector:` |
| `.service.port`  | y | .Values.global.standardServicePort | `spec.ports:` .port: value [if container port.bindToService](#container.ports-array) |
| `.service.protocol` | y | TCP | `spec.ports:` .protocol: value [if container port.bindToService](#container.ports-array) |

#### service LoadBalancer annotations

If **deployment.service.type** = **LoadBalancer** annotations are added based on the map entries of **.Values.global.profiles:** and **.Values.global.hostnames:** where each map is indexed using the current **.Values.profile** value.  See [global map documentation](#service-framework-global-values).  *Hostname* is looked up using the following logic:
```
  hostnameMap := index .Values.global.hostnames .Values.profile
  if and hostnameMap.profileOverrides (index hostnameMap.profileOverrides .Values.profile)
    hostname = index hostnameMap.profileOverrides .Values.profile
  else
    hostname = hostnameMap.name

  profileMap := index .Values.global.profiles .Values.profile
  if eq profileMap.profiledHostnames "append"
    hostname = hostname{{ .Values.global.profiles.hostnameSeparator }}{{ .Values.profile }}
  else if eq profileMap.profiledHostnames "prepend"
    hostname = {{ .Values.profile }}{{ .Values.global.profiles.hostnameSeparator }}hostname
  
  if eq profileMap.namespacedHostnames "append"
    hostname = hostname{{ .Values.global.profiles.hostnameSeparator }}{{ .Release.Namespace }}
  else if eq profileMap.namespacedHostnames "prepend"
    hostname = {{ .Release.Namespace }}{{ .Values.global.profiles.hostnameSeparator }}hostname
```

| `metadata.annotations:` item                                    | value                           |
| :-------------------------------------------------------------- | ------------------------------- |
| `dns.alpha.kubernetes.io/external`                              | *hostname*.{{ index (index .Values.profiles .Values.profile) "loadbalancerDomain" }} |
| `service.beta.kubernetes.io/aws-load-balancer-ssl-cert`         | {{ index (index .Values.global.profiles .Values.profile) "sslIAM" }} |
| `service.beta.kubernetes.io/aws-load-balancer-backend-protocol` | "http" |
| `service.beta.kubernetes.io/aws-load-balancer-ssl-ports`        | "443" |


#### deployment.service.ingress map

Each [deployment.service map](#deployment.service-map) can add a **.ingress:** map which creates a kubernetes **Ingress** that is associated to its parent **deployment.service**.  Remember, defined **ingress** [can be disabled](#ability-to-disable).

| attribute                | optional | default | kubernetes `Ingress` resource result                              |
| :--------------------------------- | :------: | ------- | -------------------------------------------------------------------- |
| `.ingress.tlsEnabled` | y | false | if true, add `spec.tls:` rule with host set to public name of parent Service |
| `.ingress.annotations:` | y | | array of entries added to `metadata.annotations:` |
| parent `.service.name` | | | `spec.rules.host.http.paths:` item's `.serviceName` set to this value |
| parent `.service.port` | | `.Values.global.standardServicePort` | `spec.rules.host.http.paths:` item's `.servicePort` set to this value |

## Resulting `Deployment` and `Job` resources

### `Deployment`

| `Deployment` resource attribute               | condition | value                           |
| :-------------------------------------------- | --------- | ------------------------------- |
| `.metadata.labels.chart`          | | `{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}` |
| `.metadata.labels.release`        | | .Release.Name |
| `.metadata.labels.heritage`       | | .Release.Service |
| `.metadata.labels.`               | `.service.selector:` defined and service enabled | `.service.selector:` entries |
| `.spec.strategy.type`             | | `RollingUpdate` |
| `.spec.strategy.rollingUpdate.maxUnavailable` | | `0` |
| `.spec.template.spec.affinity:` | if `.affinity.fromTemplate` defined | include `.affinity.fromTemplate` define name |
| `.spec.template.spec.affinity:` | if `.affinity` defined | `.affinity:` value |
| `.spec.template.spec.affinity:` | otherwise | `.affinity:` set to [default podAntiAffinity](#default-podAntiAffinity) |
| `.spec.template.spec.automountServiceAccountToken:` | if .automountServiceAccountToken set | `.automountServiceAccountToken:` value |
| `.spec.template.spec.automountServiceAccountToken:` | otherwise | `false` |
| `.spec.template.spec.serviceAccountName:` | .automountServiceAccountToken set and .global.rbac true | `{ .nameOverride }-write-secrets-and-configmaps` |
| `.spec.template.spec.imagePullSecrets:` | if `.Values.global.imagePullSecrets` | include yaml contents |
| `.spec.template.spec.imagePullSecrets:` | if deployment `.imagePullSecrets` | include yaml contents |

#### default podAntiAffinity

Each rendered Deployment will get the following default `podAntiAffinity` resource:

```
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: deploymentName
          operator: In
          values: {{ .nameOverride }}-{{ deployment name }}-{{ .releaseType }}
      topologyKey: "failure-domain.beta.kubernetes.io/zone"
  - weight: 90
    podAffinityTerm:
      labelSelector:
        matchExpressions:
        - key: deploymentName
          operator: In
          values:
          - "{{ .nameOverride }}-{{ deployment name }}-{{ .releaseType }}"
      topologyKey: "kubernetes.io/hostname"
```


### `Job`

| `Job` resource attribute          | value                        |
| :-------------------------------- | ------------------------------- |
| `.metadata.labels.chart`          | `{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}` |
| `.metadata.labels.release`        | .Release.Name |
| `.metadata.labels.heritage`       | .Release.Service |

note: more to go.  only at line 28. 



--------

To simplify re-using a Helm chart in different "profiles" (or "environments" - like "qa" or "prod), the chart's directory structure can be supplemented with a concept of profile-specific overrides like the following:

todo diagrams: directories something like below

```
your-chart/Chart.yaml               your-chart/Chart.yaml
          /values.yaml                        /values.yaml
          /templates/*.yaml                   /templates/*.yaml
                                              /profiles/
                                                        profile-a/values.yaml
                                                        profile-b/values.yaml

helm install your-chart             helm install your-chart -f your-chart/profiles/profile-a/values.yaml
```

--------

## service framework global values

The **service-framework** uses global maps to customize rendered templates based on  [.Values.profile](#profile) and [.Values.nameOverride](*nameOverride), which all parent charts must define to use **service-framework** as a requirement.

`service-framework` values:

| .Values         | optional |    description       |
| :-------------- | :------: | :------------------- |
| `.profile`      | n | Name of the "current profile".  A `.Values.global.profiles:` map of the same name must be defined which specifies which profile-specific map to use for rendering.  The recommended pattern is to have profile in the base values.yaml have value of "none" or "base" - and to override .Values.profile in each profile-specific directory with that directory name. |
| `.nameOverride` | n | Name of each "parent chart".  This name must be unique.  A `.Values.global.hostnames:` map of the same name must be defined which specifies information used to generate the **hostname** for this parent chart.  This hostname-generation may also be customized by `.Values.profile` |

### profile

The **service-framework** was created to simplify and commonize supporting a collection of related Helm charts that follow such a "profile" model.  Remember, the current profile is set by each parent chart at .Values.profile.

`.Values.global.profiles:` is a map of global profile information.

| .Values.profiles.      | optional |    description       |
| :--------------------- | :------: | :------------------- |
| hostnameSeparator | y | string to use to separate base hostname from namespace or profile, if enabled by the current profile |
| namespaceSeparator | y | string to use to separate base namespace from profile, if enabled by the current profile and profile-specific information |
| profilename: | y | map of profile-specific attributes for profile "profilename" |

`.Values.global.profiles.`profilename: is a map of profile-specific information.  E.g., .Value.profile = `prod` selects the profile defined in the following map:
 
| .Values.global.profiles.prod:   | optional |    description       |
| :------------------------------ | :------: | :------------------- |
| publicDomain | n | domain of public Ingress |
| loadbalancerDomain | y | for Services of type LoadBalancer, use this for its domain.  (in that case, required) |
| sslIAM | y | for Services of type LoadBalancer, use this for its ssl-cert.  (in that case, required) |
| namespaceHostnames | y | `prepend` or `append` the Helm `.Release.Namespace` to base hostname (from hostnames map) |
| profileHostnames | y | `prepend` or `append`  `.Values.profile` to base hostname (from hostnames map) |
| profileNamespaces | y | `prepend` or `append`  `.Values.profile` to namespace (from namespaces map) |
  
### hostnames

`.Values.global.hostnames:` is a map of hostname information for each supported parent chart `.Values.nameOverride` value.  E.g., for .Values.nameOverride = `someService`

| .Values.global.hostnames.someService:  | optional |    description       |
| :------------------------- | :------: | :------------------- |
| name              | n | base hostname for this nameOverride |
| envVar            | y | whether an env var containing the fqdn of this hostname should be added to all containers.  env var names are GLOBAL_FQDN_{{ $nameOverride | upper | replace "-" "_" }} |
| profileOverrides: | y | map of profileHostname and namespaceHostname overrides for this hostname, and optionally profile-specific name overrides |
| profileOverrides.profilename | y | when {{ eq .Values.profile "profilename" }}, use this value to override the base hostname `name` (above) |
| profileOverrides.disableProfileHostnames | y | whether to disable the .Values.global.profiles.{{ .Values.profile }}.profileHostnames |
| profileOverrides.disableNamespaceHostnames | y | whether to disable the .Values.global.profiles.{{ .Values.profile }}.namespaceHostnames |


### namespaces

`.Values.global.namespaces:` map - TBD

### standardServicePurt

`.Values.global.standardServicePort:` is a port number that will be used by Service spec.ports.port numbers if deployment.service.ports.port is not specified. 

### imagePullSecrets

`.Values.global.imagePullSecrets:` is an optional array of imagePullSecrets that will be added to every Deployment or Job's spec.template.spec.imagePullSecrets.

### global commonContainers map

`.Values.global.commonContainers:` is a map of container definitions that may be included by Deployments or Jobs.



-----

Also, describe

## Ingress

## PV, PVC

## PDB

## ServiceAccount, Role, RoleBinding

## Others??
