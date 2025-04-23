# Cross-Domain Relationship Diagrams

The "Graph of Graph" analyses represent interconnected data domains. These connections can be illustrated through diagrams that clearly display cross-domain relationships:

```mermaid
flowchart TD
    IG[Identity Graph] <-->|Events| NG[Network Graph]
    NG <--> |Events| AG[Asset Graph]
    
    IG --> CA[Compromised<br>Accounts]
    NG --> LM[Lateral<br>Movement]
    AG --> SR[Sensitive<br>Resources]
    
    CA --> CAP[Complete Attack<br>Paths]
    LM --> CAP
    SR --> CAP
    
    classDef graphNode fill:#bbf,stroke:#333,stroke-width:2px;
    classDef analysisNode fill:#bfb,stroke:#333,stroke-width:2px;
    classDef resultNode fill:#f9f,stroke:#333,stroke-width:2px;
    
    class IG,NG,AG graphNode;
    class CA,LM,SR analysisNode;
    class CAP resultNode;
```

This diagram illustrates how separate domain-specific analyses integrate to provide a comprehensive understanding of potential attack paths. Note that this is a conceptual representation of cross-domain relationships, not a visualization of the actual Kusto graph query results.

## Additional Applications of Cross-Domain Relationships

Beyond security, the "Graph of Graph" approach in Kusto enables analysis across multiple connected domains:

### Supply Chain Optimization

By connecting:

- **Supplier Graphs**: Representing supplier relationships and capabilities
- **Logistics Graphs**: Modeling transportation routes, times, and costs
- **Inventory Graphs**: Tracking inventory levels and dependencies

Organizations can identify optimal sourcing strategies, predict potential bottlenecks, and create resilient supply chains by analyzing these interconnected domains.

### Customer Journey Analysis

By connecting:

- **Customer Identity Graphs**: Representing customer profiles and segments
- **Interaction Graphs**: Modeling touchpoints across channels
- **Product Graphs**: Cataloging product relationships and dependencies

Businesses can trace complete customer journeys across channels, identify high-value conversion paths, and optimize the overall customer experience through cross-domain analysis.
