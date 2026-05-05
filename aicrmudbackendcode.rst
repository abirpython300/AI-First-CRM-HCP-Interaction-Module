.. code:: ipython3

    # =========================
    # IMPORTS
    # =========================
    from fastapi import FastAPI
    from pydantic import BaseModel
    from sqlalchemy import create_engine, Column, Integer, String
    from sqlalchemy.orm import sessionmaker, declarative_base
    import nest_asyncio
    import uvicorn
    from langgraph.graph import StateGraph
    from typing import TypedDict

.. code:: ipython3

    # =========================
    # DATABASE
    # =========================
    DATABASE_URL = "sqlite:///./crm.db"
    
    engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
    SessionLocal = sessionmaker(bind=engine)
    Base = declarative_base()
    
    class Interaction(Base):
        __tablename__ = "interactions"
    
        id = Column(Integer, primary_key=True)
        hcp_name = Column(String)
        interaction_type = Column(String)
        summary = Column(String)
        sentiment = Column(String)
        follow_up_action = Column(String)
    
    Base.metadata.create_all(bind=engine)

.. code:: ipython3

    # =========================
    # FASTAPI
    # =========================
    app = FastAPI()
    from fastapi.middleware.cors import CORSMiddleware
    
    app.add_middleware(
        CORSMiddleware,
        allow_origins=["*"],
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

.. code:: ipython3

    # =========================
    # REQUEST MODEL
    # =========================
    class ChatInput(BaseModel):
        text: str
    

.. code:: ipython3

    # =========================
    # AGENT STATE
    # =========================
    class AgentState(TypedDict):
        input_text: str
        form_data: dict
        last_action: str

.. code:: ipython3

    # =========================
    # TOOL 1: EXTRACT DATA
    # =========================
    import re
    
    def extract_tool(state: AgentState):
        text = state["input_text"]
    
        # Extract doctor name using regex
        match = re.search(r"Dr\.?\s+[A-Za-z]+", text)
        hcp_name = match.group(0) if match else "Unknown HCP"
    
        data = {
            "hcp_name": hcp_name,
            "interaction_type": "meeting",
            "summary": text,
            "sentiment": "neutral",
            "follow_up_action": "None"
        }
    
        return {"form_data": data, "last_action": "extract"}

.. code:: ipython3

    # =========================
    # TOOL 2: SENTIMENT
    # =========================
    def sentiment_tool(state: AgentState):
        data = state["form_data"]
    
        if "good" in state["input_text"].lower():
            data["sentiment"] = "positive"
        elif "bad" in state["input_text"].lower():
            data["sentiment"] = "negative"
        else:
            data["sentiment"] = "neutral"
    
        return {"form_data": data, "last_action": "sentiment"}

.. code:: ipython3

    # =========================
    # TOOL 3: FOLLOW-UP
    # =========================
    def followup_tool(state: AgentState):
        data = state["form_data"]
    
        data["follow_up_action"] = "Schedule follow-up meeting"
    
        return {"form_data": data, "last_action": "followup"}

.. code:: ipython3

    # =========================
    # TOOL 4: EDIT TOOL
    # =========================
    def edit_tool(state: AgentState):
        data = state["form_data"]
    
        if "change" in state["input_text"].lower():
            data["summary"] += " (edited)"
    
        return {"form_data": data, "last_action": "edit"}

.. code:: ipython3

    # =========================
    # TOOL 5: SAVE TOOL
    # =========================
    def save_tool(state: AgentState):
        data = state["form_data"]
    
        db = SessionLocal()
        interaction = Interaction(**data)
        db.add(interaction)
        db.commit()
        db.refresh(interaction)
        db.close()
    
        return {
            "form_data": data,
            "last_action": f"saved_id_{interaction.id}"
        }
    

.. code:: ipython3

    # =========================
    # LANGGRAPH BUILD
    # =========================
    builder = StateGraph(AgentState)
    
    builder.add_node("extract", extract_tool)
    builder.add_node("sentiment", sentiment_tool)
    builder.add_node("followup", followup_tool)
    builder.add_node("edit", edit_tool)
    builder.add_node("save", save_tool)
    
    builder.set_entry_point("extract")
    
    builder.add_edge("extract", "sentiment")
    builder.add_edge("sentiment", "followup")
    builder.add_edge("followup", "edit")
    builder.add_edge("edit", "save")
    
    agent_graph = builder.compile()

.. code:: ipython3

    # =========================
    # API: CHAT CONTROLLED FORM
    # =========================
    @app.post("/chat-agent")
    def chat_agent(input: ChatInput):
        result = agent_graph.invoke({
            "input_text": input.text,
            "form_data": {},
            "last_action": ""
        })
    
        return {
            "status": "success",
            "form_data": result["form_data"],
            "action": result["last_action"]
        }
    
    
    # ✅ ADD THIS BELOW
    @app.get("/interactions")
    def get_interactions():
        db = SessionLocal()
        data = db.query(Interaction).all()
        db.close()
    
        return [
            {
                "id": i.id,
                "hcp_name": i.hcp_name,
                "summary": i.summary,
                "sentiment": i.sentiment
            }
            for i in data
        ]

.. code:: ipython3

    from pydantic import BaseModel
    
    # Request model
    class UpdateInteraction(BaseModel):
        id: int
        summary: str = None
        sentiment: str = None
        follow_up_action: str = None
    
    
    @app.put("/update-interaction")
    def update_interaction(data: UpdateInteraction):
        db = SessionLocal()
        
        interaction = db.query(Interaction).filter(Interaction.id == data.id).first()
        
        if not interaction:
            db.close()
            return {"error": "Interaction not found"}
    
        # Update only provided fields
        if data.summary:
            interaction.summary = data.summary
        if data.sentiment:
            interaction.sentiment = data.sentiment
        if data.follow_up_action:
            interaction.follow_up_action = data.follow_up_action
    
        db.commit()
        db.refresh(interaction)
        db.close()
    
        return {
            "message": "Updated successfully",
            "id": interaction.id
        }

.. code:: ipython3

    # =========================
    # HEALTH CHECK
    # =========================
    @app.get("/")
    def home():
        return {"message": "AI CRM Agent Running 🚀"}

.. code:: ipython3

    import gradio as gr
    import requests
    
    API_URL = "http://127.0.0.1:8000/chat-agent"
    
    def call_agent(user_text):
        try:
            res = requests.post(API_URL, json={"text": user_text}, timeout=10)
            res.raise_for_status()
            data = res.json()
            form = data.get("form_data", {})
            action = data.get("action", "")
        except Exception as e:
            return (
                "", "", "", "", "",
                f"Error calling backend: {e}"
            )
    
        return (
            form.get("hcp_name", ""),
            form.get("interaction_type", ""),
            form.get("summary", ""),
            form.get("sentiment", ""),
            form.get("follow_up_action", ""),
            f"Action: {action}"
        )
    
    with gr.Blocks(title="AI-First CRM (HCP) — Log Interaction") as demo:
        gr.Markdown("## AI-First CRM — Log HCP Interaction")
    
        with gr.Row():
            # LEFT: Read-only form (controlled by AI)
            with gr.Column(scale=3):
                gr.Markdown("### Interaction Details (Read-only)")
                hcp = gr.Textbox(label="HCP Name", interactive=False)
                itype = gr.Textbox(label="Interaction Type", interactive=False)
                summary = gr.Textbox(label="Summary", lines=3, interactive=False)
                sentiment = gr.Textbox(label="Sentiment", interactive=False)
                follow = gr.Textbox(label="Follow-up Action", interactive=False)
    
            # RIGHT: Chat assistant
            with gr.Column(scale=2):
                gr.Markdown("### AI Assistant")
                user_input = gr.Textbox(
                    placeholder="e.g., Met Dr. Sharma, discussion went good, change summary",
                    label="Describe interaction"
                )
                submit = gr.Button("Log / Update")
                status = gr.Textbox(label="System Status", interactive=False)
    
        submit.click(
            fn=call_agent,
            inputs=user_input,
            outputs=[hcp, itype, summary, sentiment, follow, status]
        )
    
    demo.launch()


.. parsed-literal::

    * Running on local URL:  http://127.0.0.1:7860
    * To create a public link, set `share=True` in `launch()`.
    


.. raw:: html

    <div><iframe src="http://127.0.0.1:7860/" width="100%" height="500" allow="autoplay; camera; microphone; clipboard-read; clipboard-write;" frameborder="0" allowfullscreen></iframe></div>




.. parsed-literal::

    



.. code:: ipython3

    # =========================
    # RUN SERVER
    # =========================
    nest_asyncio.apply()
    uvicorn.run(app, host="127.0.0.1", port=8000)


.. parsed-literal::

    INFO:     Started server process [14124]
    INFO:     Waiting for application startup.
    INFO:     Application startup complete.
    

.. parsed-literal::

    INFO:     127.0.0.1:59556 - "GET / HTTP/1.1" 200 OK
    

.. parsed-literal::

    INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
    

.. parsed-literal::

    INFO:     127.0.0.1:59556 - "GET /theme.css?v=63194d3741d384f9f85db890247b6c0ef9e7abac0f297f40a15c59fe4baba916 HTTP/1.1" 200 OK
    INFO:     127.0.0.1:59593 - "GET / HTTP/1.1" 200 OK
    INFO:     127.0.0.1:59595 - "GET /manifest.json HTTP/1.1" 404 Not Found
    INFO:     127.0.0.1:59595 - "GET /manifest.json HTTP/1.1" 404 Not Found
    INFO:     127.0.0.1:59593 - "GET /theme.css?v=63194d3741d384f9f85db890247b6c0ef9e7abac0f297f40a15c59fe4baba916 HTTP/1.1" 200 OK
    INFO:     127.0.0.1:59593 - "GET /static/fonts/ui-sans-serif/ui-sans-serif-Regular.woff2 HTTP/1.1" 404 Not Found
    INFO:     127.0.0.1:59849 - "OPTIONS /chat-agent HTTP/1.1" 200 OK
    INFO:     127.0.0.1:59849 - "POST /chat-agent HTTP/1.1" 200 OK
    INFO:     127.0.0.1:59850 - "POST /chat-agent HTTP/1.1" 200 OK
    











