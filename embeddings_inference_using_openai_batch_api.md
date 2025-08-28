🚀 Leveling up our embedding pipeline with batch processing.

Context

Until now, embeddings for document chunks were created on the fly and pushed into MongoDB + Milvus in real time. While functional, this approach doesn’t scale gracefully when you’re dealing with large volumes of documents or running complex inference tasks.

Objective

I’m setting up a standalone pipeline built on the Prefect batch framework to handle both embeddings and inference in a modular, scalable way.

This pipeline will:
	1.	Batch Embeddings
	•	Read a list of target companies and their documents
	•	Identify unprocessed chunks in MongoDB
	•	Build batches and submit them to the OpenAI batch embeddings API
	•	Track job status (validating → in_progress → completed → failed/cancelled)
	•	Fetch the JSONL output, map embeddings back to their correct IDs, and insert into Milvus
	2.	Batch Inference
	•	Take a configurable list of companies and documents
	•	Apply task-specific prompts through modular inference functions
	•	Assign custom IDs to track outputs precisely
	•	Submit, monitor, and retrieve structured JSONL results
	•	Keep the system flexible enough to swap between embeddings and inference endpoints without rewriting the whole pipeline

Implementation Notes
	•	Prefect powers orchestration, scheduling, and monitoring
	•	All critical parameters (model, dimensions, API keys, connection strings) are configurable
	•	JSONL generator tailored so results directly map back to MongoDB and vector DB IDs
	•	Code organized into a main script and helpers for modularity and readability (no monolithic scripts)
	•	Multiple inference functions supported — each with its own prompts, logic, and tasks

References

To make this work reproducible, I’ve created two guidance files:
	1.	OpenAI Batch API Guide
	•	How to structure JSONL input/output for embeddings or inference
	•	How to create, submit, and monitor batch jobs
	•	Best practices for associating outputs with original document chunks
	2.	Prefect Implementation Guide
	•	Step-by-step design of the pipeline using Prefect tasks/flows
	•	How to keep configs modular (e.g., model names, dimensions, endpoints)
	•	How to integrate with MongoDB and Milvus cleanly
	•	Tips for testing with environment-specific configs before production

The end goal: a scalable, fault-tolerant batch system for both embeddings and inference that can grow with new use cases.

⸻

💡 Curious to hear from others:
	•	Are you using batch-first approaches for embeddings/inference?
	•	Or still leaning on real-time APIs for simplicity?
