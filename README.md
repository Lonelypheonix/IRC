# IRC – Intelligent Research Companion

**IRC (Intelligent Research Companion)** is a Streamlit-based application designed to help researchers:

-   Upload and OCR-process PDF papers.
    
-   Automatically extract text, figures, and captions from those PDFs.
    
-   Build a retrieval index (RAG) so you can ask questions about the paper and get answers driven by an LLM.
    
-   Extract the list of references from a paper, then automatically attempt to find and download each cited paper (via arXiv, Semantic Scholar, DuckDuckGo/Google Scholar, etc.).
    
-   For each downloaded reference, run OCR (if necessary) and generate a summary. If the PDF cannot be found, create a short “hypothetical” summary based on the title alone.
    
-   Finally, compile everything into a single, comprehensive Markdown report that includes:
    
    1.  An executive summary of the main PDF.
        
    2.  A breakdown of each major section (abstract, introduction, methods, results, conclusions).
        
    3.  A table of all references, indicating which ones were successfully downloaded and which had to be summarized hypothetically.
        
    4.  “Key insights” that connect the main paper to its references—things like trends, gaps, or novel contributions.
        
    5.  Statistics about the document (word counts, number of figures/tables mentioned, unique vocabulary, etc.).
        

The goal is to give you a one-stop “research companion” that not only OCRs and indexes your main PDF but also automatically retrieves and summarizes every cited work.

----------

## Project Structure

```
.
├── .gitignore
├── README.md
├── requirements.txt
├── app.py              # Main Streamlit application
├── pdfs/               # Temporary storage for user‐uploaded PDFs
├── references/         # Folder where downloaded reference PDFs are saved
├── extracted_images/   # Any figures/images extracted from PDFs
├── agent_logs/         # Logs of the multi‐agent communication (for debugging)
├── outputs/            # OCR results, generated Markdown reports, CSVs, etc.

```

-   **app.py**: The Streamlit app script. It handles PDF uploading, OCR, indexing, chat (RAG), running the multi‐agent reference system, and generating the final report.
    
-   **pdfs/**: Stores any PDF file the user uploads through the sidebar.
    
-   **references/**: After the multi‐agent system runs, any successfully downloaded reference‐paper PDFs will end up here.
    
-   **extracted_images/**: Contains all figures extracted from the main PDF by PyMuPDF.
    
-   **agent_logs/**: Stores debug logs created by the `AgentCommunicationHub`. Each message exchanged between agents is recorded here, and any references that failed to download are listed.
    
-   **outputs/**: Where the app writes OCR results and final reports:
    
    -   `ocr_<PDFName>_<timestamp>.txt` (raw text)
        
    -   `ocr_<PDFName>_<timestamp>.md` (Markdown OCR with figure captions)
        
    -   `report_<PDFName>_<timestamp>.md` (comprehensive research report)
        
-   **requirements.txt**: Lists every Python package the app depends on.
    

----------

## Installation

1.  **Clone the repository**
    
    ```bash
    git clone https://github.com/Lonelypheonix/IRC.git
    cd IRC
    
    ```
    
2.  **Create a virtual environment and install dependencies**
    
    ```bash
    python3 -m venv .venv
    source .venv/bin/activate        # macOS/Linux
    .\.venv\Scripts\activate         # Windows PowerShell
    pip install --upgrade pip
    pip install -r requirements.txt
    
    ```
    
3.  **Set environment variables**
    
    -   **MISTRAL_API_KEY**:
        
        ```bash
        Setup your Mistral API key
        
        ```
        
       
        
4.  **Run the Streamlit app**
    
    ```bash
    streamlit run app.py
    
    ```
    
    Then open your browser to `http://localhost:8501`.
    

----------

## Environment Variables & Third‐Party Services

1.  **Mistral AI OCR**
    
    -   Sign up at [Mistral AI](https://mistral.ai/) to get an API key.
    
    -   The code uses the `mistralai` package and calls:
        
        ```python
        client = Mistral(api_key=api_key)
        ocr_resp = client.ocr.process(...)
        
        ```
        
    -   Without a valid key, OCR calls will either fail or be skipped.
        
2.  **Ollama (Local LLM Manager)**
    
    -   If you have installed Ollama and pulled an image like `gemma3:latest` or `qwen2.5vl:latest`, the app checks `ollama.list()`.
        
    -   If Ollama is available, figure images get sent to the LLM for a 4–6 sentence description. RAG chat will also use the Ollama LLM.
        
    -   If Ollama is not installed, vision‐based captions and some RAG features will be skipped.
        
3.  **arXiv API**
    
    -   The `arxiv` Python package is used to search arXiv by title. No API key is required.
        
    -   Example usage:
        
        ```python
        import arxiv
        search = arxiv.Search(
          query='ti:"Exact Title"',
          max_results=10,
          sort_by=arxiv.SortCriterion.Relevance
        )
        for paper in search.results():
            # compare similarity_score(paper.title, citation_title)
            # if > 0.75, download with paper.download_pdf(...)
        
        ```
        
4.  **Semantic Scholar API**
    
    -   The app sends a GET request to:
        
        ```
        https://api.semanticscholar.org/graph/v1/paper/search
          ?query=<paperTitle>&limit=5&fields=title,url,openAccessPdf
        
        ```
        
    -   If a returned entry’s title is similar (≥ 0.7) to the citation, it attempts to download `openAccessPdf.url`.
        
    -   No API key is required, but be mindful of rate limiting.
        
5.  **DuckDuckGo & Google Scholar Scraping**
    
    -   If you have the `duckduckgo-search` package installed (`pip install duckduckgo-search`), it will use the `DDGS` class:
        
        ```python
        with DDGS() as ddgs:
            results = list(ddgs.text("<query> filetype:pdf", max_results=10))
            for r in results:
                url = r.get("href", "")
                if is_pdf_url(url) and similarity_score(r.get("title",""), citation_title) > 0.6:
                    download via requests.get(...)
        
        ```
        
    -   If that library isn’t available, it falls back to `DuckDuckGoSearchAPIWrapper` from `langchain_community`.
        
    -   As a last fallback, the code constructs a Google Scholar URL:
        
        ```
        https://scholar.google.com/scholar?q=<urlencoded title>
        
        ```
        
        then scrapes any `href="… .pdf"` links from the HTML.
        

----------

## Usage Instructions

1.  **Launch the App**
    
    ```bash
    streamlit run app.py
    
    ```
    
    Then open your browser to `http://localhost:8501`.
    
2.  **Upload a PDF**
    
    -   In the left sidebar, click **Select PDF file** and choose your paper.
        
    -   A spinner will appear: “Processing OCR and indexing….” Wait until it finishes.
        
3.  **Review OCR Output**
    
    -   Mistral OCR extracts all text from each page.
        
    -   PyMuPDF extracts every figure, auto-crops, and saves it under **extracted_images/**.
        
    -   If Ollama is available, each figure is captioned by the LLM.
        
    -   Raw text and Markdown with figure captions appear in **outputs/**.
        
4.  **Generate a Summary**
    
    -   Go to the “📋 Summary & Chat” tab.
        
    -   Click **Generate Summary**. The LLM produces a concise summary of the entire paper.
        
    -   Document statistics appear below (e.g., number of chunks, total characters).
        
5.  **Ask Questions**
    
    -   In the same tab, click one of the suggested questions or type your own.
        
    -   Click **Ask Question**. The app runs a semantic search over the vector store, picks the top 5 chunks, and prompts the LLM. You’ll see an answer plus an optional “View Sources” expander showing the relevant text snippets.
        
6.  **Extract & Process References**
    
    -   Switch to the “🤖 Agent System” tab.
        
    -   Click **Extract and Process All References**.
        
        1.  The Father Agent finds the “References” section via regex.
            
        2.  The Mother Agent loops over each citation, calling Web Agent → Local File Agent.
            
    -   When complete, you’ll see a table with:
        
        -   Each reference title.
            
        -   Summary text (from real downloaded PDF or hypothetical).
            
        -   “Source” field: “Downloaded PDF” or “Title‐Based (No PDF)”.
            
    -   Download the results as a CSV if desired.
        
7.  **View Downloaded PDFs**
    
    -   Go to the “📚 References” tab.
        
    -   Each PDF in **references/** is listed. Click **View** to open it in‐browser.
        
8.  **Inspect Agent Logs**
    
    -   In “📊 Agent Logs,” each logged message between agents is shown with a timestamp and sender→receiver info.
        
    -   Any failed downloads appear under “❌ Failed Downloads.”
        
9.  **Generate the Comprehensive Report**
    
    -   Switch to “📊 Report.” Once the three steps (OCR/indexing, summary, references) are complete, you’ll see three green checkmarks.
        
    -   Click **Generate Comprehensive Report**. The app assembles:
        
        1.  Executive summary.
            
        2.  Section-by-section excerpts.
            
        3.  Reference summaries.
            
        4.  LLM‐generated key insights.
            
        5.  Recommendations for future research.
            
        6.  Document statistics (word count, sentence count, figure/table mentions, unique words).
            
    -   The report is saved under **outputs/** as `report_<PDFName>_<timestamp>.md`. You can also download an HTML version. The report is rendered in‐app for immediate review.
      

## Demo

You can look at the demo on Youtube  under this link **Youtube**. 

----------
## License

This project is licensed under the **MIT License**. See the attached `LICENSE` file for details.

----------

## Acknowledgments

-   **Streamlit** for providing the rapid‐development web framework.
    
-   **Mistral AI** for the OCR SDK.
    
-   **LangChain** for vector store and prompt‐template utilities.
    
-   **Ollama** for offline LLM inference (if you have it installed locally).
    
-   **arXiv**, **Semantic Scholar**, **DuckDuckGo**, and **Google Scholar** for enabling programmatic access to PDF searches.
    

Thank you for using **IRC – Intelligent Research Companion**. If you have any questions or suggestions, please open an issue or submit a pull request!
