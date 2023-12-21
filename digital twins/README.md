# Digital Twins and Historizing Graph Changes to Kusto

Digital twins are virtual representations of physical objects or systems. They provide a way to model and simulate real-world entities, enabling organizations to gain insights and optimize their operations. One of the key challenges in managing digital twins is capturing and analyzing the changes that occur over time.

To effectively analyze timeseries data based on the graph representation of digital twins, it is crucial to historize the changes in the graph to a data storage and analytics platform like Kusto. Kusto, also known as Azure Data Explorer, is a fast and highly scalable data exploration service that enables real-time analysis of large volumes of data.

By historizing the changes in the graph to Kusto, organizations can:

1. **Track and analyze historical trends**: Storing the changes in the graph to Kusto allows organizations to track and analyze historical trends in the behavior of digital twins. This can provide valuable insights into the performance, maintenance, and optimization of physical assets.

2. **Perform time-based analysis**: With the historized graph data in Kusto, organizations can perform time-based analysis to understand how the state of digital twins evolves over time. This can help identify patterns, anomalies, and correlations that can drive informed decision-making.

3. **Enable predictive analytics**: By combining the historized graph data with other relevant data sources, organizations can leverage predictive analytics techniques to forecast future behavior and performance of digital twins. This can enable proactive maintenance, resource allocation, and optimization strategies.

To historize the changes in the graph to Kusto, organizations can leverage various techniques such as event sourcing, change data capture, or periodic snapshots. The choice of technique depends on factors like the complexity of the graph, the frequency of changes, and the desired granularity of historization.

## Setup an Azure Digital Twins service and configure data historization

[Azure Digital Twins](https://learn.microsoft.com/azure/digital-twins/) is a cloud-based service provided by Microsoft. It allows developers to model the relationships and interactions between various entities within these environments, such as devices, sensors, and people.

To configure Azure Digital Twins for data historization, you need to follow these steps

- [Create the Azure Digital Twins instance](https://learn.microsoft.com/azure/digital-twins/how-to-set-up-instance-portal).
- [Setup Data history](https://learn.microsoft.com/azure/digital-twins/concepts-data-history).

In order to ensure that the subsequent data definition language constructs work well, consider using the following table names during the setup of the data history feature in Azure Digital Twins:

- Twin property updates: **AdtPropertyEvents**
- Twin lifecycle events: **AdtTwinLifecycleEvents**
- Relationship lifecycle events: **AdtRelationshipLifecycleEvents**

## Create the entities to work with the data historized in Kusto

Once the Azure Digital Twins service is set up and the data historization is configured, organizations need to create the necessary entities in Kusto to work with the historized data. This includes creating tables, materialized views, and functions that will be used for querying and analyzing the historized graph data. The complete script to create all entities required can be found in the [DDL](DDL.kql) file.

The tables were created by the Azure Digital Twins service. If the stream of data into each table is not high (more than 4 GB per hour), it's recommended to use [streaming ingestion](https://learn.microsoft.com/azure/data-explorer/ingest-data-streaming). The following commands are enabling streaming ingestion.

```kusto
// Enable streaming ingestion policy on AdtRelationshipLifecycleEvents table
.alter table AdtRelationshipLifecycleEvents policy streamingingestion enable 

// Enable streaming ingestion policy on AdtTwinLifecycleEvents table
.alter table AdtTwinLifecycleEvents policy streamingingestion enable

// Enable streaming ingestion policy on AdtPropertyEvents table
.alter table AdtPropertyEvents policy streamingingestion enable 
```

To get the last known state of the **twin property updates** one can create a deduplication materialized view using the _arg_max_ aggregation function. The second command creates a function without parameters which filters deleted properties. In some scenarios it's interesting the get state of a property of a twin at a given point in time. The last command creates a stored function which provides the properties of a twin at a certain point in time.

```kusto
// Create a materialized view that contains the latest record for each TwinId property
.create materialized-view with (backfill = true) AdtTwinPropertyEventsDedup on table AdtPropertyEvents {
    AdtPropertyEvents
    | where isempty(RelationshipId)
    | project-away RelationshipTarget, RelationshipId
    | summarize arg_max(TimeStamp, *) by Id, ModelId, Key
}

// Create a function that returns the latest record for each TwinId property where Action is not "Delete"
.create-or-alter function TwinPropertiesLKV () {
AdtTwinPropertyEventsDedup
| where Action != "Delete"
}

// Create a function that returns the latest record for each TwinId property where Action is not "Delete" for a certain point in time
.create-or-alter function TwinProperties (pointInTime:datetime) {
 AdtPropertyEvents
    | where isempty(RelationshipId) and TimeStamp < pointInTime
    | project-away RelationshipTarget, RelationshipId
    | summarize arg_max(TimeStamp, *) by Id, ModelId, Key
    | where Action != "Delete"
}
```

The same pattern can now be applied to the **Twin lifecycle events** and **Relationship lifecycle events**.

As described in the [graph semantics documentation](https://learn.microsoft.com/azure/data-explorer/graph-overview) for Kusto, it's required to have a tabular expression for edges and optionally nodes. The following commands create two functions. _Edges_ is able to provide the adjacency list including a property bag at a given point in time. It leverages the functions which were created previously. _EdgesLKV_ creates the same output but it leverages the last known value materialized view. This is an optimization to get to the last known state of the edges in a more performant way. A similar pattern can be applied to get the tabular expression for the nodes of the graph.

```kusto
// Create a function that returns the edges for a certain point in time
.create-or-alter function with (skipvalidation = "true") Edges (pointInTime:datetime, interestingProperties: dynamic = dynamic([])) {
    AdtRelationshipLifecycleEvents
    | where TimeStamp < pointInTime
    | summarize arg_max(TimeStamp, *) by RelationshipId
    | where Action != "Delete"
    | join kind=leftouter (
        EdgeProperties(pointInTime)
        | where Key in (interestingProperties) or array_length(interestingProperties) == 0
        | extend p = bag_pack(Key, Value)
        | summarize Properties = make_bag(p) by RelationshipId
    ) on RelationshipId
    | project-away RelationshipId1
}

// Create a function that returns the latest record for each RelationshipId where Action is not "Delete"
.create-or-alter function EdgesLKV (interestingProperties: dynamic = dynamic([])) {
    AdtRelationshipLifecycleEventsDedup
    | where Action != "Delete"
    | join kind=leftouter (
        EdgePropertiesLKV
        | where Key in (interestingProperties) or array_length(interestingProperties) == 0
        | extend p = bag_pack(Key, Value)
        | summarize Properties = make_bag(p) by RelationshipId
    ) on RelationshipId
    | project-away RelationshipId1
}
```

Now we have all the pieces to create the graph. The following commands create either the last known graph or a graph at a specific point in time.

```kusto
// Create or alter a function that returns the latest version of a graph
.create-or-alter function with (skipvalidation = "true") GraphLKV(interestingEdgeProperties: dynamic = dynamic([]), interestingNodeProperties: dynamic = dynamic([])) {
EdgesLKV(interestingEdgeProperties)
| make-graph Source --> Target with NodesLKV(interestingNodeProperties) on TwinId
}

// Create a function that returns a graph for a certain point in time
.create-or-alter function with (skipvalidation = "true") Graph (pointInTime:datetime, interestingEdgeProperties: dynamic = dynamic([]), interestingNodeProperties: dynamic = dynamic([])) {
    Edges(pointInTime, interestingEdgeProperties)
    | make-graph Source --> Target with Nodes(pointInTime, interestingNodeProperties) on TwinId
}
```

## Ingesting sample data

Now we have a setup with Azure Digital Twins and Azure Data Explorer. It's ready to receive data. The goal of this section is to mimic a real-world example. We are starting with uploading a model to Azure Digital Twins and after that we are adding twins and relations based on the model.

The ontology that we are going to use is provided by Bosch Building Technologies and available on [github](https://github.com/boschglobal/building-technologies-ontology-central). We are following the [foundation examples](https://github.com/boschglobal/building-technologies-ontology-central/blob/main/com/bosch/bt/Foundation/2.0.0/samples/README.md) and concentrate on the [desk usage](https://github.com/boschglobal/building-technologies-ontology-central/blob/main/com/bosch/bt/Foundation/2.0.0/samples/README.md#desk---desk-usage) scenario.

### Uploading the ontology

The following steps will upload the ontologies into the Azure Digital Twin service.

1. [Open](https://learn.microsoft.com/azure/digital-twins/quickstart-azure-digital-twins-explorer#open-instance-in-azure-digital-twins-explorer) the Azure Digital Twins Explorer
1. Clone the [Bosch Building Technologies - Ontology Central](https://github.com/boschglobal/building-technologies-ontology-central) repository
1. [Upload](https://learn.microsoft.com/azure/digital-twins/quickstart-azure-digital-twins-explorer#upload-the-models-json-files) the foundation model [src folder](https://github.com/boschglobal/building-technologies-ontology-central/tree/main/com/bosch/bt/Foundation/2.0.0/src) 
1. [Upload](https://learn.microsoft.com/azure/digital-twins/quickstart-azure-digital-twins-explorer#upload-the-models-json-files) ontology for the [desk](https://github.com/boschglobal/building-technologies-ontology-central/blob/main/com/bosch/bt/Foundation/2.0.0/samples/src/Desk.json) scenario.

## Visualize Data, both in Azure Digital Twins and Azure Data Explorer

After the data historization is set up and the entities are created, organizations can visualize the historized data in both Azure Digital Twins and Azure Data Explorer. Azure Digital Twins provides a user-friendly interface for exploring and visualizing the graph data, while Azure Data Explorer offers advanced querying and visualization capabilities for in-depth analysis.

## Common query patterns

To effectively analyze the historized graph data in Kusto, organizations should be familiar with common query patterns. These include querying for specific time ranges, aggregating data over time intervals, filtering based on entity properties, and performing joins with other datasets. Understanding these query patterns can help organizations extract valuable insights from the historized graph data.

In conclusion, historizing the changes in the graph representation of digital twins to Kusto is essential for analyzing timeseries data and gaining valuable insights. It enables organizations to track historical trends, perform time-based analysis, and leverage predictive analytics techniques. By following the steps of setting up an Azure Digital Twins service, creating the necessary entities in Kusto, visualizing data in both Azure Digital Twins and Azure Data Explorer, and understanding common query patterns, organizations can optimize their operations, improve asset performance, and drive informed decision-making.
