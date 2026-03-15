# Zener — AI Remote Assistance Platform
**Hackathon Category:** UI Navigator  
**Stack:** Google ADK + Gemini Computer Use + Cloud Run + Firebase + React

---

## Concept

A web platform where users describe a task ("fix my printer settings", "fill out this form"),
and an AI agent **sees their screen** and executes actions on their behalf — with full live
transparency and user confirmation for risky steps.

### User Flow
```
1. User visits web app → describes task → clicks "Start Session"
2. Downloads tiny local agent (~8MB Python binary)
3. Agent requests Accessibility permission (one-time, macOS/Windows/Linux)
4. Agent connects back to cloud via WebSocket tunnel
5. Web app shows live screen stream + real-time action log
6. Agent loop: screenshot → Gemini Computer Use → execute → repeat
7. Task complete → user confirms & session ends
```

> **Judging alignment:**  
> ✅ Breaks the "text-box" paradigm — visual, live, non-chat UI  
> ✅ UI Navigator: Visual precision via Gemini multimodal, not blind clicking  
> ✅ "Live" factor: real streaming, safety confirmations shown in UI  

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         SYSTEM ARCHITECTURE                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────────┐    ┌────────────────┐ │
│  │  React App   │◄──►│  ADK Agent       │◄──►│  Local Agent   │ │
│  │  (Frontend)  │    │  (Cloud Run)     │    │  (Python)      │ │
│  │              │    │                  │    │                │ │
│  │ • Live view  │    │ • Agent loop     │    │ • mss capture  │ │
│  │ • Task input │    │ • Session mgmt   │    │ • PyAutoGUI    │ │
│  │ • Action log │    │ • Safety gate    │    │ • WS client    │ │
│  │ • Confirm UI │    │ • Rate limiting  │    │ • Auth token   │ │
│  └──────────────┘    └──────────────────┘    └────────────────┘ │
│         │                    │                        │          │
│         ▼                    ▼                        ▼          │
│  ┌──────────────┐    ┌──────────────────┐    ┌────────────────┐ │
│  │  Firebase    │    │  Gemini Computer │    │  Firestore     │ │
│  │  Auth        │    │  Use API         │    │  Sessions      │ │
│  └──────────────┘    └──────────────────┘    └────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Layer | Technology | Purpose | Judging Signal |
|-------|-----------|---------|----------------|
| Frontend | React + TypeScript + Vite | Live UI, action stream | Demo quality |
| Agent Server | **Google ADK** (Python) on Cloud Run | Agent loop, orchestration | Google Cloud Native ✅ |
| AI Brain | **Gemini Computer Use** (`gemini-2.5-computer-use-preview-10-2025`) | Screen understanding | Multimodal ✅ |
| Auth | Firebase Auth (Google Sign-In) | User identity | Google Cloud ✅ |
| DB | Firestore | Sessions, usage tracking | Google Cloud ✅ |
| Local Agent | Python 3.11+, mss, PyAutoGUI | Screen capture + execution | Core mechanic |
| Transport | WebSocket (via ADK streaming) | Real-time relay | Live factor ✅ |
| IaC | Cloud Build + `cloudbuild.yaml` | Automated deployment | Bonus points ✅ |

---

## Phase 1 — Local Agent (Python)

### 1.1 Dependencies

```
# requirements.txt
google-adk>=0.3.0           # ADK — mandatory per judging
google-genai>=0.8.0         # Gemini SDK (GA)
mss>=9.0.0                  # Fast multi-monitor screenshot
pyautogui>=0.9.54           # Mouse/keyboard automation
pillow>=10.4.0              # PNG encode/resize
websockets>=13.0            # WS client to ADK server
python-dotenv>=1.0.0        # Env config
```

### 1.2 Project Structure

```
zener-agent/
├── agent/
│   ├── __init__.py
│   ├── capture.py         # mss screen capture → base64 PNG
│   ├── executor.py        # PyAutoGUI action executor
│   ├── ws_client.py       # WebSocket relay to ADK cloud server
│   └── safety.py          # Handle require_confirmation flow
├── main.py                # Entry point — connects and loops
├── requirements.txt
└── .env.example           # ZENER_SESSION_TOKEN, SERVER_URL
```

### 1.3 Screen Capture

```python
# agent/capture.py
import mss, io
from PIL import Image

def capture_screen() -> bytes:
    """Capture primary monitor and return PNG bytes."""
    with mss.mss() as sct:
        monitor = sct.monitors[1]  # Primary monitor
        shot = sct.grab(monitor)
        img = Image.frombytes("RGB", shot.size, shot.bgra, "raw", "BGRX")
        buf = io.BytesIO()
        img.save(buf, format="PNG", optimize=True)
        return buf.getvalue()
```

### 1.4 Action Executor

```python
# agent/executor.py
import pyautogui

pyautogui.FAILSAFE = True  # Move mouse to corner to abort

ACTION_MAP = {
    "click_at":       lambda args: pyautogui.click(args["x"], args["y"]),
    "double_click_at":lambda args: pyautogui.doubleClick(args["x"], args["y"]),
    "hover_at":       lambda args: pyautogui.moveTo(args["x"], args["y"]),
    "type_text_at":   lambda args: (
        pyautogui.click(args["x"], args["y"]),
        pyautogui.write(args["text"], interval=0.03)
    ),
    "key_combination":lambda args: pyautogui.hotkey(*args["keys"].split("+")),
    "scroll_at":      lambda args: pyautogui.scroll(
        int(args.get("magnitude", 300)) * (-1 if args["direction"]=="down" else 1),
        x=args["x"], y=args["y"]
    ),
    "scroll_document":lambda args: pyautogui.scroll(
        -500 if args["direction"]=="down" else 500
    ),
}

def execute_action(name: str, args: dict) -> str:
    handler = ACTION_MAP.get(name)
    if handler:
        handler(args)
        return "ok"
    return f"unknown_action:{name}"
```

---

## Phase 2 — ADK Agent Server (Cloud Run)

> **Key judging requirement:** Must use Google GenAI SDK **or ADK**. We use **ADK** for maximum score.

### 2.1 ADK Agent Definition

```python
# server/agent.py
from google.adk.agents import Agent
from google.adk.tools import Tool
from google.genai import types

# Custom tool: relay action to local agent via WebSocket session
def relay_to_desktop(session_id: str, action_name: str, action_args: dict) -> dict:
    """Send an action to the local agent and return the screenshot result."""
    # Implementation: post to session's WebSocket channel in Firestore
    # Returns {"screenshot": "<base64_png>", "result": "ok"}
    ...

relay_tool = Tool.from_function(relay_to_desktop)

# The top-level UI Navigator agent
navigator_agent = Agent(
    name="zener_navigator",
    model="gemini-2.5-computer-use-preview-10-2025",
    description="Visual UI Navigator that controls the user's desktop to complete tasks.",
    instruction="""
        You are Zener, an expert UI navigator. 
        You receive a task description and screenshots of the user's screen.
        Use Computer Use actions to accomplish the task step by step.
        Before any destructive action, explain what you are about to do.
        After each action, capture a new screenshot and assess progress.
        Stop when the task is complete or if you encounter an unexpected state.
    """,
    tools=[relay_tool],
)
```

### 2.2 Gemini Computer Use Integration (Correct API)

```python
# server/gemini_loop.py
from google import genai
from google.genai import types
from google.genai.types import Content, Part

client = genai.Client()  # Uses GEMINI_API_KEY env var

COMPUTER_USE_CONFIG = types.GenerateContentConfig(
    tools=[
        types.Tool(
            computer_use=types.ComputerUse(
                environment=types.Environment.ENVIRONMENT_DESKTOP,  # ← Desktop, not browser
                excluded_predefined_functions=["drag_and_drop"],    # Safety
            )
        )
    ],
    thinking_config=types.ThinkingConfig(include_thoughts=True),  # Show reasoning
)

async def run_agent_turn(
    contents: list,
    screenshot_bytes: bytes,
    task: str,
) -> tuple[list, list]:
    """Run one turn of the agent loop. Returns (updated_contents, actions)."""
    # Append screenshot to history
    contents.append(Content(role="user", parts=[
        Part(text=task if not contents else "Here is the current screen state."),
        Part.from_bytes(data=screenshot_bytes, mime_type="image/png"),
    ]))

    response = client.models.generate_content(
        model="gemini-2.5-computer-use-preview-10-2025",
        contents=contents,
        config=COMPUTER_USE_CONFIG,
    )

    candidate = response.candidates[0]
    contents.append(candidate.content)

    actions = []
    for part in candidate.content.parts:
        if part.function_call:
            # ← Check safety_decision before executing
            safety = getattr(part, "safety_decision", None)
            actions.append({
                "name": part.function_call.name,
                "args": dict(part.function_call.args),
                "requires_confirmation": (
                    safety is not None and "require_confirmation" in str(safety)
                ),
            })

    return contents, actions
```

### 2.3 Safety Gate — Confirmation Flow

Per official docs, when `safety_decision = require_confirmation`:
- **Server** streams an `action_confirm` event to the React frontend
- **Frontend** shows a modal: "Zener wants to [action]. Allow?"
- User approves → frontend sends `confirm: true` back
- Server proceeds with execution

```python
# server/session.py
async def handle_action(ws_client, action: dict):
    if action["requires_confirmation"]:
        # Pause → ask user in UI
        await ws_client.send_json({"type": "confirm_required", "action": action})
        reply = await ws_client.receive_json()  # blocks until user responds
        if not reply.get("confirmed"):
            return  # Skip if denied
    # Execute
    await ws_client.send_json({"type": "execute", "action": action})
```

### 2.4 Server Structure

```
zener-server/
├── server/
│   ├── __init__.py
│   ├── agent.py           # ADK Agent definition
│   ├── gemini_loop.py     # Gemini Computer Use calls
│   ├── session.py         # Session lifecycle, safety gate
│   ├── relay.py           # WebSocket ↔ Firestore session bus
│   └── auth.py            # Firebase token verification
├── main.py                # FastAPI + ADK server entry
├── Dockerfile
├── cloudbuild.yaml        # ← IaC for bonus points
└── requirements.txt
```

### 2.5 Message Protocol

```typescript
// WebSocket message types
type AgentMessage =
  | { type: "session_ready"; sessionId: string; agentUrl: string }
  | { type: "screenshot"; data: string; timestamp: number }      // base64 PNG
  | { type: "thinking"; text: string }                          // agent reasoning
  | { type: "action_start"; name: string; args: Record<string, any> }
  | { type: "action_done"; name: string; result: string }
  | { type: "confirm_required"; action: { name: string; args: any } }
  | { type: "task_complete"; summary: string }
  | { type: "error"; message: string };

type ClientMessage =
  | { type: "start_session"; task: string; token: string }
  | { type: "confirm"; confirmed: boolean }
  | { type: "abort" };
```

---

## Phase 3 — React Frontend

### 3.1 Key UI Screens

```
src/
├── pages/
│   ├── Landing.tsx         # Hero + sign-in → "wow" first impression
│   ├── Setup.tsx           # Download agent + permission guide
│   └── Session.tsx         # Active session workspace
├── components/
│   ├── ScreenMirror.tsx    # Live PNG frames with action overlays
│   ├── ActionFeed.tsx      # Scrolling log of AI thoughts + actions
│   ├── ConfirmModal.tsx    # Safety gate user confirmation
│   ├── TaskInput.tsx       # What should Zener do?
│   └── StatusBar.tsx       # Connected / thinking / executing
├── hooks/
│   ├── useSession.ts       # WebSocket session manager
│   ├── useAuth.ts          # Firebase Auth hook
│   └── useScreenStream.ts  # Decode + display PNG frames
└── App.tsx
```

### 3.2 ScreenMirror Component

- Renders latest PNG frame as `<img>` (updated on each `screenshot` event)
- Overlays colored SVG annotations on click coordinates:
  - 🟢 Green pulse = click
  - 🔵 Blue box = type text
  - 🟡 Yellow tint + modal = confirmation required
- Renders agent "thought" text as a floating tooltip
- Auto-scales to viewport, maintains aspect ratio

### 3.3 Live Demo-Friendly Features

Per the **Demo & Presentation** judging criterion (30%):  
- Large, prominent action feed with icons and timestamps  
- "Thinking..." shimmer animation during Gemini processing  
- Architecture diagram visible in the `/about` route  
- Record session replay for the demo video

---

## Phase 4 — Firebase + Firestore

### 4.1 Data Schema

```typescript
// Firestore: users/{uid}
interface User {
  uid: string;
  email: string;
  createdAt: Timestamp;
  plan: "free" | "pro";
  usageMinutes: number;
  monthlyReset: Timestamp;
}

// Firestore: sessions/{sessionId}
interface Session {
  uid: string;
  status: "waiting" | "active" | "complete" | "error";
  task: string;
  createdAt: Timestamp;
  lastActive: Timestamp;
  actionCount: number;
  agentToken: string;  // Short-lived, signed token for local agent auth
}
```

### 4.2 Session Token Flow

1. User clicks "Start" → React calls Cloud Run `/api/session/create`
2. Server creates Firestore session, mints a `sessionToken` (JWT, 30min TTL)
3. Frontend displays QR code + download link containing `sessionToken`
4. Local agent starts with `--token <sessionToken>`, authenticates to server
5. Server verifies token → binds agent WebSocket to session

---

## Phase 5 — Deployment & IaC (Bonus Points)

### 5.1 cloudbuild.yaml

```yaml
# cloudbuild.yaml
steps:
  # Build and push agent server image
  - name: 'gcr.io/kaniko-project/executor:latest'
    args:
      - '--context=.'
      - '--dockerfile=zener-server/Dockerfile'
      - '--destination=gcr.io/$PROJECT_ID/zener-server:$COMMIT_SHA'

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'zener-server'
      - '--image=gcr.io/$PROJECT_ID/zener-server:$COMMIT_SHA'
      - '--region=us-central1'
      - '--platform=managed'
      - '--allow-unauthenticated'
      - '--set-env-vars=GEMINI_API_KEY=$$GEMINI_API_KEY,FIREBASE_PROJECT_ID=$PROJECT_ID'
    secretEnv: ['GEMINI_API_KEY']

  # Deploy React frontend to Firebase Hosting
  - name: 'node:20'
    entrypoint: bash
    args:
      - '-c'
      - 'cd zener-web && npm ci && npm run build'
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: firebase
    args: ['deploy', '--only', 'hosting']

availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/gemini-api-key/versions/latest
      env: 'GEMINI_API_KEY'
```

### 5.2 One-Command Deploy

```bash
gcloud builds submit --config cloudbuild.yaml .
```

---

## Implementation Roadmap

| Week | Focus | Deliverable |
|------|-------|-------------|
| 1 | Local Agent | Working `capture.py` + `executor.py` on all 3 OS |
| 2 | Gemini Integration | Agent loop with safety gate, tested standalone |
| 3 | ADK Agent Server | Cloud Run service with WebSocket relay |
| 4 | React Frontend | Live screen view + action feed + confirm modal |
| 5 | Integration | End-to-end session flow, Firebase auth/sessions |
| 6 | IaC + Polish + Demo | `cloudbuild.yaml`, architecture diagram, demo video |

---

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| API Key exposure | User provides own Gemini key OR server uses Cloud Secret Manager |
| Unauthorized local agent | Short-lived signed session tokens (JWT, 30min) |
| Risky actions | `safety_decision` + `require_confirmation` flow → user approves |
| Excluded functions | `drag_and_drop` and file system ops excluded by default |
| Screenshots | Never persisted; processed in-memory, discarded after turn |
| Session hijacking | Unique session IDs, token-bound WebSocket channels |
| Rate limiting | 1 action / 500ms per session; max 100 actions per session |

---

## Judging Alignment Matrix

| Criterion | Weight | How Zener Wins |
|-----------|--------|----------------|
| **Innovation & Multimodal UX** | 40% | True visual navigation — Gemini *sees* the screen; safety confirmation shown in UI; live streaming feels immersive, not turn-based |
| **UI Navigator Execution** | (sub) | Visual precision via Gemini multimodal; coordinate-based targeting; screen context understanding — not DOM hacking |
| **Technical Implementation** | 30% | ADK + GenAI SDK; Cloud Run + Firestore + Firebase Auth; error handling, safety gate, rate limiting, token auth |
| **Demo & Presentation** | 30% | Architecture diagram in app; live session video showing real desktop control; Cloud Run deployment proof in `cloudbuild.yaml` |
| **IaC Bonus** | +0.2 | `cloudbuild.yaml` in repo — automated deploy |
| **Content Bonus** | +0.6 | Write a dev.to / Medium article with `#GeminiLiveAgentChallenge` |

---

## Key Decisions — Resolved

| Question | Decision | Rationale |
|----------|----------|-----------|
| Download vs auto-install agent? | **Downloadable binary** | More trust, simpler permissions flow |
| Action failure handling? | Retry once → ask user if still failing | Best demo experience |
| Allowed actions? | Desktop actions only; explicit exclude list for file ops | Safety-first |
| Use ADK or raw GenAI SDK? | **ADK** (wraps GenAI SDK) | Mandatory for max judging score |
| Browser env vs desktop env? | **ENVIRONMENT_DESKTOP** | We control the full OS, not just a browser |
| Model? | `gemini-2.5-computer-use-preview-10-2025` | Only two supported models; this is the GA preview |

---

## Notable API Corrections vs. Original Plan

| Original (Wrong) | Corrected |
|-----------------|-----------|
| `environment=types.Environment.DESKTOP` | `types.Environment.ENVIRONMENT_DESKTOP` |
| No `safety_decision` handling | Full `require_confirmation` gate implemented |
| No ADK usage | ADK `Agent` + `Tool` wraps entire server loop |
| No `thinking_config` | `ThinkingConfig(include_thoughts=True)` for richer reasoning |
| No IaC | `cloudbuild.yaml` for automated Cloud Build + Cloud Run deploy |
| No content bonus strategy | Planned dev.to article with hashtag |
| `mss.tools.to_png(screenshot, screenshot)` (wrong args) | `PIL.Image` encode to buffer |