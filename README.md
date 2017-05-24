## New Relic Server Monitoring Agent Example

This example shows how to run a New Relic server monitoring agent as a pod in a DaemonSet on an existing Openshift cluster.

This example will create a DaemonSet which places the New Relic monitoring agent on every node in the cluster. It's also fairly trivial to exclude specific Openshift nodes from the DaemonSet to just monitor specific servers.

### Step 0: Prerequisites

This process will create privileged containers which have full access to the host system for logging. Beware of the security implications of this.

### Step 1: Configure New Relic Agent

The New Relic agent is configured via environment variables. We will configure these environment variables in a sourced bash script, encode the environment file data, and store it in a secret which will be loaded at container runtime.

The [New Relic Linux Server configuration page]
(https://docs.newrelic.com/docs/servers/new-relic-servers-linux/installation-configuration/configuring-servers-linux) lists all the other settings for nrsysmond.

To create an environment variable for a setting, prepend NRSYSMOND_ to its name. For example,

```console
loglevel=debug
```

translates to

```console
NRSYSMOND_loglevel=debug
```

Edit examples/newrelic/nrconfig.env and set up the environment variables for your NewRelic agent. Be sure to edit the license key field and fill in your own New Relic license key.

Now, let's vendor the config into a secret.

```console
$ ./config-to-secret.sh
```

<!-- BEGIN MUNGE: EXAMPLE newrelic-config-template.yaml -->

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: newrelic-config
type: Opaque
data:
  config: {{config_data}}
```

[Download example](newrelic-config-template.yaml?raw=true)
<!-- END MUNGE: EXAMPLE newrelic-config-template.yaml -->

The script will encode the config file and write it to `newrelic-config.yaml`.

Finally, submit the config to the cluster:

```console
$ kubectl create -f examples/newrelic/newrelic-config.yaml
```

### Step 2: Create service account and grant access to SCC

```console
oc create serviceaccount logging-newrelic
#add account to privileged SCC 
oc adm policy add-scc-to-user privileged -z logging-newrelic
```
oc create -f /tmp/newrelic/newrelic-daemonset.yaml 

### Step 3: Label nodes

We need to label all nodes, where we want this to run. We did this via ansible. You can do it via `oc` command

`oc label node <nodeFQDN> logging-infra-newrelic='true'`


### Step 4: Create the DaemonSet definition.

The DaemonSet definition instructs Kubernetes to place a newrelic sysmond agent on each Kubernetes node.

<!-- BEGIN MUNGE: EXAMPLE newrelic-daemonset.yaml -->

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: newrelic-agent
  labels:
    provider: openshift
    component: newrelic
    logging-infra: newrelic
spec:
  selector:
    matchLabels:
      provider: openshift
      component: newrelic
      logging-infra: newrelic
  template:
    metadata:
      labels:
        provider: openshift
        component: newrelic
        logging-infra: newrelic
    spec:
      nodeSelector:
        logging-infra-newrelic: "true"
      serviceAccount: logging-newrelic
      serviceAccountName: logging-newrelic
      hostPID: true
      hostIPC: true
      hostNetwork: true
      containers:
        - resources:
            requests:
              cpu: 0.15
          securityContext:
            privileged: true
          env:
            - name: NRSYSMOND_logfile
              value: "/var/log/nrsysmond.log"
          image: newrelic/nrsysmond
          name: newrelic
          command: [ "bash", "-c", "source /etc/kube-newrelic/config && /usr/sbin/nrsysmond -E -F" ]
          volumeMounts:
            - name: newrelic-config
              mountPath: /etc/kube-newrelic
              readOnly: true
            - name: dev
              mountPath: /dev
            - name: run
              mountPath: /var/run/docker.sock
            - name: sys
              mountPath: /sys
            - name: log
              mountPath: /var/log
      volumes:
        - name: newrelic-config
          secret:
            secretName: newrelic-config
        - name: dev
          hostPath:
              path: /dev
        - name: run
          hostPath:
              path: /var/run/docker.sock
        - name: sys
          hostPath:
              path: /sys
        - name: log
          hostPath:
              path: /var/log
```

[Download example](newrelic-daemonset.yaml?raw=true)
<!-- END MUNGE: EXAMPLE newrelic-daemonset.yaml -->

The daemonset instructs Openshift to spawn pods on each node, mapping /dev/, /run/, /sys/, and /var/log to the container. It also maps the secrets we set up earlier to /etc/kube-newrelic/config, and sources them in the startup script, configuring the agent properly.
