Kustomize
===

<!-- TOC -->

- [1. Installation](#1-installation)
- [2. Hello world](#2-hello-world)
    - [2.1. Establish the base](#21-establish-the-base)
    - [2.2. Create Overlays](#22-create-overlays)
        - [2.2.1. Staging Kustomization](#221-staging-kustomization)
        - [2.2.2. Staging Patch](#222-staging-patch)
        - [2.2.3. Production Kustomization](#223-production-kustomization)
        - [2.2.4. Production Patch](#224-production-patch)
    - [2.3. Deploy](#23-deploy)
- [3. Demo: SpringBoot](#3-demo-springboot)
    - [3.1. Download resources](#31-download-resources)
    - [3.2. Initialize kustomization.yaml](#32-initialize-kustomizationyaml)
    - [3.3. Add the resources](#33-add-the-resources)
    - [3.4. Add configMap generator](#34-add-configmap-generator)
    - [3.5. Customize configMap](#35-customize-configmap)
    - [3.6. Name Customization](#36-name-customization)
    - [3.7. Label Customization](#37-label-customization)
    - [3.8. Download Patch for JVM memory](#38-download-patch-for-jvm-memory)
    - [3.9. Download Patch for health check](#39-download-patch-for-health-check)
    - [3.10. Add patches](#310-add-patches)

<!-- /TOC -->

kustomize是sig-cli的一个子项目，它的设计目的是给kubernetes的用户提供一种可以重复使用同一套配置的声明式应用管理，从而在配置工作中用户只需要管理和维护kubernetes的API对象，而不需要学习或安装其它的配置管理工具，也不需要通过复制粘贴来得到新的环境的配置。

> kustomize lets you customize raw, template-free YAML files for multiple purposes, leaving the original YAML untouched and usable as is.  
kustomize targets kubernetes; it understands and can patch kubernetes style API objects. It's like make, in that what it does is declared in a file, and it's like sed, in that it emits editted text.  
This tool is sponsored by sig-cli (KEP), and inspired by DAM.

当我们运行一个kubernetes环境的时候，我们需要一些含有API对象的YAML文件，这些文件中规定了要部署什么样的应用，需要多少份副本，开辟多大的存储空间，分配多少内存和CPU等信息。通过修改这些YAML文件的内容我们可以对这些信息进行相应的改动，比如我们需要增加一个副本，就需要修改对应YAML文件的replica数值；如果我们需要部署最新版本的docker镜像，就需要修改对应YAML文件中的docker镜像的版本或标签。我们可以把所有这些为了满足需求而进行的修改成为自定义kubernetes的配置。

在我们开发一个微服务架构的应用的过程中，我们会创建一些YAML文件来部署一个开发的环境，在这个环境下，我们需要进行各种测试。一旦所有的测试达到了我们的预期，我们就会把这个应用部署到生产环境。因为我们已经有了一套开发环境的配置，我们可以通过复制这些配置，再进行生产环境下的自定义，就可以得倒一套用于生产环境的配置。

然而这种方法的可扩展性并不好，当我们的微服务数量很多或者环境数量很多时，我们就有许多套的配置，这些配置只有细微的差别，而在很大程度上都一样。当我们对配置进行改动或者升级的时候，就非常容易漏掉一些改动或者实施了额外的甚至是错误的改动。我们来看两个简单的例子。

假如我们有一个应用，我们已经把它部署在十个不同的环境中，每一个环境的配置都是通过复制,粘贴和修改得到的。每个环境中所运行的应用版本号都是1.0，现在我们想把所有环境中的版本号升级到2.0。我们就需要修改对应每一个环境下该应用的版本号。这不是一个复杂的改动，我们可以对每一个环境依次进行修改，所有的改动很快就可以完成。但是在这个过程中我们很有可能只修改了其中的九个环境而漏掉了一个；我们也很有可能把其中一个版本号改成2,0。

我们再来看另外一个例子。在小王通过复制，粘贴和修改所得到的一个配置中，小李是没有办法区分哪些值是保持不变，而哪些值是被修改过的。当小李想要修改这个新的配置时，他不知道哪些可以安全地改动而那些可能会影响的当前环境的正确运行。

kustomize允许用户将不同环境所共享的配置放在一个文件目录下，而将其他不同的配置放在另外的目录下。这样用户就可以很容易的区分那些值是当前环境所特有的，从而在修改的时候会额外关注。kustomize可以非常好地解决这些问题。

# 1. Installation

Kustomize 已经内置于 kubectl v1.14 。
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.14.0/bin/linux/amd64/kubectl
cp kubectl /usr/local/bin/kubectl
chmod +x /usr/local/bin/kubectl
```


# 2. Hello world

步骤：

1. Clone an existing configuration as a base.
2. Customize it.
3. Create two different overlays (staging and production) from the customized base.
4. Run kustomize and kubectl to deploy staging and production.

首先创建一个应用程序的目录，比如：
```
mkdir -p ./hello
```

## 2.1. Establish the base

```bash
mkdir -p ./hello/base
```

```yaml
cat << EOF > ./hello/base/kustomization.yaml
commonLabels:
  app: hello

resources:
- deployment.yaml
- service.yaml
- configMap.yaml
EOF

cat << EOF > ./hello/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: the-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        deployment: hello
    spec:
      containers:
      - name: the-container
        image: monopole/hello:1
        command: ["/hello",
                  "--port=8080",
                  "--enableRiskyFeature=$(ENABLE_RISKY)"]
        ports:
        - containerPort: 8080
        env:
        - name: ALT_GREETING
          valueFrom:
            configMapKeyRef:
              name: the-map
              key: altGreeting
        - name: ENABLE_RISKY
          valueFrom:
            configMapKeyRef:
              name: the-map
              key: enableRisky
EOF

cat << EOF > ./hello/base/service.yaml
kind: Service
apiVersion: v1
metadata:
  name: the-service
spec:
  selector:
    deployment: hello
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 8666
    targetPort: 8080
EOF

cat << EOF > ./hello/base/configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: the-map
data:
  altGreeting: "Good Morning!"
  enableRisky: "false"
EOF
```

文件结构：

```bash
tree hello
hello
└── base
    ├── configMap.yaml
    ├── deployment.yaml
    ├── kustomization.yaml
    └── service.yaml
```

可以通过下面的命令将程序应用到集群：
```
kubectl apply -k ./hello/base
```

可以通过`kubectl kustomize ./hello/base`命令将上面的资源合成到一个里面
```yaml
apiVersion: v1
data:
  altGreeting: Good Morning!
  enableRisky: "false"
kind: ConfigMap
metadata:
  labels:
    app: hello
  name: the-map
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hello
  name: the-service
spec:
  ports:
  - port: 8666
    protocol: TCP
    targetPort: 8080
  selector:
    app: hello
    deployment: hello
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello
  name: the-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
        deployment: hello
    spec:
      containers:
      - command:
        - /hello
        - --port=8080
        - --enableRiskyFeature=$(ENABLE_RISKY)
        env:
        - name: ALT_GREETING
          valueFrom:
            configMapKeyRef:
              key: altGreeting
              name: the-map
        - name: ENABLE_RISKY
          valueFrom:
            configMapKeyRef:
              key: enableRisky
              name: the-map
        image: monopole/hello:1
        name: the-container
        ports:
        - containerPort: 8080
```

## 2.2. Create Overlays

创建开发和生产得叠加层：

```
mkdir -p ./hello/overlays/staging
mkdir -p ./hello/overlays/production
```

### 2.2.1. Staging Kustomization

在 `staging` 目录中，进行定制新名称前缀和一些不同标签的自定义。

```yaml
cat << EOF > ./hello/overlays/staging/kustomization.yaml
namePrefix: staging-
commonLabels:
  variant: staging
  org: acmeCorporation
commonAnnotations:
  note: Hello, I am staging!
bases:
- ../../base
patchesStrategicMerge:
- map.yaml
EOF
```

### 2.2.2. Staging Patch

通过添加一个 `configMap` 来修改 Good Morning 服务得问候语：to Have a pineapple!

```yaml
cat << EOF > ./hello/overlays/staging/map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: the-map
data:
  altGreeting: "Have a pineapple!"
  enableRisky: "true"
EOF
```

### 2.2.3. Production Kustomization

在 `production` 目录中，使用不同的名称前缀和标签进行自定义。

```yaml
cat << EOF > ./hello/overlays/production/kustomization.yaml
namePrefix: production-
commonLabels:
  variant: production
  org: acmeCorporation
commonAnnotations:
  note: Hello, I am production!
bases:
- ../../base
patchesStrategicMerge:
- deployment.yaml
EOF
```

### 2.2.4. Production Patch

制作一个增加副本数量的生产补丁（因为生产需要更多流量）。

```yaml
cat << EOF > ./hello/overlays/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: the-deployment
spec:
  replicas: 10
EOF
```


再看一下文件结构：

```bash
tree hello
hello
├── base
│   ├── configMap.yaml
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── production
    │   ├── deployment.yaml
    │   └── kustomization.yaml
    └── staging
        ├── kustomization.yaml
        └── map.yaml
```

## 2.3. Deploy

采用下面得命令来部署不同场景下的系统

```
kubectl apply -k ./hello/overlays/staging
```
```
kubectl apply -k ./hello/overlays/production
```

通过`kubectl kustomize ./hello/overlays/staging`再次查看一下资源：
```yaml
apiVersion: v1
data:
  altGreeting: Have a pineapple!
  enableRisky: "true"
kind: ConfigMap
metadata:
  annotations:
    note: Hello, I am staging!
  labels:
    app: hello
    org: acmeCorporation
    variant: staging
  name: staging-the-map
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    note: Hello, I am staging!
  labels:
    app: hello
    org: acmeCorporation
    variant: staging
  name: staging-the-service
spec:
  ports:
  - port: 8666
    protocol: TCP
    targetPort: 8080
  selector:
    app: hello
    deployment: hello
    org: acmeCorporation
    variant: staging
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    note: Hello, I am staging!
  labels:
    app: hello
    org: acmeCorporation
    variant: staging
  name: staging-the-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
      org: acmeCorporation
      variant: staging
  template:
    metadata:
      annotations:
        note: Hello, I am staging!
      labels:
        app: hello
        deployment: hello
        org: acmeCorporation
        variant: staging
    spec:
      containers:
      - command:
        - /hello
        - --port=8080
        - --enableRiskyFeature=
        env:
        - name: ALT_GREETING
          valueFrom:
            configMapKeyRef:
              key: altGreeting
              name: staging-the-map
        - name: ENABLE_RISKY
          valueFrom:
            configMapKeyRef:
              key: enableRisky
              name: staging-the-map
        image: monopole/hello:1
        name: the-container
        ports:
        - containerPort: 8080
```

可以看出，新的资源是一个包含了变动的完整的资源。
---

# 3. Demo: SpringBoot

In this tutorial, you will learn - how to use `kustomize` to customize a basic Spring Boot application's
k8s configuration for production use cases.

In the production environment we want to customize the following:

- add application specific configuration for this Spring Boot application
- configure prod DB access configuration
- resource names to be prefixed by 'prod-'.
- resources to have 'env: prod' labels.
- JVM memory to be properly set.
- health check and readiness check.

First make a place to work:
<!-- @makeDemoHome @test -->
```
DEMO_HOME=$(mktemp -d)
```

## 3.1. Download resources

To keep this document shorter, the base resources
needed to run springboot on a k8s cluster are off in a
supplemental data directory rather than declared here
as HERE documents.

Download them:

<!-- @downloadResources @test -->
```
CONTENT="https://raw.githubusercontent.com\
/kubernetes-sigs/kustomize\
/master/examples/springboot"

curl -s -o "$DEMO_HOME/#1.yaml" \
  "$CONTENT/base/{deployment,service}.yaml"
```

## 3.2. Initialize kustomization.yaml

The `kustomize` program gets its instructions from
a file called `kustomization.yaml`.

Start this file:

<!-- @kustomizeYaml @test -->
```
touch $DEMO_HOME/kustomization.yaml
```

## 3.3. Add the resources

<!-- @addResources @test -->
```
cd $DEMO_HOME

kustomize edit add resource service.yaml
kustomize edit add resource deployment.yaml

cat kustomization.yaml
```

`kustomization.yaml`'s resources section should contain:

> ```
> resources:
> - service.yaml
> - deployment.yaml
> ```

## 3.4. Add configMap generator

<!-- @addConfigMap @test -->
```
echo "app.name=Kustomize Demo" >$DEMO_HOME/application.properties

kustomize edit add configmap demo-configmap \
  --from-file application.properties

cat kustomization.yaml
```

`kustomization.yaml`'s configMapGenerator section should contain:

> ```
> configMapGenerator:
> - files:
>   - application.properties
>   name: demo-configmap
> ```

## 3.5. Customize configMap

We want to add database credentials for the prod environment. In general, these credentials can be put into the file `application.properties`.
However, for some cases, we want to keep the credentials in a different file and keep application specific configs in `application.properties`.
 With this clear separation, the credentials and application specific things can be managed and maintained flexibly by different teams.
For example, application developers only tune the application configs in `application.properties` and operation teams or SREs
only care about the credentials.

For Spring Boot application, we can set an active profile through the environment variable `spring.profiles.active`. Then
the application will pick up an extra `application-<profile>.properties` file. With this, we can customize the configMap in two
steps. Add an environment variable through the patch and add a file to the configMap.

<!-- @customizeConfigMap @test -->
```
cat <<EOF >$DEMO_HOME/patch.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: sbdemo
spec:
  template:
    spec:
      containers:
        - name: sbdemo
          env:
          - name: spring.profiles.active
            value: prod
EOF

kustomize edit add patch patch.yaml

cat <<EOF >$DEMO_HOME/application-prod.properties
spring.jpa.hibernate.ddl-auto=update
spring.datasource.url=jdbc:mysql://<prod_database_host>:3306/db_example
spring.datasource.username=root
spring.datasource.password=admin
EOF

kustomize edit add configmap \
  demo-configmap --from-file application-prod.properties

cat kustomization.yaml
```

`kustomization.yaml`'s configMapGenerator section should contain:
> ```
> configMapGenerator:
> - files:
>   - application.properties
>   - application-prod.properties
>   name: demo-configmap
> ```

## 3.6. Name Customization

Arrange for the resources to begin with prefix
_prod-_ (since they are meant for the _production_
environment):

<!-- @customizeLabel @test -->
```
cd $DEMO_HOME
kustomize edit set nameprefix 'prod-'
```

`kustomization.yaml` should have updated value of namePrefix field:

> ```
> namePrefix: prod-
> ```

This `namePrefix` directive adds _prod-_ to all
resource names, as can be seen by building the
resources:

<!-- @build1 @test -->
```
kustomize build $DEMO_HOME | grep prod-
```

## 3.7. Label Customization

We want resources in production environment to have
certain labels so that we can query them by label
selector.

`kustomize` does not have `edit set label` command to
add a label, but one can always edit
`kustomization.yaml` directly:

<!-- @customizeLabels @test -->
```
cat <<EOF >>$DEMO_HOME/kustomization.yaml
commonLabels:
  env: prod
EOF
```

Confirm that the resources now all have names prefixed
by `prod-` and the label tuple `env:prod`:

<!-- @build2 @test -->
```
kustomize build $DEMO_HOME | grep -C 3 env
```

## 3.8. Download Patch for JVM memory

When a Spring Boot application is deployed in a k8s cluster, the JVM is running inside a container. We want to set memory limit for the container and make sure
the JVM is aware of that limit. In K8s deployment, we can set the resource limits for containers and inject these limits to
some environment variables by downward API. When the container starts to run, it can pick up the environment variables and
set JVM options accordingly.

Download the patch `memorylimit_patch.yaml`. It contains the memory limits setup.

<!-- @downloadPatch @test -->
```
curl -s  -o "$DEMO_HOME/#1.yaml" \
  "$CONTENT/overlays/production/{memorylimit_patch}.yaml"

cat $DEMO_HOME/memorylimit_patch.yaml
```

The output contains

> ```
> apiVersion: apps/v1beta2
> kind: Deployment
> metadata:
>   name: sbdemo
> spec:
>   template:
>     spec:
>       containers:
>         - name: sbdemo
>           resources:
>             limits:
>               memory: 1250Mi
>             requests:
>               memory: 1250Mi
>           env:
>           - name: MEM_TOTAL_MB
>             valueFrom:
>               resourceFieldRef:
>                 resource: limits.memory
> ```

## 3.9. Download Patch for health check
We also want to add liveness check and readiness check in the production environment. Spring Boot application
has end points such as `/actuator/health` for this. We can customize the k8s deployment resource to talk to Spring Boot end point.

Download the patch `healthcheck_patch.yaml`. It contains the liveness probes and readyness probes.

<!-- @downloadPatch @test -->
```
curl -s  -o "$DEMO_HOME/#1.yaml" \
  "$CONTENT/overlays/production/{healthcheck_patch}.yaml"

cat $DEMO_HOME/healthcheck_patch.yaml
```

The output contains

> ```
> apiVersion: apps/v1beta2
> kind: Deployment
> metadata:
>   name: sbdemo
> spec:
>   template:
>     spec:
>       containers:
>         - name: sbdemo
>           livenessProbe:
>             httpGet:
>               path: /actuator/health
>               port: 8080
>             initialDelaySeconds: 10
>             periodSeconds: 3
>           readinessProbe:
>             initialDelaySeconds: 20
>             periodSeconds: 10
>             httpGet:
>               path: /actuator/info
>               port: 8080
> ```

## 3.10. Add patches

Add these patches to the kustomization:

<!-- @addPatch @test -->
```
cd $DEMO_HOME
kustomize edit add patch memorylimit_patch.yaml
kustomize edit add patch healthcheck_patch.yaml
```

`kustomization.yaml` should have patches field:

> ```
> patches:
> - patch.yaml
> - memorylimit_patch.yaml
> - healthcheck_patch.yaml
> ```

The output of the following command can now be applied
to the cluster (i.e. piped to `kubectl apply`) to
create the production environment.

<!-- @finalBuild @test -->
```
kustomize build $DEMO_HOME  # | kubectl apply -f -
```


