# **Build a RAG Chatbot in n8n (Step-by-Step Lab)**

This hands-on lab walks you through building a Retrieval-Augmented Generation (RAG) chatbot in **n8n** that:

* Watches a **Google Drive** folder for new files

* Splits and embeds the content with **OpenAI Embeddings**

* Stores and retrieves vectors from **Pinecone**

* Answers questions through an **LLM** (via **OpenRouter**) with retrieved context

* Exposes a **chat webhook** you can call from any UI

Assumptions: You have your documents locally (you’ll upload them to Google Drive during the lab).

---

## **0\) Prerequisites**

* **n8n** instance (cloud or self-hosted) and access to the n8n UI.

* Accounts and API keys:

  * **Google** (Drive access)

  * **OpenAI** (for embeddings)

  * **OpenRouter** (for chat model; you can also use OpenAI here if you prefer)

  * **Pinecone** (index created)

* Pinecone **index dimension** must match your embedding model

  * `text-embedding-3-small` → 1536

  * `text-embedding-3-large` → 3072

**Create a Google Drive folder** for this lab and note its **Folder ID** (from the URL).

---

## **1\) Create Credentials in n8n**

In **n8n → Settings → Credentials → New**:

1. **Google Drive OAuth2 API**

   * Name: `Google Drive Trigger`

   * Authenticate with your Google account and grant Drive access.

2. **OpenAI API**

   * Name: `OpenAi account`

   * API key: paste your OpenAI key.

3. **OpenRouter API**

   * Name: `OpenRouter account`

   * API key: paste your OpenRouter key.

4. **Pinecone API**

   * Name: `RagTest`

   * API key \+ environment (and project, if your Pinecone setup requires it).

---

## **2\) Create the Ingestion Workflow (Drive → Split → Embed → Pinecone)**

We’ll build a linear flow that fires when a new file appears in Drive.

### **A) Add nodes (with settings)**

1. **Google Drive Trigger**

* **Trigger on**: File created

* **Folder**: paste your Drive Folder ID

* **Polling**: Every minute (or set a cron)

* **Credentials**: `Google Drive Trigger`

2. **Google Drive** (Downloader)

* **Operation**: Download

* **File ID**: `={{ $json.id }}`

* **Credentials**: `Google Drive Trigger`

* **Connect**: **Google Drive Trigger → Google Drive (Download)**

3. **Recursive Character Text Splitter**

* **Chunk size**: `1000`

* **Chunk overlap**: `200`

* This node prepares chunking for the loader. You’ll connect it to the loader’s “splitter” input.

4. **Default Data Loader**

* **Data type**: Binary

* **Text splitting mode**: Custom

* **Connect**:

  * **Recursive Character Text Splitter → Default Data Loader** (splitter port to splitter port)

  * **Google Drive (Download) → Default Data Loader** (Main → Main)

5. **Embeddings OpenAI**

* **Model**: pick your embedding model (e.g., `text-embedding-3-small`)

* **Credentials**: `OpenAi account`

6. **Vector Store: Pinecone** (Insert mode)

* **Mode**: Insert

* **Index**: your Pinecone index (e.g., `ragtest`)

* **Namespace**: `FAQ` (or another logical name)

* **Credentials**: `RagTest`

* **Connect**:

  * **Default Data Loader → Pinecone (ai\_document → ai\_document)**

  * **Embeddings OpenAI → Pinecone (ai\_embedding → ai\_embedding)**

  * **Google Drive (Download) → Pinecone (Main → Main)**

At this point, dropping a file into your Drive folder should download, split, embed, and insert into Pinecone.

---

## **3\) Create the Chat Workflow (Webhook → Retrieval Tool → LLM Agent)**

We’ll add a simple chat entrypoint that can answer with retrieved context.

1. **When chat message received** (Trigger)

* Use defaults. When you save the workflow, n8n will show **Test** and **Production** webhook URLs.

2. **Vector Store: Pinecone** (Retrieve as tool)

* **Mode**: Retrieve as tool

* **Tool description**: “Use this to search the knowledge base.”

* **Index**: same as above (e.g., `ragtest`)

* **Namespace**: `FAQ` (must match ingestion)

* **Credentials**: `RagTest`

* **Embeddings**: add another **Embeddings OpenAI** node (or reuse the same model) and connect **ai\_embedding → ai\_embedding**.

3. **OpenRouter Chat Model**

* **Model**: for example `anthropic/claude-3-5-sonnet` (or your preferred model that your key allows)

* **Credentials**: `OpenRouter account`

4. **AI Agent**

* Leave defaults. This node will use:

  * The **LLM** you connect (OpenRouter Chat Model)

  * The **Pinecone “Retrieve as tool”** you connect

**Connect:**

* **When chat message received → AI Agent** (Main → Main)

* **OpenRouter Chat Model → AI Agent** (ai\_languageModel → ai\_languageModel)

* **Pinecone (Retrieve as tool) → AI Agent** (ai\_tool → ai\_tool)

Save the workflow.

---

## **4\) Test the Ingestion**

1. Upload a small PDF or text file to your **Google Drive folder**.

2. In n8n, open **Executions** for this workflow and verify steps:

   * Trigger fired

   * File downloaded

   * Chunks created

   * Embeddings computed

   * Vectors inserted into Pinecone (namespace `FAQ`)

Check Pinecone’s console to see the namespace and vector count increase.

---

## **5\) Test the Chat**

1. Open the **When chat message received** node → copy the **Test URL**.

2. Use curl or Postman to send a simple chat payload:

curl \-X POST "\<YOUR\_TEST\_WEBHOOK\_URL\>" \\  
  \-H "Content-Type: application/json" \\  
  \-d '{"message":"What does the FAQ say about refunds?"}'

3. In n8n **Executions**, verify:

   * The Agent calls the **Pinecone Retrieval tool**

   * The LLM returns an answer grounded in your ingested content

4. When ready, toggle the workflow **Active** and switch to the **Production** URL for your real clients/UI.

---

## **6\) Troubleshooting**

* **No vectors in Pinecone**

  * Check Pinecone index **dimension** matches your embedding model.

  * Confirm the **Insert** node uses the same **namespace** you plan to query.

* **Drive trigger not firing**

  * Re-auth the Google credential and ensure the **Folder ID** is correct.

  * Try dropping a different file type (TXT, PDF) to test.

* **Poor retrieval quality**

  * Tune chunking: try **Chunk size 800–1200**, **Overlap 150–250**.

  * Ensure documents are text-extractable; if scanned PDFs, run OCR upstream.

* **LLM does not use the tool**

  * Add a brief system instruction in the **AI Agent** or **Chat Model**:  
     “If the user asks about the knowledge base, call the retrieval tool.”

* **Timeouts or auth errors**

  * Verify your **OpenRouter** key has access to the selected model.

  * Ensure **OpenAI** and **Pinecone** credentials are valid and not rate-limited.

---

## **7\) Optional: Import From JSON (Fast Path)**

If you have a prebuilt workflow JSON:

* In n8n: **Workflows → Import from file** → select your JSON

* After import, open each node and:

  * Re-select your **Credentials**

  * Update **Folder ID**, **Pinecone Index/Namespace**, and **model names**

* Save and test as above

---

## **8\) Hardening & Next Steps**

* Add a **metadata enricher** node to store doc titles/URLs with vectors and include them in answers.

* Add **filters** to restrict retrieval to specific tags or sources.

* Add a **rate limiter** or **auth check** in front of the chat webhook.

* Instrument usage by logging questions, tool calls, and citations to a database.

---

### **Completion Checklist**

* Google Drive trigger created and authorized

* Files download and split successfully

* Embeddings generated with the correct model

* Vectors upserted to Pinecone (right index and namespace)

* Chat webhook responds with grounded answers

* Workflow toggled **Active** with the **Production** webhook URL

You now have a working n8n RAG chatbot that ingests from Google Drive and answers with Pinecone-backed context.

