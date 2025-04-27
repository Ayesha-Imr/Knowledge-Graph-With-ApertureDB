Project details:

Idea and Implementation Plan

Overview:
I want to build a technical blog showcasing how to automatically create a Knowledge Graph using ApertureDB and LLMs. The core goal is to let users submit raw data (e.g., text) and have it automatically parsed into entities and relationships to create a structured knowledge graph without manual intervention.

Key Use Case:
User submits raw data (e.g., text).


An LLM (e.g., GPT-4) extracts entities and relationships from the text, creating a structured JSON output:


Entities: People, Places, Companies, Jobs, etc.


Relationships: Works-At, Lives-In, Manages, etc.


Based on this output, ApertureDB is used to create nodes (entities) and edges (relationships).


Optional: Graph visualization to showcase the result.



Detailed Plan & Steps:

1. Entity Extraction
Global Entity List Extraction:

Use the full text to extract all entities globally (e.g., Person, Location, Company) along with all their possible properties (which are optional so each instance of that entity can have one, none or some properties defined).


Entities are not assigned IDs here; IDs will be assigned later.

Format for entity extraction:
  
  [
    {"Entity1_name": ["property1", "property2", "property3"]},
    {"Entity2_name": ["property1", "property2", "property3"]},
    {"Entity3_name": ["property1", "property2"]},...,
    {"EntityN_name": ["property1", "property2", "property3",..., "propertyN"]}
  ]

Example output for global entity extraction:

{
  "Person": ["name", "age", "email"],
  "Company": ["name", "industry", "email"],
  "Location": ["address", "latitude", "longitude"]
}


2. Generate Instances of each entity in parallel
Split the input text into logical chunks (e.g., paragraphs or 1,000-1,500 characters max per chunk).


Global entity JSON context (result from Step 1) is passed along with each chunk to respective entity extraction agents. These agents will work in parallel to extract instances of entities (e.g., John Doe, Acme Corp, New York) and their properties. For example, 5-6 chunks can be processed at the same time depending on how much parallelism can be applied. The agent working on a chunk consults the list of available entities and the chunk text, it then creates a list of dicts where each dict corresponds to an entity and contains all its instances with their (available) properties defined.

Format:
[ 
{ “Entity”: “Entity1_name”, “Instances”: [
  {"property1": "property1_value", "property2": property2_value", "property3": property3_value"},
  {"property1": "property1_value", "property2": property2_value"},...]
},
{ “Entity”: “Entity2_name”, “Instances”: [
  {"property1": "property1_value", "property2": property2_value", "property3": property3_value"},
  {"property1": "property1_value", "property2": property2_value"},...]
},...,
{ “Entity”: “EntityN_name”, “Instances”: [
  {"property1": "property1_value", "property2": property2_value", "property3": property3_value", "property4": property4_value"},
  {"property1": "property1_value", "property2": property2_value"},...]
}
]


Example output for entity instances extraction :
[ 
{ “Entity”: “Person”, “Instances”: [
  {"name": "John Doe", "age": 30, "email": "john@example.com"},
  {"name": "Jane Smith",email": "jane@example.com"}]
},
{ “Entity”: “Company”, “Instances”: [
  {"name": "Google", "industry": “Software Development”, "email": "google@google.com"},
  {"name": "Microsoft", "industry": “Software Development”}]
}
]

3. Duplicate Removal & ID Assignment (Parallel)
Since multiple agents may have created the same instance due to it being mentioned in different chunks this step will check the output from Step 2 for duplicate instances and remove them, retaining the instance with the highest number of defined properties.
ID Assignment: This step will also be responsible for assigning unique IDs to each instance of the entities (e.g., assigning a person_id, company_id, etc.).
This step ensures that all entity instances are uniquely identified before relationships are extracted.
We'll use globally unique UUIDs for this purpose. The UUIDs will be generated using the uuid4() function from the uuid library in Python.
This step will not use any LLMs, but will use a simple algorithm to check for duplicates and assign IDs. The algorithm will check for duplicates based on the properties of the instances (e.g., name, email) and will keep the instance with the most defined properties.
For example, if we have:
  
  [ 
  { “Entity”: “Person”, “Instances”: [
    {"name": "John Doe", "age": 30, "email": "john@example.com"},
    {"name": "John Doe", "email": "john@example.com"},...]
  },...
  ]

then the result would be:

 [ 
  { “Entity”: “Person”, “Instances”: [
    {"id" : "unique_UUID", "name": "John Doe", "age": 30, "email": "john@example.com"},...]
  },...
  ]

and if we have:

  [ 
  { “Entity”: “Person”, “Instances”: [
    {"name": "John Doe", "age": 30, "email": "john@example.com"},
    {"name": "John Doe", "email": "john@example.com", "address": "London"},...]
  },...
  ]

  then the result would be:

  [ 
  { “Entity”: “Person”, “Instances”: [
    {"id" : "unique_UUID", "name": "John Doe", "age": 30, "email": "john@example.com", "address": "London"},...]
  },...
  ]


At the end, the output will be merged to create a list of dicts where each dict corresponds to an entity and contains all its unique and uniquely identifiable instances with their (available) properties defined.



Example output for duplicate removal and ID assignment (for Person class):
[ 
{ “Entity”: “Person”, “Instances”: [
  {"id": "unique_UUID", "name": "John Doe", "age": 30, "email": "john@example.com"},
  {"id": "unique_UUID", "name": "Jane Smith", "email": "jane@example.com"}]
},
{ “Entity”: “Company”, “Instances”: [
  {"id": "unique_UUID", "name": "Google", "industry": “Software Development”, "email": "google@google.com"},
  {"id": "unique_UUID", "name": "Microsoft", "industry": “Software Development”}]
}
]



4. Global Relationship Extraction (Full Data Pass)
After extracting all entities, perform a single global LLM pass (using the full input text + the list of all entities with all their instances - the result of step 3) to extract relationships between entities.
Relationships will be extracted between instances of the same class or across different classes (e.g., John Doe — works_at → Acme Corp).


Example output for relationship extraction:
[
  {"relationship": "works_at", "source": {"class": "Person", "id": "unique_UUID"}, "destination": {"class": "Company", "id": "unique_UUID"}},
  {"relationship": "lives_in", "source": {"class": "Person", "id": "unique_UUID"}, "destination": {"class": "Location", "id": "unique_UUID"}}
]


4. Create Knowledge Graph in ApertureDB
Use ApertureDB Python SDK to:


Create entities (nodes) in the graph.


Create relationships (edges) between entities.


No LLM required for this step - query templates can be populated with values from JSON outputs from the previous steps
Optionally, visualize the graph using libraries like networkx or matplotlib.



