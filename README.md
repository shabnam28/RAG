# RAG
I will implement a Retrieval-Augmented Generation (RAG) application in this repository.
## What is RAG?

RAG stands for Retrieval-Augmented Generation. It’s a technique that combines:
- Retrieval – Querying a large database, documents, or knowledge base to find relevant information.
- Generation – Using a language model (like GPT, Groq, Gemini-flash) to create an answer based on the retrieved information.

Instead of the AI just “guessing” or relying on its memory, RAG ensures the output relies on the company's real sources.
<img width="876" height="375" alt="Screenshot 2025-11-20 at 10 22 47" src="https://github.com/user-attachments/assets/9b857c34-c139-464b-ab67-905d1b91ce58" />

### Example
User ask : “What is our refund policy?”
With RAG: The system retrieves your internal refund policy document and The LLM AI model generates a response directly from that document, giving an exact and reliable answer.

## RAG Workflow Steps

### 1. Data Collection / Ingestion: 
Collect all the sources like (PDFs, Word documents, HTML pages, emails, spreadsheets, internal knowledge bases, databases)  that the **RAG** system will use:

A Power Automate flow is triggered whenever a new application email arrives at the `hello@...` inbox. The flow performs the following actions:
 
1. **Trigger:** New email received at the `hello@...` address
2. **Store attachment:** Saves the received application .pdf/ .docx file into **Azure Blob Storage**
3. **Update Candidate Table (SQL Database):** Inserts a new record with:
   - **File name** – name of the received application file
   - **Inserted date** – timestamp when the record was added
   - **Email subject** – subject line of the received email
   - **To email address** – the internal recipient address the application was sent to (e.g., `hello@...`)
**Output of this step:** Raw application files stored in Blob Storage + a SQL record tracking metadata for each application in Candidate_Applications_TABLE.
**Table Schema: `Candidate_Applications_TABLE`**

| Column | Type | Nullable | Description |
|---|---|---|---|
| `id` | `int` | No | Primary key / auto-generated record ID |
| `candidate_filename` | `nvarchar` | No | Name of the received application file |
| `inserted_at` | `datetime` | No | Timestamp when the record was inserted |
| `application_title` | `nvarchar` | No | Subject line of the received email |
| `email` | `nvarchar` | No | From email address |
 
---

### 2. Data Parsing

LLMs cannot read PDFs or Excel files directly so we are using Loaders to convert raw files into text that the LLM model can understand them.  Document loaders provide a standard interface for reading data from different sources (such as Slack, Notion, or Google Drive) into LangChain’s Document format. This ensures that data can be handled consistently regardless of the source.
All document loaders implement the BaseLoader interface.

#### 2.1 Tools Examples:
| Loader Type       | Example                 | Notes                                   |
| ----------------- | ----------------------- | --------------------------------------- |
| PDF Loader        | `PyPDFLoader`           | Reads PDF pages into text               |
| DOCX Loader       | `Docx2txtLoader`        | Extracts text from Word documents       |
| TXT Loader        | `TextLoader`            | Simple plain text                       |
| CSV Loader        | `CSVLoader`             | Reads specific columns as text          |
| JSON Loader       | `JSONLoader`            | Parses JSON fields as text              |
| Web / HTML Loader | `UnstructuredURLLoader` | Fetches webpage content and cleans HTML |
| Email Loader      | `EmailLoader`           | Extracts email subject/body             |

An **Azure Function** with an **Event Grid trigger** listens for new blobs across several containers and routes processing based on which container/file type triggered the event.
 
#### 2.2 **Trigger:** New blob created in the `cvfiles` container, with a `.pdf` extension.
 
#### 2.3 **Flow:**
1. The Event Grid event payload is parsed to get the container name and blob name/URL.
2. The PDF file is downloaded from Blob Storage.
3. Text is extracted from the PDF using **PyPDF2**.
4. The extracted text is sent to an LLM (via an OpenAI-compatible client, `AZURE_OPENAI_ENDPOINT` / `AZURE_OPENAI_DEPLOYMENT`) with a strict extraction prompt that builds a structured, #### #### 2.4 **factual-only** candidate profile (no inference/hallucination) containing:
   - `candidate_title`
   - `years_experience`
   - `industry_domains`
   - `technical_skills`
   - `professional_experience` (role, company, dates, description)
   - `education` (degree, field, institution)
   - `languages`
   - `additional_information`
5. The resulting JSON metadata is uploaded to the **`cv-metadata`** container as `{candidate_name}.json`.
#### 2.5 **Output of this step:** A structured JSON profile per candidate, stored in `cv-metadata`, which in turn triggers **Step 3**.
 
### 3. Chunking

Split loaded documents into smaller chunks because LLMs have a context window limit and feeding an entire book or large report will fail.

| Method                               | Description                                             | Notes                              |
| ------------------------------------ | ------------------------------------------------------- | ---------------------------------- |
| **Fixed-size / token-based**         | Split text every N tokens (e.g., 500 tokens)            | Simple, but may break sentences    |
| **Fixed-size / character-based**     | Split text every N characters                           | Simple, but may cut words          |
| **Sentence-based / paragraph-based** | Split by sentences or paragraphs                        | Preserves meaning, better for QA   |
| **Overlap / sliding window**         | Each chunk overlaps with the previous (e.g., 50 tokens) | Keeps context continuity           |
| **Semantic chunking**                | Split by meaning using embeddings or NLP                | Advanced, best for large documents |

#### Tools Exapmles: langchain.text_splitter, nltk, or custom Python scripts

### 4. Embedding Generation

Convert each text chunk into a vector representation using an embedding model:
#### Tools Examples:
 - OpenAI: text-embedding-3-small or text-embedding-3-large
 - Cohere, HuggingFace, or other embedding models

## 5. Vector Store / Index

Store all embeddings in a vector database for fast similarity search. In fact, we have to definea  collection and a database schema.
### Schema Example:
        ids = []
        metadatas = []
        documents_text = []
        embedding_list= []

### Tools Example ; 

- FAISS (local, fast)
- Weaviate, Pinecone, Milvus (cloud-ready)
- Chroma (open-source, simple)
  
## 6. Retrieval

When the user asks a question, we must convert the query into a query embedding using the same model, search the vector store for top-k, and return the most relevant chunks to the LLM.

## 7. Generation

Feed the retrieved chunks + user query to the LLM and LLM generates an answer grounded in the retrieved documents, reducing hallucinations.


<img width="664" height="774" alt="Screenshot 2025-11-20 at 11 31 36" title="RAG Workflow"  src="https://github.com/user-attachments/assets/fadfde55-6fd7-4b16-a115-d8d0dcb431b4" />


