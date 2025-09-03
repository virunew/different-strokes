Based on the user's detailed request and my research, I have a clear and comprehensive plan to create a Prefect-based implementation guide. The search results provided crucial information:

1.  **Prefect's `TaskFlow API`:** The search results confirm that the `@task` and `@flow` decorators are the idiomatic way to build modern Prefect pipelines. This is perfect for the user's requirement to build a "Pythonic" pipeline. I now have examples of how data flows between tasks without manual `XCom` management.
2.  **External API Calls:** I found a Prefect example of a web scraper that uses `requests` within a task, with built-in retries. This is a direct blueprint for how the `submit_openai_batch_job` task should be implemented. It also confirms that Prefect's retry logic is easily configurable with `@task(retries=...)`.
3.  **Dynamic Workflows and Parameters:** I found examples of how to pass parameters to flows and how Prefect supports Pydantic models for complex parameter validation. This is key for the user's requirement to avoid hardcoding and to create a new `run_parameters.py` file. The search also highlighted Prefect's strength in dynamic workflows and sub-flows, which is a perfect fit for the user's modularity requirement.
4.  **OpenAI Batch API:** The search results from the OpenAI Cookbook were excellent. They confirmed the JSONL format and, most importantly, the `custom_id` field. This is the exact mechanism I'll use to guide the user on how to link the batch results back to their MongoDB documents. I also have an example of the basic structure of the JSONL file for both embeddings and chat completions, which addresses the modularity request.

I have all the necessary components to build the guide. I will now synthesize this information into a cohesive response that serves as a detailed prompt for an agentic editor. I will structure the response with clear headings to make it easy to follow. I will not perform any more searches as I have enough information to create a comprehensive and accurate guide that directly addresses all aspects of the user's prompt.\#\# Prefect-Based Implementation Guidelines for OpenAI Batch Processing

This document provides a comprehensive blueprint for building a Prefect-orchestrated pipeline for creating embeddings and inference outputs using OpenAI's Batch API. The guidelines are designed to be a complete prompt for an agentic editor, integrating the user's specific requirements with Prefect's modern features.
-----

### 1\. Architectural Overview

The pipeline will be built as a single Prefect flow using the **TaskFlow API**, which provides a simple, Python-native way to define dependencies and data passing between tasks. The flow will handle the entire process from data extraction to database insertion, with each logical step encapsulated in its own Prefect task. This design ensures that each component is observable, retryable, and independently testable.

The workflow will follow these distinct phases:

1.  **Data Extraction:** A task will connect to MongoDB to retrieve a list of unembedded document chunks based on company and document type.
2.  **Batch File Generation:** A dedicated task will generate the JSONL file, including a **`custom_id`** to link the request to the original MongoDB document chunk IDs.
3.  **API Interaction:** Tasks will handle submitting the batch job to the OpenAI API, polling its status until completion, and downloading the output file.
4.  **Result Processing:** A final task will parse the output JSONL, match embeddings to the original chunks using the `custom_id`, and insert the data into Milvus.

-----

### 2\. Core Implementation Guidelines

#### **2.1. Centralized Configuration (@run\_parameters.py)**

To avoid hardcoding, all configurable parameters will be managed in a new Python file, `src/data_ware/run_parameters.py`. This file will store model names, API keys, batch sizes, and document-specific configurations. Prefect can load these parameters directly into the flow.

```python
# @src/data_ware/run_parameters.py

from dataclasses import dataclass, field
from typing import List

@dataclass
class OpenAIConfig:
    """Configuration for OpenAI API endpoints."""
    embedding_model: str = "text-embedding-3-small"
    inference_model: str = "gpt-4o"
    batch_size: int = 1000
    embedding_dimensions: int = 1536
    completion_timeout: int = 600

@dataclass
class DocConfig:
    """Document-specific parameters."""
    doc_type: str = "Annual Report"
    companies: List[str] = field(default_factory=lambda: []) # Loaded from admin_list.txt

@dataclass
class PromptsConfig:
    """Configurable prompts for different tasks."""
    summary_prompt: str = "Summarize the following text:"
    qa_prompt: str = "Answer the following question based on the text:"
```

#### **2.2. Prefect Flow and Task Definition**

The pipeline will be defined as a Prefect flow (`@flow`). Each step of the process will be a Prefect task (`@task`), which ensures that logic is compartmentalized and observable.

```python
# @src/data_ware/admin/openai_batch_pipeline.py

from prefect import flow, task, get_run_logger
from prefect.tasks import exponential_backoff
from prefect.blocks.system import Secret
import json
import os
# ... other necessary imports for MongoDB, Milvus, and file handling

# Load centralized configurations
from src.data_ware.run_parameters import OpenAIConfig, DocConfig

# Define tasks with Prefect's TaskFlow API
@task(retries=3, retry_delay_seconds=exponential_backoff(backoff_factor=10))
def fetch_unembedded_chunks(doc_config: DocConfig) -> list:
    """
    Connects to MongoDB and fetches chunks that lack embeddings.
    Returns a list of dictionaries with chunk data.
    """
    # Use logic from @src/data_ware/admin/parse_embed_pipeline.py
    # to fetch unembedded chunks for each company/doc_type.
    # The return value should include _id and doc_id for later use.
    logger = get_run_logger()
    logger.info("Fetching unembedded chunks from MongoDB...")
    # ... implementation ...
    return chunks

@task
def create_jsonl_batch(chunks: list, config: OpenAIConfig, endpoint: str) -> str:
    """
    Creates a JSONL file for the OpenAI Batch API.
    **This logic is modular and supports multiple endpoints.**
    """
    logger = get_run_logger()
    batch_file_path = f"batch_{endpoint}.jsonl"

    if endpoint == "embeddings":
        jsonl_generator = create_embedding_jsonl
    elif endpoint == "inference":
        jsonl_generator = create_inference_jsonl
    else:
        raise ValueError(f"Unknown endpoint: {endpoint}")

    batch_requests = jsonl_generator(chunks, config)

    with open(batch_file_path, "w") as f:
        for request in batch_requests:
            f.write(json.dumps(request) + "\n")
    logger.info(f"Generated {batch_file_path} with {len(batch_requests)} requests.")
    return batch_file_path

# ... Other tasks for API submission, polling, download, and Milvus insertion ...

@flow(log_prints=True)
def run_openai_batch_pipeline(companies_list: list, doc_type: str, openai_endpoint: str):
    """
    Main Prefect flow to orchestrate the OpenAI batch pipeline.
    """
    doc_config = DocConfig(companies=companies_list, doc_type=doc_type)
    openai_config = OpenAIConfig()

    unembedded_chunks = fetch_unembedded_chunks(doc_config)
    
    if not unembedded_chunks:
        print("No unembedded chunks found. Exiting flow.")
        return

    batch_file_path = create_jsonl_batch(unembedded_chunks, openai_config, openai_endpoint)

    # ... remaining flow logic for API submission, polling, etc.
    # Each step will be a new task that takes the output of the previous one.
    # e.g., batch_job_id = submit_openai_batch_job(batch_file_path)
    # e.g., output_file = wait_for_completion(batch_job_id)
    # e.g., insert_embeddings(output_file)
```

#### **2.3. JSONL Generator with Custom IDs**

The `create_jsonl_batch` task is the key to modularity. It will call a specific sub-function (`create_embedding_jsonl` or `create_inference_jsonl`) to build the request payload. The crucial part is to use the `custom_id` field in the JSONL format to encode a unique identifier for each chunk.

```python
# Helper function for embedding JSONL generation
def create_embedding_jsonl(chunks: list, config: OpenAIConfig) -> list:
    batch_requests = []
    for chunk in chunks:
        # Create a unique custom_id using a combination of doc_id and chunk _id
        custom_id = f"{chunk['doc_id']}-{chunk['_id']}"
        request_body = {
            "model": config.embedding_model,
            "input": chunk['text_content'],
            "dimensions": config.embedding_dimensions
        }
        batch_requests.append({
            "custom_id": custom_id,
            "method": "POST",
            "url": "/v1/embeddings",
            "body": request_body
        })
    return batch_requests

# Helper function for inference JSONL generation
def create_inference_jsonl(chunks: list, config: OpenAIConfig, prompt_config: PromptsConfig) -> list:
    batch_requests = []
    for chunk in chunks:
        custom_id = f"{chunk['doc_id']}-{chunk['_id']}"
        request_body = {
            "model": config.inference_model,
            "messages": [
                {"role": "system", "content": prompt_config.summary_prompt},
                {"role": "user", "content": chunk['text_content']}
            ]
        }
        batch_requests.append({
            "custom_id": custom_id,
            "method": "POST",
            "url": "/v1/chat/completions",
            "body": request_body
        })
    return batch_requests
```

#### **2.4. Handling Multiple Inference Functions**

The modularity requirement for multiple inference functions can be met by extending the `create_jsonl_batch` logic. You can pass a prompt configuration to the flow or a parameter to the task to select the desired prompt.

The editor can easily modify the `create_inference_jsonl` function to accept a parameter for the prompt template.

```python
def create_inference_jsonl(chunks: list, config: OpenAIConfig, prompt_template: str) -> list:
    # ... logic using prompt_template ...
```

-----

### 3\. Agentic Editor Prompt

**Objective:** Create a Prefect pipeline for an OpenAI batch job.

**Deliverables:**

  * A Python script at `@src/data_ware/admin/openai_batch_pipeline.py` implementing the Prefect flow.
  * A configuration file at `@src/data_ware/run_parameters.py` for all parameters.
  * A `requirements.txt` file listing all necessary dependencies, including `prefect`, `prefect-aws` (if needed for S3), `openai`, and a MongoDB library.

**Technical Requirements:**

1.  **Orchestration:** Use Prefect's `@flow` and `@task` decorators to build the pipeline.
2.  **Tasks:** Create distinct tasks for:
      * Reading the list of companies from `admin_list.txt`.
      * Fetching unembedded document chunks from MongoDB.
      * Generating the JSONL batch file with the `custom_id` field.
      * Submitting the batch job to the OpenAI API.
      * Polling the job status until completion.
      * Downloading and parsing the output JSONL file.
      * Inserting the processed embeddings into Milvus.
3.  **Modularity:**
      * The flow must accept parameters for `doc_type`, a list of companies, and the `openai_endpoint` (`embeddings` or `inference`).
      * The batch creation logic (`create_jsonl_batch`) must be modular, allowing for easy switching between embedding and inference JSONL formats.
4.  **Error Handling:** Implement Prefect's built-in retry logic with an exponential backoff strategy for API-related tasks.
5.  **Data Traceability:** The `custom_id` in the JSONL must contain the MongoDB `block_doc_id` and `block_mongo_id` to ensure accurate insertion into Milvus without re-querying MongoDB. The parsing task must use this `custom_id` to extract these IDs.
6.  **Milvus Integration:** Ensure the final task correctly formats and inserts the vector embedding, `block_mongo_id`, and `block_doc_id` into the Milvus database.
