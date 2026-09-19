# AI Solutions with azure data management services
focus on how AI application store data, store embeddings, retrieve semantic information and cache results

| Service                           | Main purpose                             | Vector capability       |
| --------------------------------- | ---------------------------------------- | ----------------------- |
| **Azure Cosmos DB for NoSQL**     | Globally distributed NoSQL/JSON database | Vector search + DiskANN |
| **Azure Database for PostgreSQL** | Relational SQL database                  | pgvector + HNSW         |
| **Azure Managed Redis**           | Very fast in-memory data store/cache     | RediSearch + FLAT/HNSW  |

## Embedding and Vector Search

### 1.1 what is embadding?
its convert something such as text into list of numbers called a vector, actual vector produced can contain hundreds or thousands of dimensions.

example -> "I love cats" -> vector [0.12, -0.44, 0.81, 0.32, ...]
(the important in not individual numbers, its their position relative to other vectors)

Simple idea

Imagine:

"I like dogs"       → Vector A
"I love puppies"    → Vector B
"How to repair SQL" → Vector C

Vectors A and B should be relatively close together because their meanings are similar.Vector C should be farther away.

Therefore:
Embedding = numerical representation of meaning.
This is what allows an application to search based on meaning, rather than just exact keywords.

### 1.2 Keyword search vs semantic search

Suppose a user searches:
"How can I reduce my Azure bill?"

Your database contains:
"Methods for lowering cloud computing costs."

A traditional exact keyword search may struggle because the wording is different.
A vector search can recognize that:

reduce bill ≈ lowering costs
because their embeddings are similar.

Remember

Traditional search
Words → match words

Vector search
Meaning → match meaning

### 1.3 Vector Similarity
once the data represents as vector, it need to determine how close two vectors are

common measuremnets:
| Metric             | Meaning                   |
| ------------------ | ------------------------- |
| **Cosine**         | Compares direction/angle  |
| **Euclidean / L2** | Straight-line distance    |
| **Inner product**  | Measures vector alignment |

### 1.4 Basic Vector Search process
memorizing with 4 steps:
document -> embedding model -> vector -> store vector + original data

when user search:
user question -> embedding model -> query vector -> vector DB -> similarity search -> most similar documents

*important details* - both store documents and user query must be convert into vector using embedding process

## Retrieval - Augmented generations (RAG)

### 2.1 What is RAG?
instead of asking LLM to answer entirely from what its learn, LLM first retrieves information from own data. Information supplied to the LLM known as *context*

Retrieve first -> Generate second

### Why use RAG?
LLM may not know latest or updated details sometimes

we take example for internal employee handbook in a company, instead of retraining the model, the company can:

company documents -> split into chunks -> generate embeddings -> store vectors

when employee asks "how many annual leave days do I have?"

the system performs:
question -> create embedding -> vector search -> retrieve relevant handbook chunks -> add chunks to LLM prompt -> LLM generate answer

### The 4 exam steps of RAG

step 1 - embedded the questions
"what is our refund policy?" -> embedding model -> [0.18, 0.75, ...]

step 2 - search the vector store
Find the top-N / top-k vectors most similar to the question.

step 3 - Construct the prompt
context [retrieved company information]
question "What is our refund policy"

step 4 - Call the LLM
LLM use retrieved context to construct the answer
memory trick - [E -> S -> P -> G]
Embed → Search → Prompt → Generate

### 2.4 Metadata filtering
vector similarity isnt always enough

imagine company store documents for many customer:
Document A → Tenant A
Document B → Tenant B
Document C → Tenant A

if tenant A asks a question, we dont want document from tenant B appeared. Therefore we can combine *vector + metadata filter*

example: WHERE tenant = "TenantA"
then perform similarity search

other metadata filter might include: date, category, user, department, document type. 

## Azure Cosmos DB for NoSQL

### 3.1 What is Cosmos DB
cosmos DB is a globally distributed NoSAL database design for low-latency applications. It stores JSON-like documents called items.

the hierarchy is important: Cosmos DB Account -> DB -> container -> Items

example:
{
    "id": "101",
    "category": "AI",
    "title": "Introduction to RAG",
    "embedding": [0.12, 0.81, 0.33]
}

therefore, Cosmos DB can store *normal app data, metadata and embeddings* together

### 3.2 Partition Keys
a partition key determines how cosmos DV distributes data (its like primary keys). Imagine 1 millions documents, Cosmos DH doesnt what all of them concentrated in one locationl A partition keys helps distributed the data and loaded.

good partition key should generally have:
High cardinality + even distribution
good example: customerId, userId, tenantId
bad example: active, inative

### 3.3 Hot Partitions

example:
Partition A → 90% of requests
Partition B → 5%
Partition C → 5%

so, partition A becomes a hot partition. That can cause throttling.
exam materials associates throttling with: HTTP 429

tricky point: even a DB have unused RU capacity, a hot partition can still cause throtting
remember: bad partition key -> uneven traffic -> hot partition -> 429 throttling

### Important Partition key rule
memorize: The partition key cannot simply be changed after container creation. 

exam wording: Which database configuration should be planned carefully before creating a Cosmos DB container?
think: Partition key

## Cosmos DB Request Units (RUs)
Cosmos DB measure database operation cost using request units (RU)
example of operations consume RUs: read, write, update, query, vector search

## 4.1 Point reads
important in exam: point read are cheaper than queries

point read knows both: id + partition key
so Cosmos DB knows exactly where the items is.

conceptually:
"Give me item 100
where partition = Customer7"

much easier than:
"Search all documents for items matching these conditions."

memorize:
Lowest-cost Cosmos DB lookup → point read using ID + partition key.

## 4.2 What increase RU consumption?

Larger documents require more processing: larger items -> more RUs

Queries: generally cost more than point reads

Cross-partition queries: searching multiple partitions requires more work, therefore when possible include *the partition key in the query*.

Excessive indexing:
More indexing -> More index maintenance -> Higher write RU

## Cosmos DB indexing
by default Cosmos DB indexes properties automatically. it make query convinient but indexing everything isnt always necessary. we can exclude unused paths.

example
/document
/title
/category
might need indexing, but perhaps
/hugeRawContent
doesnt.

exclude unused path can reduce write RU and storage requirements

### 5.1 Composite indexes
can improve involving multiple properties, particularly multi-propery sorting / query patterns

general exam principle:
Index according to the application's query patterns

not:
Index absolutely everything.
