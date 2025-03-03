import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"

# Kubecost

Our Kubecost integration allows you to import `kubesystem` and `cloud` cost allocations from your Kubecost instance into Port, according to your mapping and definition.

## Common use cases

- Map your monitored Kubernetes resources and cloud cost allocations in Kubecost.

## installation

Install the integration via Helm by running this command:

```bash showLineNumbers
# The following script will install an Ocean integration in your K8s cluster using helm
# integration.identifier: Change the identifier to describe your integration
# integration.config.kubecostHost: The URL of you Kubecost server. Used to make API calls

helm upgrade --install my-kubecost-integration port-labs/port-ocean \
	--set port.clientId="CLIENT_ID"  \
	--set port.clientSecret="CLIENT_SECRET"  \
	--set initializePortResources=true  \
    --set scheduledResyncInterval=60 \
	--set integration.identifier="my-kubecost-integration"  \
	--set integration.type="kubecost"  \
	--set integration.eventListener.type="POLLING"  \
	--set integration.config.kubecostHost="https://kubecostInstance:9090"
```

## Ingesting Kubecost objects

The Kubecost integration uses a YAML configuration to describe the process of loading data into the developer portal.

Here is an example snippet from the config which demonstrates the process for getting cost allocation data from Kubecost:

```yaml showLineNumbers
createMissingRelatedEntities: true
deleteDependentEntities: true
resources:
  - kind: kubesystem
    selector:
      query: "true"
      window: "month"
    port:
      entity:
        mappings:
          blueprint: '"kubecostResourceAllocation"'
          identifier: .name
          title: .name
          properties:
            cluster: .properties.cluster
            namespace: .properties.namespace
            startDate: .start
            endDate: .end
            cpuCoreHours: .cpuCoreHours
            cpuCost: .cpuCost
            cpuEfficiency: .cpuEfficiency
            gpuHours: .gpuHours
            gpuCost: .gpuCost
            networkCost: .networkCost
            loadBalancerCost: .loadBalancerCost
            pvCost: .pvCost
            pvBytes: .pvBytes
            ramBytes: .ramBytes
            ramCost: .ramCost
            ramEfficiency: .ramEfficiency
            sharedCost: .sharedCost
            externalCost: .externalCost
            totalCost: .totalCost
            totalEfficiency: .totalEfficiency
```

The integration makes use of the [JQ JSON processor](https://stedolan.github.io/jq/manual/) to select, modify, concatenate, transform and perform other operations on existing fields and values from Kubecost's API events.

### Configuration structure

The integration configuration determines which resources will be queried from Kubecost, and which entities and properties will be created in Port.

:::tip Supported resources
The following resources can be used to map data from Kubecost, it is possible to reference any field that appears in the API responses linked below for the mapping configuration.

- [`kubesystem`](https://docs.kubecost.com/apis/apis-overview/api-allocation#allocation-schema)
- [`cloud`](https://docs.kubecost.com/apis/apis-overview/cloud-cost-api#cloud-cost-aggregate-api)

:::

:::note
You will be able to see `cloud` cost data after you have successfully configured the Cloud Billing API on your Kubecost instance according to this [documentation](https://docs.kubecost.com/install-and-configure/install/cloud-integration)
:::

- The root key of the integration configuration is the `resources` key:

  ```yaml showLineNumbers
  # highlight-next-line
  resources:
    - kind: kubesystem
      selector:
      ...
  ```

- The `kind` key is a specifier for an Kubecost object:

  ```yaml showLineNumbers
    resources:
      # highlight-next-line
      - kind: kubesystem
        selector:
        ...
  ```

- The `selector` and the `query` keys allow you to filter which objects of the specified `kind` will be ingested into your software catalog:

  ```yaml showLineNumbers
  resources:
    - kind: kubesystem
      # highlight-start
      selector:
        query: "true" # JQ boolean expression. If evaluated to false - this object will be skipped.
        window: "month"
        aggregate: "pod"
        idle: true
      # highlight-end
      port:
  ```

  - **window** - Duration of time over which to query. Accepts: words like `today`, `week`, `month`, `yesterday`, `lastweek`, `lastmonth`; durations like `30m`, `12h`, `7d`; RFC3339 date pairs like `2021-01-02T15:04:05Z,2021-02-02T15:04:05Z`; Unix timestamps like `1578002645,1580681045`.
  - **aggregate** - Field by which to aggregate the results. Accepts: `cluster`, `node`, `namespace`, `controllerKind`, `controller`, `service`, `pod`, `container`, `label:name`, and `annotation:name`. Also accepts comma-separated lists for multi-aggregation, like `namespace,label:app`.
  - **step** - Duration of a single allocation set. If unspecified, this defaults to the window, so that you receive exactly one set for the entire window. If specified, such as `30m`, `2h`, `1d` etc, it works chronologically backward, querying in durations of step until the full window is covered. Default is `window`.
  - **accumulate** - If true, sum the entire range of sets into a single set. Default value is `false`.
  - **idle** - If true, include idle cost (i.e. the cost of the un-allocated assets) as its own allocation. Default is `true`.
  - **external** - If true, include external, or out-of-cluster costs in each allocation. Default is `false`.
  - **filterClusters** - Comma-separated list of clusters to match; e.g. `cluster-one,cluster-two` will return results from only those two clusters.
  - **filterNodes** - Comma-separated list of nodes to match; e.g. `node-one,node-two` will return results from only those two nodes.
  - **filterNamespaces** - Comma-separated list of namespaces to match; e.g. `namespace-one,namespace-two` will return results from only those two namespaces.
  - **filterControllerKinds** - Comma-separated list of controller kinds to match; e.g. `deployment`, job will return results with only those two controller kinds.
  - **filterControllers** - Comma-separated list of controllers to match; e.g. `deployment-one,statefulset-two` will return results from only those two controllers.
  - **filterPods** - Comma-separated list of pods to match; e.g. `pod-one,pod-two` will return results from only those two pods.
  - **filterAnnotations** - Comma-separated list of annotations to match; e.g. `name:annotation-one,name:annotation-two` will return results with either of those two annotation key-value-pairs.
  - **filterControllerKinds** - Comma-separated list of controller kinds to match; e.g. `deployment`, job will return results with only those two controller kinds.
  - **filterLabels** - Comma-separated list of annotations to match; e.g. `app:cost-analyzer, app:prometheus` will return results with either of those two label key-value-pairs.
  - **filterServices** - Comma-separated list of services to match; e.g. `frontend-one,frontend-two` will return results with either of those two services.
  - **shareIdle** - If true, idle cost is allocated proportionally across all non-idle allocations, per-resource. That is, idle CPU cost is shared with each non-idle allocation's CPU cost, according to the percentage of the total CPU cost represented. Default is `false`.
  - **splitIdle** - If true, and shareIdle == false, Idle Allocations are created on a per cluster or per node basis rather than being aggregated into a single idle allocation. Default is `false`.
  - **idleByNode** - If true, idle allocations are created on a per node basis. Which will result in different values when shared and more idle allocations when split. Default is `false`.
  - And any query parameter that could be found in the [Kubecost allocation API](https://docs.kubecost.com/apis/apis-overview/api-allocation#allocation-api) and [Kubecost Cloud API](https://docs.kubecost.com/apis/apis-overview/cloud-cost-api#cloud-cost-aggregate-api)

- The `port`, `entity` and the `mappings` keys are used to map the Kubecost object fields to Port entities. To create multiple mappings of the same kind, you can add another item in the `resources` array;

  ```yaml showLineNumbers
  resources:
    - kind: kubesystem
      selector:
        query: "true"
      port:
        # highlight-start
        entity:
          mappings: # Mappings between one Kubecost object to a Port entity. Each value is a JQ query.
            identifier: .name
            title: .name
            blueprint: '"KubecostResourceAllocation"'
            properties:
              cluster: .properties.cluster
              namespace: .properties.namespace
              startDate: .start
              endDate: .end
              cpuCoreHours: .cpuCoreHours
              cpuCost: .cpuCost
              cpuEfficiency: .cpuEfficiency
              gpuHours: .gpuHours
              gpuCost: .gpuCost
              networkCost: .networkCost
              loadBalancerCost: .loadBalancerCost
              pvCost: .pvCost
              ramBytes: .ramBytes
              ramCost: .ramCost
              ramEfficiency: .ramEfficiency
              sharedCost: .sharedCost
              externalCost: .externalCost
              totalCost: .totalCost
              totalEfficiency: .totalEfficiency
        # highlight-end
    - kind: kubesystem # In this instance cost is mapped again with a different filter
      selector:
        query: '.name == "MyNodeName"'
      port:
        entity:
          mappings: ...
  ```

  :::tip Blueprint key
  Note the value of the `blueprint` key - if you want to use a hardcoded string, you need to encapsulate it in 2 sets of quotes, for example use a pair of single-quotes (`'`) and then another pair of double-quotes (`"`)
  :::

### Ingest data into Port

To ingest Kubecost objects using the [integration configuration](#configuration-structure), you can follow the steps below:

1. Go to the DevPortal Builder page.
2. Select a blueprint you want to ingest using Kubecost.
3. Choose the **Ingest Data** option from the menu.
4. Select Kubecost under the Cloud cost providers category.
5. Modify the [configuration](#configuration-structure) according to your needs.
6. Click `Resync`.

## Examples

Examples of blueprints and the relevant integration configurations:

### Cost allocation

<details>
<summary>Cost allocation blueprint</summary>

```json showLineNumbers
{
  "identifier": "kubecostResourceAllocation",
  "description": "This blueprint represents an Kubecost resource allocation in our software catalog",
  "title": "Kubecost Resource Allocation",
  "icon": "Cluster",
  "schema": {
    "properties": {
      "cluster": {
        "type": "string",
        "title": "Cluster"
      },
      "namespace": {
        "type": "string",
        "title": "Namespace"
      },
      "startDate": {
        "title": "Start Date",
        "type": "string",
        "format": "date-time"
      },
      "endDate": {
        "title": "End Date",
        "type": "string",
        "format": "date-time"
      },
      "cpuCoreHours": {
        "title": "CPU Core Hours",
        "type": "number"
      },
      "cpuCost": {
        "title": "CPU Cost",
        "type": "number"
      },
      "cpuEfficiency": {
        "title": "CPU Efficiency",
        "type": "number"
      },
      "gpuHours": {
        "title": "GPU Hours",
        "type": "number"
      },
      "gpuCost": {
        "title": "GPU Cost",
        "type": "number"
      },
      "networkCost": {
        "title": "Network Cost",
        "type": "number"
      },
      "loadBalancerCost": {
        "title": "Load Balancer Cost",
        "type": "number"
      },
      "pvCost": {
        "title": "PV Cost",
        "type": "number"
      },
      "pvBytes": {
        "title": "PV Bytes",
        "type": "number"
      },
      "ramBytes": {
        "title": "RAM Bytes",
        "type": "number"
      },
      "ramCost": {
        "title": "RAM Cost",
        "type": "number"
      },
      "ramEfficiency": {
        "title": "RAM Efficiency",
        "type": "number"
      },
      "sharedCost": {
        "title": "Shared Cost",
        "type": "number"
      },
      "externalCost": {
        "title": "External Cost",
        "type": "number"
      },
      "totalCost": {
        "title": "Total Cost",
        "type": "number"
      },
      "totalEfficiency": {
        "title": "Total Efficiency",
        "type": "number"
      }
    },
    "required": []
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {}
}
```

</details>

<details>
<summary>Integration configuration</summary>

```yaml showLineNumbers
createMissingRelatedEntities: true
deleteDependentEntities: true
resources:
  - kind: kubesystem
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"kubecostResourceAllocation"'
          identifier: .name
          title: .name
          properties:
            cluster: .properties.cluster
            namespace: .properties.namespace
            startDate: .start
            endDate: .end
            cpuCoreHours: .cpuCoreHours
            cpuCost: .cpuCost
            cpuEfficiency: .cpuEfficiency
            gpuHours: .gpuHours
            gpuCost: .gpuCost
            networkCost: .networkCost
            loadBalancerCost: .loadBalancerCost
            pvCost: .pvCost
            pvBytes: .pvBytes
            ramBytes: .ramBytes
            ramCost: .ramCost
            ramEfficiency: .ramEfficiency
            sharedCost: .sharedCost
            externalCost: .externalCost
            totalCost: .totalCost
            totalEfficiency: .totalEfficiency
```

</details>

### Cloud cost

<details>
<summary> Cloud cost blueprint</summary>

```json showlineNumbers
{
  "identifier": "kubecostCloudAllocation",
  "description": "This blueprint represents an Kubecost cloud resource allocation in our software catalog",
  "title": "Kubecost Cloud Allocation",
  "icon": "Cluster",
  "schema": {
    "properties": {
      "provider": {
        "type": "string",
        "title": "Provider"
      },
      "accountID": {
        "type": "string",
        "title": "Account ID"
      },
      "invoiceEntityID": {
        "type": "string",
        "title": "Invoice Entity ID"
      },
      "startDate": {
        "title": "Start Date",
        "type": "string",
        "format": "date-time"
      },
      "endDate": {
        "title": "End Date",
        "type": "string",
        "format": "date-time"
      },
      "listCost": {
        "title": "List Cost Value",
        "type": "number"
      },
      "listCostPercent": {
        "title": "List Cost Percent",
        "type": "number"
      },
      "netCost": {
        "title": "Net Cost Value",
        "type": "number"
      },
      "netCostPercent": {
        "title": "Net Cost Percent",
        "type": "number"
      },
      "amortizedNetCost": {
        "title": "Amortized Net Cost",
        "type": "number"
      },
      "amortizedNetCostPercent": {
        "title": "Amortized Net Cost Percent",
        "type": "number"
      },
      "invoicedCost": {
        "title": "Invoice Cost",
        "type": "number"
      },
      "invoicedCostPercent": {
        "title": "Invoice Cost Percent",
        "type": "number"
      }
    },
    "required": []
  },
  "mirrorProperties": {},
  "calculationProperties": {},
  "relations": {}
}
```

</details>

<details>
<summary>Integration configuration</summary>

```yaml showLineNumbers
createMissingRelatedEntities: true
deleteDependentEntities: true
resources:
  - kind: cloud
    selector:
      query: "true"
    port:
      entity:
        mappings:
          blueprint: '"kubecostCloudAllocation"'
          identifier: .properties.service
          title: .properties.service
          properties:
            provider: .properties.provider
            accountID: .properties.accountID
            invoiceEntityID: .properties.invoiceEntityID
            startDate: .window.start
            endDate: .window.end
            listCost: .listCost.cost
            listCostPercent: .listCost.kubernetesPercent
            netCost: .netCost.cost
            netCostPercent: .netCost.kubernetesPercent
            amortizedNetCost: .amortizedNetCost.cost
            amortizedNetCostPercent: .amortizedNetCost.kubernetesPercent
            invoicedCost: .invoicedCost.cost
            invoicedCostPercent: .invoicedCost.kubernetesPercent
```

</details>
