# Building a resource graph using Kusto Graph Semantics

A resource graph allows you to explore and query your resources at scale using a powerful query language. It helps you gain insights into your resource inventory, track changes, and perform complex queries to support governance, management, and security scenarios. This service is particularly useful for large environments where manual management and querying would be time-consuming and error-prone. Additionally, a resource graph is an activity graph that gets constantly updated as resources change over time.

Managing authorization and permissions on assets tracked by a resource graph is crucial for ensuring security and proper access control. Permissions can be managed by assigning them to a group or an identity. Identities can be part of a group, and groups can be nested within other groups, allowing for flexible and scalable permission management. This hierarchical structure enables administrators to efficiently manage access rights and ensure that only authorized users can interact with specific resources.

Kusto Graph Semantics is optimized for activity graphs, which represent dynamic systems where entities and their relationships change over time. The resource graph scenario focuses on the exploration and querying of resources at scale.

**Use Cases of a Resource Graph**

- **Inventory Management**: Quickly gain insights into your resource inventory across multiple environments.
- **Change Tracking**: Track changes to resources over time to support governance and compliance.
- **Complex Queries**: Perform complex queries to analyze resource information and support security scenarios.
- **Access Control**: Manage permissions efficiently using group-based permissions to enhance security.

By following the guidelines and examples provided, organizations can harness the power of Kusto to gain valuable insights into their resource inventory, track changes, and perform complex queries to support governance, management, and security scenarios.

## Getting data

To get data for a resource graph, you can use services like Azure Resource Graph to query resource changes. Refer to the [documentation on how to get resource changes](https://learn.microsoft.com/azure/governance/resource-graph/changes/get-resource-changes) for more details.

## Create entities

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

## Visualize the graph

After the data historization is set up and the entities are created, organizations can visualize the historized data in both Azure Digital Twins and Azure Data Explorer. Azure Digital Twins provides a user-friendly interface for exploring and visualizing the graph data, while Azure Data Explorer offers advanced querying and visualization capabilities for in-depth analysis.

[Azure Digital Twins Explorer](https://learn.microsoft.com/azure/digital-twins/concepts-azure-digital-twins-explorer) is a tool that helps you to manage and visualize your digital twin data. It provides a user-friendly interface to interact with your Azure Digital Twins instance, allowing you to explore the digital twin graph, create or update twins and relationships, and execute queries.

With Azure Digital Twins Explorer, you can:

- Visualize the topology of your digital twin graph.
- Create, update, and delete digital twins and relationships.
- Query your digital twin data using the built-in query editor.
- Monitor changes in your digital twin data in real-time.

To use Azure Digital Twins Explorer, you need to have an Azure Digital Twins instance and the necessary permissions to access it. Once you have these, you can open the Azure Digital Twins Explorer and connect to your instance to start exploring your digital twin data.

The following image shows the twin graph based on the data we loaded in the previous steps.

![You should see a picture of Azure Digital Twins showing a graph of twins](media/azure-digital-twins-viz.png "Azure Digital Twins visualization")

[Kusto Explorer](https://learn.microsoft.com/azure/data-explorer/kusto/tools/kusto-explorer) is a rich desktop application that enables you to explore your data in Azure Data Explorer (Kusto). It provides a powerful interface for running Kusto Query Language (KQL) queries, with features like IntelliSense, syntax highlighting, and tabular and graphical results.

With the new graph semantics feature, Kusto Explorer can be used to visualize graph data. This allows you to:

- Visualize the structure of your graph data in a graphical format.
- Explore the relationships between different entities in your graph.
- Run complex graph queries and see the results in a visual, easy-to-understand format.

To use Kusto Explorer, you need to have an Azure Data Explorer cluster and the necessary permissions to access it. Once you have these, you can open Kusto Explorer, connect to your cluster, and start exploring your graph data.

Calling the function, which we created earlier, results in a graph being drawn in Kusto Explorer.

```kusto
GraphLKV()
```

![You should see a picture of Kusto Explorer showing a graph of twins](media/kusto-explorer-viz.png "Kusto Explorer visualization")

## Common query patterns

To effectively analyze the historized graph data in Kusto, organizations should be familiar with common query patterns. These include querying for specific time ranges, aggregating data over time intervals, filtering based on entity properties, and performing joins with other datasets. Understanding these query patterns can help organizations extract valuable insights from the historized graph data.

In conclusion, historizing the changes in the graph representation of digital twins to Kusto is essential for analyzing timeseries data and gaining valuable insights. It enables organizations to track historical trends, perform time-based analysis, and leverage predictive analytics techniques. By following the steps of setting up an Azure Digital Twins service, creating the necessary entities in Kusto, visualizing data in both Azure Digital Twins and Azure Data Explorer, and understanding common query patterns, organizations can optimize their operations, improve asset performance, and drive informed decision-making.

```kusto
// Rooted Query (Given a random TwinId, retrieve ID + all properties of all twins exactly 5 hops away)
GraphLKV()
| graph-match cycles=none (n1)-[e*5..5]-(n2)
	where n1.TwinId == "site-1"
	project n2.TwinId, n2.Properties
```

|n2_TwinId|n2_Properties|
|---|---|
|occupancy-1|{<br>  "function": "Virtual",<br>  "type": "Occupancy"<br>}|
|occupancy-3|{<br>  "function": "Virtual",<br>  "type": "Occupancy",<br>  "status": null,<br>  "currentValue.value": null<br>}|
|occupancy-2|{<br>  "function": "Virtual",<br>  "type": "Occupancy"<br>}|
|occupancy-4|{<br>  "function": "Virtual",<br>  "type": "Occupancy",<br>  "currentValue.value": 1,<br>  "status": "occupied"<br>}|

```kusto
// Unrooted query (Find two twins where a given highly changing property value is the same which are up to 5 hops away from each other.)
let interestingProperties = dynamic(["function"]);
GraphLKV(interestingProperties)
| graph-match cycles=none (n1)-[e*1..5]-(n2)    
	where tostring(n1.Properties.function) == tostring(n2.Properties.function) and isnotempty(n1.Properties.function)
	project n1.TwinId, n2.TwinId
```

|n1_TwinId|n2_TwinId|
|---|---|
|occupancy-3|occupancy-1|
|occupancy-3|occupancy-2|
|occupancy-1|occupancy-3|
|occupancy-1|occupancy-2|
|occupancy-2|occupancy-1|
|occupancy-2|occupancy-3|


```kusto
GraphLKV()
| graph-match (occupancy)-->(desk)-->(room)
	where 
		occupancy.ModelId == "dtmi:com:bosch:bt:foundation:point:MultiStateDataPoint;2" and 
		occupancy.Properties.status != "occupied" and
		room.ModelId == "dtmi:com:bosch:bt:foundation:space:Space;2" 
	project desk = desk.TwinId, room = room.TwinId
| summarize count() by room
```

|room|count_|
|---|---|
|room-1|3|


```kusto
GraphLKV()
| graph-match (occupancy)-->(desk)-->(room)
	where occupancy.ModelId == "dtmi:com:bosch:bt:foundation:point:MultiStateDataPoint;2" and room.ModelId == "dtmi:com:bosch:bt:foundation:space:Space;2"
	project occupied = tostring(occupancy.Properties.status), room = room.TwinId
| summarize occupancy = round(dcount(occupied == "occoupied") / toreal(count()),2) by room
```

|room|occupancy|
|---|---|
|room-1|0,33|
|room-2|1|


```kusto
// Star-pattern: where are rooms with empty desks and christian is located in that room
GraphLKV()
| graph-match (alice)-[e*1..5]-(room)<--(desk)<--(occupancy),(bob)-->(room)
	where alice.TwinId == "Alice" and occupancy.Properties.status != "occupied" and occupancy.ModelId == "dtmi:com:bosch:bt:foundation:point:MultiStateDataPoint;2" and 
	bob.TwinId == "Bob"
	project desk = desk.TwinId, room = room.TwinId
| summarize count() by room
```

```kusto
//Create a graph at a specific point in time
Graph(datetime(2022-03-22))
```