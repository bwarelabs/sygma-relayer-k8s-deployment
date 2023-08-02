# sygma-relayer-k8s-deployment
This guide is a step-by-step manual that will help you deploy Sygma Relayer and join the Sygma Bridge MPC consensus.
*Note: This process is not permissionless and should be agreed upon with the Sygma team in advance. For more information please visit [website](https://buildwithsygma.com/)*

This deployment guide is based on assumptions that the user will use AWS as an infrastructure provider and will use Helm Chart to spin an the relayers.

## Prerequisites

### Prepare configuration parameters
1. Prepare a Private key that will be used to send execution transactions on the destination network.
**Note #1: you can use one private key for different domains).**
**Note #2: Never use this private key for something else and keep it secret**
***Note #3: Do not forget to top up the balance of the sender key and set some periodic checks of the balance***

The current TestNet operation requires private keys for 3 EVM networks: Goerli, Sepolia and Rhala whiele the current Mainnet requires keys for 3 EVM networks: Ethereum, Khala and Phala (**as it states for now PLEASE CONFIRM THIS WITH SYGMA TEAM BEFOREHAND**).

2. For each network, you should have an RPC provider. Relaying partners must align with the Sygma team on the specific clients or RPC providers to use, so we can ensure appropriate provider redundancy in order to increase network robustness and reliability.


## Deployment environment

### Helm chart

**#1: Install Helm**
 In order to intall Helm please follow this guide or any other guide that you trust [Install Helm](https://phoenixnap.com/kb/install-helm)

**#2: Create Helm Chart**

To create a new Helm chart, use:

      helm create \<chart name\>

For example:
    
      helm create sygmarelayer

After creation you should have a directory containing the following files and directories:
  * directory charts - Used for adding dependent charts. Empty by default.
  * directory templates - Configuration files that deploy in the cluster.
  * Chart.yaml file - Used to describe de project
  * values.yaml file - Used basically to store the files data that will be in templates directory

#### values.yaml file - this is a customable file, you can play with the names, values, order and indentation
```yaml

image: # containers related data
  * sygma_relayer:
    * registry: ghcr.io/sygmaprotocol/sygma-relayer
    * tag: v1.8.1
    * pullPolicy: Always
  * otel:
    * registry: your_own_registry
    * tag: you_own_tag 
    * pullPolicy: Always
    
node: # project related data
  * name: sygma-relayer
  * partof: chainsafe


pvc: # in order to deploy a relayer we will need a persistent volume - the following lines are just examples for a PVC, you can choose whatever you want 
  storageClass: "ssd-standard-gp3"
  accessModes: ReadWriteOnce
  storage: 10Gi


CustomValues: # used to store any other variables that you'll use inside your template files
  dd_service: "TESTNET-relayers-\<relayer-id\>"
  syg_relayer_loglevel: "debug"
  dd_apm_enabled: "true"
  dd_apm_non_local_traffic: "true"
  dd_tags: "env:TESTNET,project:chainbridge,relayerid:\<relayer-id\>"
  dd_log_level: "INFO"
  env: "TESTNET"
  relayer_id : ""
```

#### configmap_relayer.yaml file - located in templates directory; this file is used to store the relayer related specific parameters

# !!! all of these variables values should be stored in SSM considering that they are sensitive; in order to add an extra layer of security you should replace the configmap file with a secret file, in which, every value is 64base encoded !!! 

```Yaml

apiVersion: v1

kind: ConfigMap

metadata:
  name: sygma-relayer-configmap

data: 
  SYG_RELAYER_MPCCONFIG_KEY: "generated_below_using_cli"
  SYG_RELAYER_MPCCONFIG_KEYSHAREPATH: "/mount/r{{ .Values.CustomValues.relayer_id }}.keyshare"
  SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_ENCRYPTIONKEY: "provided_by_sygma_team"
  SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_PATH: "/mount/r{{ .Values.CustomValues.relayer_id }}-top.json"
  SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_URL: "provided_by_sygma_team"
  SYG_RELAYER_OPENTELEMETRYCOLLECTORURL: "http://localhost:4318"
  SYG_RELAYER_ID: "{{ .Values.CustomValues.relayer_id }}" -> in this way you can access values stored in values.yaml file
  SYG_RELAYER_ENV: "TESTNET"
  SYG_CHAINS: \<you can see how this should look below\>
```


 #### persistent_volume_claim.yaml file - located in templates directory; this file is used to configure a volume that will be used to store data

```Yaml

apiVersion: v1

kind: PersistentVolumeClaim

metadata:
  name: {{ .Values.node.name }}-pvc
  labels:
    app.kubernetes.io/instance: {{ .Values.node.name }}
    app.kubernetes.io/name: {{ .Values.node.name }}
    app.kubernetes.io/part-of: {{ .Values.node.partof }}

spec:
  accessModes:
    - {{ .Values.pvc.accessModes }}
  resources:
    requests:
      storage: {{ .Values.pvc.storage }}
  storageClassName: {{ .Values.pvc.storageClass }}

```

 #### deployment.yaml file - located in templates directory;

```Yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.node.name }}
  labels:
    app.kubernetes.io/instance: {{ .Values.node.name }}
    app.kubernetes.io/name: {{ .Values.node.name }}
    app.kubernetes.io/part-of: {{ .Values.node.partof }}
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app.kubernetes.io/instance: {{ .Values.node.name }}
      app.kubernetes.io/name: {{ .Values.node.name }}
      app.kubernetes.io/part-of: {{ .Values.node.partof }}
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/instance: {{ .Values.node.name }}
        app.kubernetes.io/name: {{ .Values.node.name }}
        app.kubernetes.io/part-of: {{ .Values.node.partof }}

    spec:
      volumes:
        - name: mount
          persistentVolumeClaim:
            claimName: {{ .Values.node.name }}-pvc 
      containers:
        - name: sygma-relayer
          image: {{ index  .Values.image "sygma_relayer" "registry" }}:{{ index  .Values.image "sygma_relayer" "tag" }}
          imagePullPolicy: {{ index  .Values.image "sygma_relayer" "pullPolicy" }}
          ports:
            - name: port1
              containerPort: 9000
            - name: port2
              containerPort: 9001
          env:
            - name: DD_SERVICE
              value: {{ .Values.CustomValues.dd_service }}
            - name: SYG_RELAYER_LOGLEVEL
              value: {{ .Values.CustomValues.syg_relayer_loglevel }}
            - name: SYG_RELAYER_MPCCONFIG_KEY
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_MPCCONFIG_KEY
            - name: SYG_RELAYER_MPCCONFIG_KEYSHAREPATH
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_MPCCONFIG_KEYSHAREPATH
            - name: SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_ENCRYPTIONKEY
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_ENCRYPTIONKEY
            - name: SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_PATH
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_PATH
            - name: SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_URL
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_URL
            - name: SYG_CHAINS
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_CHAINS 
            - name: SYG_RELAYER_OPENTELEMETRYCOLLECTORURL
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_OPENTELEMETRYCOLLECTORURL
            - name: SYG_RELAYER_ID
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_ID
            - name: SYG_RELAYER_ENV
              valueFrom:
                configMapKeyRef:
                  name: sygma-relayer-configmap
                  key: SYG_RELAYER_ENV    
          command:
            - /bin/bash
            - '-c'
          args: 
            - /bridge run --config=env --config-url=https://config-server.test.buildwithsygma.com/share/ --name=R${SYG_RELAYER_ID} --blockstore=/mount/relayer${SYG_RELAYER_ID}/lvldbdata
          volumeMounts:
            - name: mount
              mountPath: /mount

              
        - name: otel-collector # used to export you relayer metrics towards datadog server managed by Sygma Team
          image: {{ index  .Values.image "otel" "registry" }}:{{ index  .Values.image "otel" "tag" }}
          imagePullPolicy: {{ index  .Values.image "otel" "pullPolicy" }}
          env:
            - name: DD_API_KEY
              value: "stored_in_ssm" 

---
apiVersion: v1
kind: Service
metadata:
  name: sygma-relayer
  labels:
    chart: {{ template "app.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: '*'
    service.beta.kubernetes.io/aws-load-balancer-type: nlb  
spec:
  type: LoadBalancer # here you can choose whatever method you want to export your relayer as dns
  selector:
    app.kubernetes.io/name: {{ .Values.node.name }}
  ports:
    - name: port1
      protocol: TCP
      port: 9000
      targetPort: 9000
    - name: port2
      protocol: TCP
      port: 9001
      targetPort: 9001
```



### Relayer configuration

We use SSM to hold all of our secrets. To access it, go to your AWS Account, in the Search bar type `SSM`. Go into the Systems Manager Service. On the left side menu, go into Parameter Store.
Now you can create any secrets that you want, and then reference it in the `secrets` section in the Task Definition.

During the infrastructure, provisioning terraforms scripts will create a number of secret parameters in the [Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html). You should manually set this parameter according to the following description

- **SYG_CHAINS-** domain configuration. One configuration for all domains (networks). **JUST AN EXAMPLE CONFIRM LIST OF NETWORKS WITH SYGMA TEAM**

    ```json
   [{
     "id": 1,
     "name": "goerli",
     "type": "evm",
     "key": "", 
     "endpoint": "",
     "maxGasPrice": 1000000000000,
     "gasMultiplier": 1.5,
     "latest": true
     },
     {
     "id": 2,
     "name": "sepolia",
     "type": "evm",
     "key": "", 
     "endpoint": "",
     "startBlock": 3518450, 
     "fresh": true,
     "latest": true 
     },
     {
     "id": 3,
     "name": "rhala",
     "type": "substrate",
     "key": "",
     "endpoint": "wss://subbridge-test.phala.network/rhala/ws",
     "latest": false
     }
     ]
    ```

- **SYG_RELAYER_MPCCONFIG_KEY -** secret libp2p key

    *This key is used to secure libp2p communication between relayers. The format of the key is* RSA 2048 with base64 protobuf encoding.

    It is possible to generate a new key by using the CLI command `peer gen-key` from the [relayer repository](https://github.com/sygmaprotocol/sygma-relayer).
    This will generate an output LibP2P peer identity and LibP2P private key, you will need both. But for this param use LibP2P private key.
    **Note: keep private key secret**

    If you generate the key by yourself, you can find out complementary peer identity by running `peer info --private-key=<libp2p_identity_priv_key>`  This identity you will need later.

- **SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_PATH**

    Example: `/mount/r1-top.json` Should be unique per relayer. (eg /mount/r1-top.json, /mount/r2-top.json, etc)

- **SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_ENCRYPTIONKEY**

    AES secret key that is used to encrypt libp2p network topology.
    In order to obrain this secret key you need to fetch it from Sygma AWS account using AWS secrest sharing [Read this](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_examples_cross.html) and [This](https://repost.aws/knowledge-center/secrets-manager-share-between-accounts)
    Generally you would need to share some of your roles ID and we will provide and access to it in our AWS account. For details proactivelly contact Sygma team.

Note:


- **SYG_RELAYER_MPCCONFIG_TOPOLOGYCONFIGURATION_URL**

    URL to fetch topology file from.  The Sygma team will provide you with this key.

- **SYG_RELAYER_MPCCONFIG_KEYSHAREPATH**

    Example: `/mount/r1.keyshare` - path to the file that will contain MPC private key share. Should be unique per relayer. (eg /mount/r1.keyshare, /mount/r2.keyshare, /mount/r3.keyshare, etc)


### Launching a relayer to existing MPC set

After all the configuration parameters above are set we need to add your Relayer(s) to the Sygma MPC network by updating the network topology.

Provide the Sygma team with the next params so we update the topology map:

1. The network address. This could be a DNS name or IP address.
2. Peer ID - determined from libp2p private key being used for that relayer (you can use relayers [CLI command](https://github.com/sygmaprotocol/sygma-relayer/blob/main/cli/peer/info.go) for this `peer info --private-key=<pk>`)

The final address of your relayer in the topology map will look this `"/dns4/relayer2/tcp/9001/p2p/QmeTuMtdpPB7zKDgmobEwSvxodrf5aFVSmBXX3SQJVjJaT"`

After all the information is provided Sygma team will regenerate Topology Map and initiate key Resharing by calling the Bridge contract [method](https://github.com/sygmaprotocol/sygma-solidity/blob/master/contracts/Bridge.sol#L351) with a new topology map hash.


### Confirm
Go to the logs.
You should see something like
`Processing 0 deposit events in blocks: 3422205 to 3422209` - this is the log that parses the blocks in search of bridging events. That means that relayer is up and listening.


## Other

#### To change the number of relayers to deploy
 - add for each relayer a separate config_map file and deployment file or scale any way you want


#### Log configuration & sharing logs with the Sygma
In order to share logs live with the Sygma team's DataDog monitoring system, we chose to deploy the [Fluentbit](https://fluentbit.io/) log processor and forwarder. We chose to deploy it via the helm chart available [here](https://github.com/fluent/helm-charts/tree/main/charts/fluent-bit).
You should first install the `fluent` helm repo:
```
helm repo add fluent https://fluent.github.io/helm-charts
```
And then you can install the `fluent-bit` release. As an example:
```
helm install fluent-bit fluent/fluent-bit
```
As for the `values.yaml` file, you should modify the `config` section accordingly in order to properly integrate with DataDog. As an example:
```
config:
  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*chainsafe_sigma-relayer*.log # THIS IS WHERE YOU SHOULD MATCH THE SIGMA RELAYER CONTAINER LOGS
        multiline.parser docker, cri
        Tag relayer # NAME YOUR TAG HERE
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
  outputs: |
    [OUTPUT]
        Name              datadog
        Match             relayer # THIS SHOULD BE THE SAME AS TAG FROM INPUT SECTION
        Host              http-intake.logs.datadoghq.com
        TLS               on
        compress          gzip
        apikey            DD_API_KEY # YOU SHOULD GET THIS FROM THE SYGMA TEAM
        dd_service        # COMPLETE ACCORDING TO YOUR RELAYER
        dd_source         # COMPLETE ACCORDING TO YOUR RELAYER
        dd_message_key    log
        dd_tags           env:,project:,relayerid: # COMPLETE TAGS ACCORDING TO ENVIRONMENT AND YOUR RELAYER

```
If you are running more than 1 relayer and you wish to send logs for all of them, you should define separate INPUTS and OUTPUTS according to your needs. As an example:
```
config:
  inputs: |
    [INPUT]
        Name tail
        Path /var/log/containers/*chainsafe_sigma-relayerX*.log # THIS IS WHERE YOU SHOULD MATCH THE SIGMA RELAYER CONTAINER LOGS
        multiline.parser docker, cri
        Tag relayer-X # NAME YOUR TAG HERE
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
    [INPUT]
        Name tail
        Path /var/log/containers/*chainsafe_sigma-relayerY*.log # THIS IS WHERE YOU SHOULD MATCH THE SIGMA RELAYER CONTAINER LOGS
        multiline.parser docker, cri
        Tag relayer-Y # NAME YOUR TAG HERE
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On
  outputs: |
    [OUTPUT]
        Name              datadog
        Match             relayer-X # THIS SHOULD BE THE SAME AS TAG FROM INPUT SECTION
        Host              http-intake.logs.datadoghq.com
        TLS               on
        compress          gzip
        apikey            DD_API_KEY # YOU SHOULD GET THIS FROM THE SYGMA TEAM
        dd_service        # COMPLETE ACCORDING TO YOUR RELAYER
        dd_source         # COMPLETE ACCORDING TO YOUR RELAYER
        dd_message_key    log
        dd_tags           env:,project:,relayerid: # COMPLETE TAGS ACCORDING TO ENVIRONMENT AND YOUR RELAYER
    [OUTPUT]
        Name              datadog
        Match             relayer-Y # THIS SHOULD BE THE SAME AS TAG FROM INPUT SECTION
        Host              http-intake.logs.datadoghq.com
        TLS               on
        compress          gzip
        apikey            DD_API_KEY # YOU SHOULD GET THIS FROM THE SYGMA TEAM
        dd_service        # COMPLETE ACCORDING TO YOUR RELAYER
        dd_source         # COMPLETE ACCORDING TO YOUR RELAYER
        dd_message_key    log
        dd_tags           env:,project:,relayerid: # COMPLETE TAGS ACCORDING TO ENVIRONMENT AND YOUR RELAYER

```

### OTLP AGENT 
We use OpenTelemetry Agent as a sidecar container for aggragting relayers metrics, for now. Read the followings to build the OpenTelemetry Agent

**Two stages are required for the configuration**
- Building OpenTemetry Agent
- Configuring the otel-collector in deployment file

#### Building OpenTemetry Agent
See the otlp-agent dirctory [here](https://github.com/sygmaprotocol/sygma-relayer-deployment/tree/main/otlp-agent).
The agent requires three major files:
- Builder: `otlp-builder.yml`
- Config File: `otlp-config.yml`
- Dockerfile


#### The Integration of the OpenTelemetry Agent

The Otlp Agent endpoint must be set on the Relayers as environment variable
```
               "name": "SYG_RELAYER_OPENTELEMETRYCOLLECTORURL",
               "value": "http://localhost:4318"
```

For easy reference, the env variables should be Organisation name with the environment to differentiate Relayers on the network.
For `SYG_RELAYER_ENV` use TESTNET if it is a testnet instance and `MAINNET` if it is a production.
For `SYG_RELAYER_ID` we need to make sure that it is unique for all relayers. So make sure that you have consulted with Sygma team about proper relayerid.
```
               "name": "SYG_RELAYER_ID",
               "value": "{{ relayerId }}"
            
            
               "name": "SYG_RELAYER_ENV",
               "value": "TESTNET"
```

#### Sharing metrics with Sygma
For sharing metrics you would need to use DataDog API key provided by Sygma team. Set this key to DD_API_KEY env variable in the deployment file.


The Sygma Team Highly Recomend to use private repository for the otlp agent.
