Here’s a clean, copy-pasteable plan to run Batch Embeddings with OpenAI. It covers the JSONL format, Python and Node.js code, polling, downloading results, and parsing the output correctly—plus the exact docs to point your agent at.



0) One-time setup
	•	Create an API key and set it as OPENAI_API_KEY in your environment.
	•	Install the SDK:
	•	Python: pip install openai
	•	Node: npm i openai

Model tip: use text-embedding-3-small for speed/cost or text-embedding-3-large for higher quality. Check the model card for current dims/context.  ￼

Batch basics: Batch runs your requests asynchronously from a JSONL file and writes results to an output file at a discounted price, typically within 24 hours.  ￼

⸻

1) Prepare the JSONL input (the critical bit)

Each line is an independent HTTP request specification:

{"custom_id":"doc-0001","method":"POST","url":"/v1/embeddings","body":{"model":"text-embedding-3-small","input":"Your first text chunk here"}}
{"custom_id":"doc-0002","method":"POST","url":"/v1/embeddings","body":{"model":"text-embedding-3-small","input":"Your second text chunk here"}}

Notes that save headaches:
	•	Keep one logical item per line so result mapping is trivial.
	•	custom_id is required if you want easy, stable mapping on the way out. The output order is not guaranteed to match the input—use custom_id.  ￼
	•	The url must start with /v1/embeddings for embeddings batches.  ￼

Reference: Batch guide and API reference.  ￼

⸻

2) Python: create the file, upload, start the batch, poll, download, parse

# python >=3.8
from openai import OpenAI
import json, time, pathlib

client = OpenAI()  # uses OPENAI_API_KEY

# 2.1 Write JSONL
requests = [
    {"id": "doc-0001", "text": "Your first text chunk here"},
    {"id": "doc-0002", "text": "Your second text chunk here"},
]

input_path = pathlib.Path("embeddings.jsonl")
with input_path.open("w", encoding="utf-8") as f:
    for r in requests:
        line = {
            "custom_id": r["id"],
            "method": "POST",
            "url": "/v1/embeddings",
            "body": {
                "model": "text-embedding-3-small",
                "input": r["text"],
                # Optional: request float arrays instead of base64 strings
                # "encoding_format": "float",
                # Optional: pick dimensions if supported by your model
                # "dimensions": 1536
            },
        }
        f.write(json.dumps(line, ensure_ascii=False) + "\n")

# 2.2 Upload the batch input file
up = client.files.create(
    file=open(input_path, "rb"),
    purpose="batch",
)

# 2.3 Kick off the batch job
batch = client.batches.create(
    input_file_id=up.id,
    endpoint="/v1/embeddings",
    completion_window="24h",
    # metadata={"job": "my-embeddings-run"}  # optional
)

print("Batch started:", batch.id)

# 2.4 Poll until done (or failed/expired/canceled)
TERMINAL = {"completed", "failed", "expired", "canceled"}
while True:
    b = client.batches.retrieve(batch.id)
    print("Status:", b.status)
    if b.status in TERMINAL:
        break
    time.sleep(30)

if b.status != "completed":
    raise RuntimeError(f"Batch not completed. Final status: {b.status}")

# 2.5 Download results file (JSONL)
out_file_id = b.output_file_id
resp = client.files.content(out_file_id)
text = resp.text  # contents of the JSONL results
out_path = pathlib.Path("embeddings_output.jsonl")
out_path.write_text(text, encoding="utf-8")

print("Wrote:", out_path)

# 2.6 Parse results into a dict of {custom_id: embedding}
id_to_embedding = {}
with out_path.open("r", encoding="utf-8") as f:
    for line in f:
        obj = json.loads(line)
        # On success, embeddings are in obj["response"]["body"]["data"][0]["embedding"]
        if obj.get("error"):
            # Capture failed line details if needed
            print("Failed:", obj["custom_id"], obj["error"])
            continue

        body = obj["response"]["body"]
        emb = body["data"][0]["embedding"]
        id_to_embedding[obj["custom_id"]] = emb

print("Total embeddings:", len(id_to_embedding))

Docs:
	•	Batch API (overview and statuses).  ￼
	•	API reference: Files (upload & download).  ￼
	•	Cookbook example (retrieving results with files.content).  ￼
	•	Embeddings API reference (request schema).  ￼

⸻

3) Node.js: same flow

// Node 18+, "type": "module" or use require(...)
import OpenAI from "openai";
import fs from "node:fs/promises";

const client = new OpenAI(); // uses process.env.OPENAI_API_KEY

// 3.1 Write JSONL
const rows = [
  { id: "doc-0001", text: "Your first text chunk here" },
  { id: "doc-0002", text: "Your second text chunk here" },
];

const jsonl = rows.map(r => JSON.stringify({
  custom_id: r.id,
  method: "POST",
  url: "/v1/embeddings",
  body: {
    model: "text-embedding-3-small",
    input: r.text,
    // encoding_format: "float",
    // dimensions: 1536
  }
})).join("\n");

await fs.writeFile("embeddings.jsonl", jsonl, "utf8");

// 3.2 Upload file
const file = await client.files.create({
  file: await fs.readFile("embeddings.jsonl"),
  purpose: "batch",
});

// 3.3 Create batch
const batch = await client.batches.create({
  input_file_id: file.id,
  endpoint: "/v1/embeddings",
  completion_window: "24h",
});

console.log("Batch started:", batch.id);

// 3.4 Poll
const TERMINAL = new Set(["completed","failed","expired","canceled"]);
let status = "validating";
while (!TERMINAL.has(status)) {
  const b = await client.batches.retrieve(batch.id);
  status = b.status;
  console.log("Status:", status);
  if (!TERMINAL.has(status)) {
    await new Promise(r => setTimeout(r, 30000));
  } else {
    if (status !== "completed") throw new Error(`Batch ended: ${status}`);
    // 3.5 Download results
    const out = await client.files.content(b.output_file_id);
    const text = await out.text();
    await fs.writeFile("embeddings_output.jsonl", text, "utf8");
  }
}

// 3.6 Parse results
const lines = (await fs.readFile("embeddings_output.jsonl","utf8")).trim().split("\n");
const idToEmbedding = new Map();
for (const line of lines) {
  const obj = JSON.parse(line);
  if (obj.error) {
    console.warn("Failed:", obj.custom_id, obj.error);
    continue;
  }
  const emb = obj.response.body.data[0].embedding;
  idToEmbedding.set(obj.custom_id, emb);
}
console.log("Total embeddings:", idToEmbedding.size);

Docs (same): Batch, Files, Embeddings reference; Cookbook for files.content.  ￼ ￼

⸻

4) Operational guardrails (learned the hard way)
	•	Pricing & SLA: Batch is discounted and typically completes within 24 hours. If a batch expires, you still get the partial results and are charged for completed work.  ￼
	•	Statuses you’ll see: validating → in_progress → finalizing → completed (or failed/expired/canceled). Your code above handles terminal states.  ￼
	•	Order isn’t preserved: always map by custom_id in the output.  ￼
	•	URL must match endpoint: for embeddings use /v1/embeddings, or you’ll get the “URL does not prefix-match the batch endpoint” error.  ￼
	•	Result payload shape: Each output line echoes your custom_id and includes response.body.data[0].embedding on success, or an error field on failure. (See cookbook snippet on reading the output file.)  ￼
	•	Optional: If you want direct float arrays instead of base64, set encoding_format: "float" in the embeddings body; otherwise you may receive base64 and need to decode. This behavior varies by SDK and has changed over time—test in your environment.  ￼

⸻

5) Where to point your agent (official docs)
	•	Batch API — overview & reference (how batches work, statuses, limits, discount):  ￼ ￼
	•	Embeddings API reference (request/response schema):  ￼
	•	Files API reference (upload input file, download output file):  ￼
	•	Cookbook: Batch processing (practical end-to-end example, including retrieving results):  ￼

⸻

References : 

https://platform.openai.com/docs/guides/batch/overview

https://platform.openai.com/docs/api-reference/embeddings

https://platform.openai.com/docs/api-reference/files

https://cookbook.openai.com/examples/batch_processing_with_openai
⸻
