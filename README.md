# ai_agent_customer_service/main.py

from fastapi import FastAPI
from pydantic import BaseModel
import os

app = FastAPI()

try:
    import ssl  # Check if SSL module is available
except ImportError:
    ssl = None
    raise ImportError("The 'ssl' module is not available in your Python environment. Please ensure Python is compiled with SSL support.")

try:
    import openai
    openai.api_key = os.getenv("OPENAI_API_KEY") or "your-openai-api-key"
except ImportError:
    openai = None
    # Fallback mock for sandboxed environments without internet access
    class MockOpenAI:
        def ChatCompletion(self):
            return self

        def create(self, **kwargs):
            return {
                'choices': [
                    {'message': {'content': "This is a mock response. The real OpenAI API is not available in this environment."}}
                ]
            }

    openai = MockOpenAI()

class Message(BaseModel):
    user_id: str
    text: str
    language: str = "en"

# Simple multilingual dictionary for handoff message
translation_map = {
    "es": {"I’ve escalated this to a human agent who will join shortly.": "He escalado esto a un agente humano que se unirá en breve."},
    "fr": {"I’ve escalated this to a human agent who will join shortly.": "J'ai transmis cela à un agent humain qui vous rejoindra bientôt."},
    # Add more manual translations as needed
}

@app.post("/chat")
async def chat(msg: Message):
    translated_text = msg.text

    # Human handoff logic (simple trigger)
    if any(trigger in msg.text.lower() for trigger in ["human", "agent", "not helpful"]):
        response_text = "I’ve escalated this to a human agent who will join shortly."
        if msg.language != "en" and msg.language in translation_map:
            response_text = translation_map[msg.language].get(response_text, response_text)
        return {"response": response_text}

    # Call OpenAI GPT-4 or use mock fallback
    try:
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[{"role": "user", "content": translated_text}]
        )
        reply = response['choices'][0]['message']['content']
    except Exception:
        reply = "I'm sorry, I'm having trouble processing your request right now."

    return {"response": reply}

# Run this with: uvicorn main:app --reload
