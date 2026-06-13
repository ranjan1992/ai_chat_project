# 🤖 AI Chat Assistant

A minimal, full-stack AI chat application that connects a browser UI to a locally running **Mistral LLM** via **Ollama**, served through a **FastAPI** backend.

---

## 📁 Project Structure

```
ai_chat_project-master/
│
├── app.py                  ← FastAPI backend (brain of the app)
│
└── static/
    ├── index.html          ← Chat UI (structure)
    ├── script.js           ← Frontend logic (sends messages, renders replies)
    └── style.css           ← Visual styling
```

---

## 🏗️ Architecture Overview

Here's how all the pieces talk to each other:

```
┌─────────────────────────────────────────────────────────┐
│                      BROWSER                            │
│                                                         │
│   index.html  ──loads──►  script.js                    │
│       │                       │                         │
│  (renders UI)        (handles button click,             │
│                        sends POST /chat)                │
└───────────────────────┬─────────────────────────────────┘
                        │  HTTP POST /chat?prompt=...
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   FASTAPI  (app.py)                     │
│                   localhost:8000                         │
│                                                         │
│   GET  /        ──►  serves index.html                  │
│   GET  /static  ──►  serves CSS & JS files              │
│   POST /chat    ──►  forwards prompt to Ollama          │
└───────────────────────┬─────────────────────────────────┘
                        │  HTTP POST (JSON)
                        ▼
┌─────────────────────────────────────────────────────────┐
│                OLLAMA  (local AI runtime)                │
│                localhost:11434                           │
│                                                         │
│   Model: mistral                                        │
│   Endpoint: /api/generate                               │
│   Returns: { "response": "AI reply text..." }           │
└─────────────────────────────────────────────────────────┘
```

> **Key insight:** FastAPI acts as a **proxy/middleman** — the browser never talks to Ollama directly. All AI calls go through the Python backend.

---

## 🔄 Request Lifecycle (Step-by-Step)

```
User types "Hello AI" and clicks Send
              │
              ▼
[1] script.js grabs input value
              │
              ▼
[2] Appends "You: Hello AI" to chat box (optimistic UI)
              │
              ▼
[3] fetch POST /chat?prompt=Hello%20AI
              │
              ▼
[4] FastAPI @app.post("/chat") receives request
              │
              ▼
[5] Python requests.post → Ollama /api/generate
    Body: { "model": "mistral", "prompt": "Hello AI", "stream": false }
              │
              ▼
[6] Ollama runs the Mistral model, returns JSON:
    { "model": "mistral", "response": "Hello! How can I help?", ... }
              │
              ▼
[7] FastAPI extracts json_response["response"]
    Returns: { "response": "Hello! How can I help?" }
              │
              ▼
[8] script.js receives JSON, appends "AI: Hello! How can I help?"
              │
              ▼
[9] chat box auto-scrolls to bottom
```

---

## 🐍 Backend Deep Dive — `app.py`

```python
from fastapi import FastAPI, HTTPException, Query
from fastapi.staticfiles import StaticFiles
from fastapi.responses import FileResponse
import requests
import os
import json
```

| Import | Purpose |
|---|---|
| `FastAPI` | The web framework — creates the app object |
| `HTTPException` | Lets us send error responses with HTTP status codes (e.g. 500) |
| `Query` | Tells FastAPI: read this parameter from the URL query string |
| `StaticFiles` | Mounts a folder so HTML/CSS/JS files are served directly |
| `FileResponse` | Sends a file from disk as an HTTP response |
| `requests` | Python HTTP client — used to call Ollama's API |
| `os` | Used for `os.path.join()` to build file paths safely |
| `json` | Parses the raw text response from Ollama into a Python dict |

---

### 🔧 App Initialization

```python
app = FastAPI()

app.mount("/static", StaticFiles(directory="static"), name="static")

OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL_NAME = "mistral"
```

- `FastAPI()` creates the application instance — everything hangs off this object.
- `app.mount(...)` tells FastAPI: *"Any request starting with `/static` should be answered by serving files from the `static/` folder."* This is how `style.css` and `script.js` get delivered to the browser automatically.
- `OLLAMA_URL` and `MODEL_NAME` are constants. Hardcoded here — in a real app these would be in a `.env` file.

---

### 🏠 Homepage Route

```python
@app.get("/")
def serve_homepage():
    return FileResponse(os.path.join("static", "index.html"))
```

**How it works:**

```
Browser visits http://localhost:8000/
        │
        ▼
FastAPI matches GET "/"
        │
        ▼
serve_homepage() runs
        │
        ▼
os.path.join("static", "index.html")
  → builds path: "static/index.html"  (OS-safe)
        │
        ▼
FileResponse sends the HTML file to browser
```

> `os.path.join` is used instead of `"static/index.html"` directly because it handles Windows (`\`) vs Unix (`/`) path separators automatically — makes the code portable.

---

### 💬 Chat Route — The Core Logic

```python
@app.post("/chat")
def chat(prompt: str = Query(..., description="User prompt for AI model")):
```

- `@app.post("/chat")` — this function handles HTTP **POST** requests to `/chat`.
- `prompt: str = Query(...)` — FastAPI reads the `prompt` value from the **URL query string** (e.g. `/chat?prompt=Hello`). The `...` means it is **required** — FastAPI will automatically return a 422 error if it's missing.

```python
    headers = {"Content-Type": "application/json"}

    try:
        response = requests.post(
            OLLAMA_URL,
            json={"model": MODEL_NAME, "prompt": prompt, "stream": False},
            headers=headers
        )
```

- `requests.post(...)` fires an HTTP POST to Ollama running locally.
- Passing `json=` (not `data=`) automatically serializes the dict to JSON AND sets the Content-Type header — but the explicit `headers` dict here adds it redundantly (harmless).
- `"stream": False` is critical — it tells Ollama to wait until the full response is generated before returning. If `True`, the response comes back as a stream of tokens, which this code is not designed to handle.

```python
        print("Ollama Response ", response.text)

        response_data = response.text.strip()

        try:
            json_response = json.loads(response_data)
        except json.JSONDecodeError:
            raise HTTPException(
                status_code=500, detail=f"Invalid JSON response from Ollama : {response_data}")
```

**Two-layer error handling:**

```
Ollama returns response.text (raw string)
              │
              ▼
         .strip()  ← removes leading/trailing whitespace
              │
              ▼
     json.loads(response_data)
              │
         ┌────┴────┐
       success    JSONDecodeError
         │              │
         ▼              ▼
   json_response    raise HTTPException(500)
   (Python dict)    "Invalid JSON from Ollama"
```

```python
        ai_response = json_response.get("response")

        if not ai_response:
            raise HTTPException(
                status_code=500, detail="No valid response from Ollama")

        return {"response": ai_response}

    except requests.exceptions.RequestException as e:
        raise HTTPException(
            status_code=500, detail=f"Request to Ollama failed: {str(e)}")
```

- `.get("response")` safely extracts the AI's text. If the key doesn't exist, it returns `None` instead of throwing a `KeyError`.
- The `if not ai_response` guard catches both `None` and empty string `""`.
- The outer `except requests.exceptions.RequestException` catches **network-level failures** — Ollama not running, connection refused, timeout, etc.

**Error handling map:**

```
┌──────────────────────────────────────────────────────┐
│ What can go wrong?         │ How it's caught          │
├────────────────────────────┼──────────────────────────┤
│ Ollama is not running      │ RequestException → 500   │
│ Ollama returns bad JSON    │ JSONDecodeError → 500    │
│ Ollama returns empty reply │ if not ai_response → 500 │
│ prompt missing from URL    │ FastAPI auto → 422        │
└──────────────────────────────────────────────────────┘
```

---

### 🚀 Entry Point

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000, reload=True)
```

- `if __name__ == "__main__"` — only runs when you execute `python app.py` directly. If the file is imported as a module, this block is skipped.
- `host="0.0.0.0"` — listens on **all network interfaces**, not just localhost. Makes the app accessible from other devices on the same network.
- `port=8000` — the app is available at `http://localhost:8000`.
- `reload=True` — during development, uvicorn watches for file changes and auto-restarts. **Remove this in production.**

---

## 🌐 Frontend Deep Dive

### `index.html` — The Structure

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>AI Chat Assistant</title>
    <link rel="stylesheet" href="static/style.css" />   ← CSS loaded in <head>
  </head>
  <body>
    <h1>AI Chat Assistant</h1>

    <div id="chat-box"></div>             ← Messages appear here (starts empty)

    <input type="text" id="user-input"   ← User types here
           placeholder="Type a message....." />

    <button onclick="sendMessage()">Send</button>   ← Triggers JS function

    <script src="static/script.js"></script>        ← JS loaded at end of body
  </body>
</html>
```

> The `<script>` tag is placed at the **end of `<body>`** (not in `<head>`) — this ensures the DOM elements (`chat-box`, `user-input`) exist before JavaScript tries to access them.

---

### `script.js` — The Logic

```javascript
async function sendMessage() {
```
`async` — marks this function as asynchronous. Needed because `fetch()` is async (it returns a Promise), and we use `await` inside.

```javascript
  let inputField = document.getElementById("user-input");
  let chatBox = document.getElementById("chat-box");
```
Grabs references to the two DOM elements we need to read from and write to.

```javascript
  let userMessage = inputField.value;
  chatBox.innerHTML += `<p><strong>You:</strong>${userMessage}</p>`;
  inputField.value = "";
```

- Reads what the user typed.
- **Immediately** adds it to the chat box (before we even hear back from the server — this is called an "optimistic update").
- Clears the input field so the user can type again.

```javascript
  let response = await fetch(
    `/chat?prompt=${encodeURIComponent(userMessage)}`,
    { method: "POST" }
  );
```

- `fetch(...)` sends an HTTP POST to `/chat?prompt=...`.
- `encodeURIComponent(userMessage)` — **critical safety step**. Converts special characters (spaces → `%20`, `&` → `%26`, etc.) so the URL stays valid. Without this, a message like "What's 2+2?" would corrupt the URL.
- `await` pauses execution here until the server responds.

```javascript
  if (!response.ok) {
    chatBox.innerHTML += `<p><strong>AI:</strong>Error to fetch response.</p>`;
    return;
  }
```
`response.ok` is `true` if the HTTP status is 200–299. If FastAPI returned a 500 error, this catches it and shows a fallback message.

```javascript
  let data = await response.json();
  chatBox.innerHTML += `<p><strong>AI:</strong> ${data.response}</p>`;
  chatBox.scrollTop = chatBox.scrollHeight;
}
```

- `response.json()` parses the JSON body `{ "response": "..." }` into a JS object.
- Appends the AI's reply to the chat box.
- `chatBox.scrollTop = chatBox.scrollHeight` — auto-scrolls to the bottom so the latest message is always visible.

---

### `style.css` — The Visual Layer

```css
body {
  font-family: Arial, Helvetica, sans-serif;
  text-align: center;
  margin: auto;          /* centers the body content */
}

#chat-box {
  width: 80%;
  height: 300px;
  overflow-y: scroll;    /* adds vertical scrollbar when content overflows */
  border: 1px solid #ccc;
  padding: 10px;
  margin: auto;          /* centers the chat box */
}

input {
  width: 70%;
  padding: 10px;
  margin: 10px 0;
}

button {
  padding: 10px;
  cursor: pointer;       /* shows hand cursor on hover */
}
```

> `overflow-y: scroll` on `#chat-box` is what enables scrolling through chat history. Combined with the `scrollTop = scrollHeight` in JS, it creates the "auto-scroll to latest message" behavior.

---

## 🔁 Data Flow Summary

```
User Input (browser)
       │
       │  encodeURIComponent()
       ▼
POST /chat?prompt=Hello%20AI
       │
       │  FastAPI Query() extracts "Hello AI"
       ▼
requests.post(OLLAMA_URL, json={
  "model": "mistral",
  "prompt": "Hello AI",
  "stream": false
})
       │
       │  Ollama runs Mistral locally
       ▼
{ "model": "mistral", "response": "Hi there!", ... }
       │
       │  json_response.get("response")
       ▼
{ "response": "Hi there!" }   ← FastAPI returns this
       │
       │  data.response
       ▼
"AI: Hi there!"  appended to chat box
```

---

## ⚙️ How to Run

### Prerequisites

- Python 3.9+
- [Ollama](https://ollama.com) installed and running
- Mistral model pulled: `ollama pull mistral`

### Steps

```bash
# 1. Clone / unzip the project
cd ai_chat_project-master

# 2. Create a virtual environment
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install fastapi uvicorn requests

# 4. Make sure Ollama is running
ollama serve                    # in a separate terminal

# 5. Start the app
python app.py
# OR
uvicorn app:app --reload --port 8000

# 6. Open browser
# http://localhost:8000
```

---

## ⚠️ Known Limitations & Improvement Ideas

| Limitation | What it means | How to fix |
|---|---|---|
| No conversation history | Each message is sent standalone — the AI has no memory of previous messages | Send the full chat history as context with each request |
| `stream: False` | Waits for the full response before displaying — slow for long answers | Implement SSE (Server-Sent Events) for streaming |
| No input validation | Empty messages are sent as-is | Add `if (!userMessage.trim()) return;` in JS |
| Prompt in query string | `?prompt=...` in URL has length limits and is logged in server access logs | Move to POST request body |
| `reload=True` in production | Auto-reload is a dev feature, adds overhead | Remove or use `--no-reload` flag |
| No `.env` for config | `OLLAMA_URL` and `MODEL_NAME` are hardcoded | Use `python-dotenv` to load from `.env` |

---

## 🧱 Tech Stack

| Layer | Technology |
|---|---|
| **AI Model** | Mistral (via Ollama) |
| **Backend** | FastAPI + Uvicorn |
| **HTTP Client** | Python `requests` |
| **Frontend** | Vanilla HTML + CSS + JavaScript |
| **Transport** | HTTP (REST) |
