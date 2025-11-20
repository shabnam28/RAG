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
