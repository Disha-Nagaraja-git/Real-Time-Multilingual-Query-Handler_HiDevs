# Real-Time Multilingual Query Handler

Timport os
import json
import logging
from typing import Optional
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from pydantic import BaseModel
from dotenv import load_dotenv
import openai
import asyncio
from datetime import datetime

load_dotenv()
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", None)
openai.api_key = OPENAI_API_KEY

app = FastAPI()
logging.basicConfig(level=logging.INFO)

METRICS = {
    "total_messages": 0,
    "translated_count": 0,
    "errors": 0,
}

class IncomingMessage(BaseModel):
    user_id: str
    text: str
    language: Optional[str] = None
    need_response: Optional[bool] = False

class ConnectionManager:
    def __init__(self):
        self.active_connections: dict[str, WebSocket] = {}

    async def connect(self, user_id: str, websocket: WebSocket):
        await websocket.accept()
        self.active_connections[user_id] = websocket

    def disconnect(self, user_id: str):
        if user_id in self.active_connections:
            del self.active_connections[user_id]

    async def send_json(self, user_id: str, message: dict):
        ws = self.active_connections.get(user_id)
        if ws:
            await ws.send_json(message)

manager = ConnectionManager()

async def translate_with_openai(text: str, src_lang: Optional[str] = None) -> str:
    if OPENAI_API_KEY is None:
        raise RuntimeError("OPENAI_API_KEY is not set")

    system_msg = "You are a helpful translation assistant. Translate the user's text to clear, natural English."
    user_prompt = f"Translate the following text to English:\n\n\"\"\"\n{text}\n\"\"\""

    try:
        response = openai.ChatCompletion.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": system_msg},
                {"role": "user", "content": user_prompt},
            ],
            max_tokens=512,
            temperature=0.0,
        )
        return response.choices[0].message.content.strip()
    except Exception:
        logging.exception("Translation failed")
        raise

async def handle_message(user_id: str, payload: IncomingMessage):
    METRICS["total_messages"] += 1
    ts = datetime.utcnow().isoformat()

    try:
        translated = await translate_with_openai(payload.text, payload.language)
        METRICS["translated_count"] += 1

        auto_reply = None
        if payload.need_response:
            prompt = f'User query: "{translated}". Create a short, polite support reply.'
            response = openai.ChatCompletion.create(
                model="gpt-3.5-turbo",
                messages=[
                    {"role": "system", "content": "You are a customer-support assistant."},
                    {"role": "user", "content": prompt},
                ],
                max_tokens=150,
                temperature=0.6,
            )
            auto_reply = response.choices[0].message.content.strip()

        return {
            "timestamp": ts,
            "user_id": user_id,
            "original_text": payload.text,
            "translated_text": translated,
            "auto_reply": auto_reply,
        }

    except Exception as e:
        METRICS["errors"] += 1
        return {
            "timestamp": ts,
            "user_id": user_id,
            "original_text": payload.text,
            "error": str(e),
        }

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: str):
    await manager.connect(user_id, websocket)
    try:
        while True:
            data = await websocket.receive_text()
            try:
                parsed = json.loads(data)
                incoming = IncomingMessage(**parsed)
            except Exception as e:
                await websocket.send_json({"error": "invalid payload", "details": str(e)})
                continue

            result = await handle_message(user_id, incoming)
            await websocket.send_json({"type": "translation_result", "payload": result})

    except WebSocketDisconnect:
        manager.disconnect(user_id)
    except Exception:
        manager.disconnect(user_id)
        logging.exception("Websocket error")

@app.get("/metrics")
async def get_metrics():
    return METRICS

@app.get("/")
async def root():
    return {"message": "Real-Time Multilingual Query Handler is running."}

## Installation
<!doctype html>
<html>
<head>
  <meta charset="utf-8"/>
  <title>RT Multilingual Client</title>
  <style>
    body { font-family: Arial, sans-serif; padding: 20px; }
    textarea { width: 100%; height: 80px; }
    #log { border: 1px solid #ccc; padding: 10px; height: 200px; overflow: auto; margin-top: 10px; white-space: pre-wrap; }
  </style>
</head>
<body>
  <h2>Real-Time Multilingual Query Handler — Demo Client</h2>

  <label>User ID: <input id="userId" value="user123" /></label><br/><br/>
  <textarea id="inputText" placeholder="Type message in any language..."></textarea><br/>
  <label><input id="needResponse" type="checkbox"/> Ask server to auto-generate a reply</label><br/><br/>
  <button id="connectBtn">Connect</button>
  <button id="sendBtn" disabled>Send</button>

  <div id="log"></div>

  <script>
    let ws = null;
    const log = (t) => {
      const el = document.getElementById('log');
      el.textContent += t + "\n";
      el.scrollTop = el.scrollHeight;
    };

    document.getElementById('connectBtn').addEventListener('click', () => {
      const uid = document.getElementById('userId').value.trim();
      if(!uid) { alert('Enter user id'); return; }
      const url = `ws://${location.hostname}:8000/ws/${uid}`;
      ws = new WebSocket(url);
      ws.onopen = () => {
        log('[connected to server]');
        document.getElementById('sendBtn').disabled = false;
      };
      ws.onmessage = (ev) => {
        try {
          const msg = JSON.parse(ev.data);
          log('[server] ' + JSON.stringify(msg, null, 2));
        } catch(e) {
          log('[server raw] ' + ev.data);
        }
      };
      ws.onclose = () => {
        log('[disconnected]');
        document.getElementById('sendBtn').disabled = true;
      };
      ws.onerror = (e) => { log('[error] ' + e); };
    });

    document.getElementById('sendBtn').addEventListener('click', () => {
      if(!ws || ws.readyState !== 1) { alert('Not connected'); return; }
      const payload = {
        user_id: document.getElementById('userId').value.trim(),
        text: document.getElementById('inputText').value.trim(),
        need_response: document.getElementById('needResponse').checked
      };
      ws.send(JSON.stringify(payload));
      log('[you] ' + JSON.stringify(payload));
      document.getElementById('inputText').value = '';
    });
  </script>
</body>
</html>
