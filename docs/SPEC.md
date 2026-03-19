The goal is to create a data mesh platform using technology like gRPC, REST, and Kafka. The main goals I want to accomplish are as follows:

- I want to make it easy for engineers to expose data via a range of protocols in a standard format organized around entities absolute identifiers. I want this to be distributed so that different services could export data about the same entity by "extending" that entity's type to include the new data exposed by that service. These will be correlated as all services will communicate about an entity using the same base UUID for identifying that entity. Ideally I'd like to expose this via a library so for our end developers they could import a package, create a new instance of a "data exposer" and that data exposer would have a function takes in the uuid for the entity we are extending in the data exposer and we return an object which includes the new data to be exposed.

- The data is queriable behind a unified interface, so the extended types exposed by all the different services are aggregated and hidden from those querying the entities. This should allow for the hiding of what services produce what data. The aggregated type should be discoverable and explorable. The types should also be commentable 

- There should be query libraries which can be used to query entities and their data fields. These libraries should be typed based on the aggregated type from the different services. This should be created in such a way to allow queries from both human driven systems and from software systems.

- The query language should be built in such a way to allow users to only select the fields they care about when querying so that only the systems which drive those data fields will be called when generating the result of the query.

Some things to keep in mind:

- This is taking inspiration from the federated graphql idea pioneered by apollo graphql

- There will need to be a system to act as a "router" which does the magic of hiding consumer of data from the producer of data. This will be what aggreagtes types, routes queries for data, and constructs the final result from the different API responses, as well as doing high level error handling and metrics aggregation.

- Because we are supporting both pull and push for data ingestion the router will also need to listen to events sent from push based data sources and store that. Options should exist to allow push sources to work in different ways, for example whether the data aggregator should consider the first event as the most canonical, the last event, or whether it should combine the data from events in some way, like not overwriting values with null if a previous query has a non-null value and a subsequent query has a null value.