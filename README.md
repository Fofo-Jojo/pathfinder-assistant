# pathfinder-assistant
Finding rules across all the Pathfinder 2e (Remaster) resources can be a huge pain; I'm making this AI assistant in hopes that I can reduce the time I spend flipping through books and searching through Archives of Nethys or Reddit online. If I can get that working well, I'd love if I could add the ability to do things like create enemies, NPCs, and encounters.

## Features
I have plans for the following features, broken up by minimum viable product and additions after that. This will help me focus on the features that matter first.

### Minimum Viable Product
* Chatbot interface
* Saved conversations that can be reopened and continued later
* Ability to find rules in following books:
    * Player Core
    * Player Core 2
    * GM Core
    * Monster Core

### Future Additions
* Ability to search Archives of Nethys (AoN) if books don't have answer
* Ability to search Reddit if books and AoN don't have answer
* Creators (if they don't work out of the box):
    * NPC
    * Enemies
    * Encounters
    * Civilizations
    * Settings
* Virtual DM

## Technologies
The plan is to use the following technologies to build this assistant.
* An open source LLM from huggingface as the language model
* A home brewed agent system as many open source ones never feel quite right
* A home brewed knowledge graph/vector database hybrid, as I haven't been satisfied with the open source ones
    * This will use one of the open source vectorizers from huggingface
    * It will also use an open source reranker 

### Basic Pipeline
1. User queries assistant
1. Query is sent to query cleaning agent that rewrites and clarifies query
1. Context identifying agent determines if any previous messages are relevant to cleaned query
1. Cleaned query and identified context is sent to planning agent that breaks the query into steps; creating an itemized query
    * Planning agent needs to identify steps that need to be done in serial vs. steps that can be parallelized; maybe via a Directed Acyclic Graph (DAG)?
1. Following the DAG from beginning to end
    1. Send the current node's query and the results of any previous steps to a routing agent
    1. Look through previous messages one more time to determine if any are relevant to current context?
    1. The routing agent will look at the query and any added context to determine what knowledge agent this should be sent to; example options
        * One agent for all books? Or separate?
            * Player Core 1 & 2
            * GM Core
            * Monster Core
        * Archives of Nethys
        * Reddit
    1. The response of the knowledge agent and the query will be sent to a checker agent to determine if the query was actually answered
    1. If the query wasn't answered, then another agent will be tried using the router agent again (add context for what knowledge agents have been tried)
    1. Repeat until query is answered or knowledge agent options are exhausted; if exhausted, final result will be "Answer not found" or similar
    1. Pass final result to next node
        * In between nodes there might need to be a collating agent that looks at all the previous steps in the current chain and formats the queries and results well for the next node
1. The last few nodes in the DAG should always be the following:
    1. Collating agent: takes all the final results and formats them well, documents everything
    1. Checker agent: makes sure that result of collating agent actually appears to contain answer to initial query (what happens if not?)
    1. Summarizer agent: Takes collating agents very detailed answer and extracts answer into shortest possible version; would just pass collating agent response if it determines it can't reduce it further
    1. Checker agent: makes sure summarizer agent result actually appears to contain answer to initial query

### Knowledge Graph/Vector DB hybrid

#### Vector Databases
Vector databases are an alternative search solution to simple "Search for occurrences of \<list of words\>". Vector databases allow for a more semantic/meaning based search by embedding a query string into a vector of floating point numbers representing it's meaning. That vector can then be compared to vector embeddings of chunks of text in the database. By selecting the most similar vectors, you have selected the chunks of text that are potentially the most relevant to the query.

Pros:
* Easy
* Fast

Cons:
* Embedders often only support smaller chunks of text
* Lossy compression, so even if larger chunks are supported information relevant to text might not make it into vector
* Not great when the answer is spread across multiple chunks of text
* Existing vector DBs have no built in hierarchy or connections between chunks of text

#### Knowledge Graphs
Knowledge graphs are another alternative search solution. They are represented as graph composed of nodes and edges. The nodes represent specific pieces of knowledge, while the edges represent relationships between the nodes. Once you've found a seed node (or nodes) for a query you can iteratively check for other relevant nodes via the relationships.

Pros:
* Provide ability to search both depth and breadth of knowledge
* Finds answers more reliably if graph is built well

Cons:
* Very time consuming to build a full graph
    * Especially if very granular level of inforation is sought

#### Hybrid
I believe a hyrbid solution could work quite well. Where vectors are used to find seed nodes, then relationships can be used to traverse the graph. Instead of vectorizing small chunks of text, summaries of the node's content could be vectorized. However, instead of simply returning the summary the full content could be returned. This is made possible by new LLMs supporting larger and larger contexts.

Let's explore what the knowledge graph for a multibook agent might look like and how to build it.
1. Start by establishing a hierarchy based on books, chapters, sections, subsections, ..., and paragraphs
    * These would establish any parent, child relationships
1. Starting from lowest desired level and working upward
    * If lowest level
        * Create a summary that fits in embedding model if necessary
    * Otherwise
        * Establish child relationships
        * Create a summary from child node summaries; accounting for order
    * Create vector from summary
    * Establish parent relationships
    * Establish sibling relationshsips; accounting for order (next vs. previous)
    * Store embedding, summary, content, and relationships in nodes
1. Once the initial hierarchy is created, establish other types of relationships
    * Human in the loop likely required, if not all human
    * Use summary vectors to identify list of potentially related nodes; be very loose with matching criteria here, want to start with many
    * Prune any existing child/parent or sibling relationships from the list
    * Use LLM to do some pre-filtering of whether or not potentially related nodes are actually related (lean towards higher false positive rate)
    * Have a human look through remaining pairs to identify and prune remaining false negatives

Once the hybrid vector DB/knowledge graph is built it can be searched by following this workflow (WIP):
1. Seeding: Vectorize/embed query and search for N closest vectors within knowledge graph; aiming to create knowledge "seeds" so N should be vary large.
1. Filtering 1: Use a reranker to compare query and summaries or content; select top M ranked seeds, where M is much smaller than N.
1. Filtering 2: Ask an LLM if seed's content is relevant to query.
1. Expand: Create list of related nodes (do not include already visited nodes)
1. Filtering 3: Compare vectorized query to vectors of related nodes and select top N again
1. Repeat filtering and expansion steps until leads die out; if designed well there should be a decay rate such that only a small portion of the graph should be explored.