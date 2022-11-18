# Network Performance Workflow

## Workflow Description

This workflow defines and end-to-end benchmark test for network performance, specifically using the [uperf](https://github.com/uperf/uperf) benchmark utility. This is targeted at Kubernetes environments and includes orchestration of necessary objects via the k8s APIs.

A single top-level schema provides all of the input constructs required to describe the benchmark workload to be run, as well global parameters that further define the environment within which the test is run and how other parallel data collections will be handled. Finally, all data and metadata from the sequence of tests are post-processed into an approproate document format and indexed into Elasticsearch.

## Workflow Schema
The workflow schema defines the input parameters expected from the workflow user. The workflow author can determine how these parameters are presented to the user, independent of the subordinate plugin schemas. 

In this example workflow, we require only three input parameters: `kubeconfig`, `run_id`, and an object for `elasticsearch` connectivity.

```yaml
input:
  root: RootObject
  objects:
    RootObject:
      id: RootObject
      properties:
        kubeconfig:
          display:
            description: The complete kubeconfig file as a string
            name: Kubeconfig file contents
          type:
            type_id: string
        run_id:
          display:
            description: A unique run identifier for tracking groups of workflows triggered by external automation/CI
            name: Run ID
          type:
            type_id: string
          default: "\"\""
          required: false
        elasticsearch:
          display:
            description: The ElasticSearch service access parameters
            name: ElasticSearch parameters
          type:
            type_id: list
            items:
              id: ElasticSearch
              type_id: ref
    ElasticSearch:
      id: ElasticSearch
      properties:
        host: 
          display:
            description: The URL for the ElasticSearch host
            name: ElasticSearch host URL
          type:
            type_id: string
        username:
          display:
            description: The username for the ElasticSearch service
            name: ElasticSearch username
          type:
            type_id: string
        password:
          display:
            description: The password for the ElasticSearch service
            name: ElasticSearch password
          type:
            type_id: string
        index:
          display:
            description: The index for the ElasticSearch service
            name: ElasticSearch index
          type:
            type_id: string

```

## Workflow Definition
The workflow definition provides the series of steps to be performed by plugins or engine functions. You can think of each plugin step as being in a "star pattern" with the engine at the center. The engine passes input to the plugin's input API as defined by it's schema, and it receives output from the plugin's output API. The engine is responsible for passing the output of one plugin to the input of another in order to faciliate the chain of steps as defined in the workflow.

Relationships between the workflow schema, the engine, and the individual plugins can be managed with an expression language based on [Jsonnet](https://jsonnet.org/). We can (among other things):

1. Pass the parameters provided by the user to the workflow schema through to specific plugins
    - **Example:** The `kubeconfig` plugin receives the kubeconfig as a string from the workflow input.
2. Pass the output of one plugin to the configuration the engine will use to deploy another plugin
    - **Example:** The output of the `kubeconfig` plugin (a set of kubernetes authentication credentials) is used as input to the deploy configs of the plugins that will run in kubernetes.
3. Pass the output of one plugin to the input of another plugin
    - **Example:** The outputs of the `uuidgen`, `uperf_client`, and `metadata` plugins are passed to the `elasticsearch` plugin as well as to the final workflow `output`.

Note that all plugins have their own schemas and sets of input requirements. Within the workflow, the author can determine how values will be passed to the plugins. You can choose to expose those parameters as part of the workflow schema to the user, or you can set or pass by variable values directly within the workflow itself, depending on your needs and use case.

```yaml
steps:
  uuidgen:
    plugin: quay.io/arcalot/arcaflow-plugin-utilities:latest
    step: uuid
    input: {}
  kubeconfig:
    plugin: quay.io/arcalot/arcaflow-plugin-kubeconfig:latest
    input:
      kubeconfig: !expr $.input.kubeconfig
  uperf_server:
    plugin: quay.io/arcalot/arcaflow-plugin-uperf:latest
    step: uperf_server
    deploy:
      type: kubernetes
      connection: !expr $.steps.kubeconfig.outputs.success.connection
      pod:
        metadata:
          namespace: default
          labels:
            arcaflow: uperf
        spec:
          pluginContainer:
            imagePullPolicy: Always
    input:
      run_duration: 60
      port: 20000
  service:
    plugin: quay.io/arcalot/arcaflow-plugin-service:latest
    input:
      connection: !expr $.steps.kubeconfig.outputs.success.connection
      service:
        metadata:
          namespace: default
        spec:
          ports:
           - name: control
             port: 20000
             protocol: TCP
           - name: controludp
             port: 20000
             protocol: UDP
           - name: workload
             port: 30000
             protocol: TCP
           - name: workloadudp
             port: 30000
             protocol: UDP
          selector:
            arcaflow: uperf
  uperf_client:
    plugin: quay.io/arcalot/arcaflow-plugin-uperf:latest
    step: uperf
    deploy:
      type: kubernetes
      connection: !expr $.steps.kubeconfig.outputs.success.connection
      pod:
        spec:
          pluginContainer:
            imagePullPolicy: Always
    input:
      port: 20000
      name: "netperf"
      groups:
        - nthreads: 1
          transactions:
            - iterations: 1
              flowops:
                - type: "accept"
                  remotehost: !expr $.steps.service.outputs.success.name
                  port: 20000
                  protocol: "tcp"
            - duration: "10s"
              flowops:
                - type: "write"
                  size: 64
                - type: "read"
                  size: 64
            - iterations: 1
              flowops:
                - type: "disconnect"
  metadata:
    plugin: quay.io/arcalot/arcaflow-plugin-metadata:latest
    deploy:
      type: kubernetes
      connection: !expr $.steps.kubeconfig.outputs.success.connection
      pod:
        spec:
          pluginContainer:
            imagePullPolicy: Always
    input: {}
  elasticsearch:
    plugin: quay.io/arcalot/arcaflow-plugin-elasticsearch:latest
    input:
      url: !expr $.input.elasticsearch.host
      username: !expr $.input.elasticsearch.username
      password: !expr $.input.elasticsearch.password
      index: !expr $.input.elasticsearch.index
      data:
        uuid: !expr $.steps.uuidgen.outputs.success.uuid
        run_id: !expr $.input.run_id
        uperf: !expr $.steps.uperf_client.outputs.success
        metadata: !expr $.steps.metadata.outputs.success
output:
  uuid: !expr $.steps.uuidgen.outputs.success.uuid
  run_id: !expr $.input.run_id
  uperf: !expr $.steps.uperf_client.outputs.success
  metadata: !expr $.steps.metadata.outputs.success
```