# CALLSHIELD INDIA
## Complete Web Development Workflow & Input/Output Specification

**Project:** Gnani AI Hackathon  
**Frontend:** React  
**Main Backend:** Java Spring Boot  
**AI Inference:** Gnani Evon v3.3 GGUF using llama.cpp on AWS EC2  
**Speech-to-Text:** Gnani Prisma  
**Text-to-Speech:** Gnani Timbre  
**Translation:** Google Cloud Translation API (or another translation service)

**Current scope:** Build a fully functional end-to-end application using pretrained AI models and existing APIs. Fine-tuning will be considered only after the application is working.

---

## 1. What Exactly Are We Building?

CallShield India is a multilingual AI application that analyzes phone conversations to detect possible financial scams and social-engineering attempts.

A user can:

1. Upload a recorded call.
2. Paste a conversation transcript.
3. Run a simulated live conversation.
4. View the conversation's fraud-risk classification.
5. See suspicious phrases and potential risk indicators, where supporting analysis is available.
6. Receive safety recommendations.
7. Listen to a warning in their preferred language.

**Three primary classifications:**

- **HIGH RISK:** Strong indications of a scam.
- **SUSPICIOUS:** Some warning signs, but insufficient evidence for a high-risk classification.
- **LOW RISK:** No meaningful scam indicators identified in the provided conversation.

The frontend should also support an **UNDETERMINED** state for failed, incomplete or invalid AI responses.

---

## 2. Complete Application Workflow

```text
                    CALLSHIELD INDIA
                           |
                           v
                USER OPENS WEBSITE
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
         Upload         Paste Text    Live Simulation
         Audio          Transcript    (Advanced MVP)
             |             |             |
             v             |             |
      Gnani Prisma         |             |
      Speech-to-Text       |             |
             |             |             |
             +-------------+-------------+
                           |
                           v
                  ORIGINAL TRANSCRIPT
                           |
                           v
                LANGUAGE IDENTIFICATION
                           |
                           v
               OPTIONAL TRANSLATION API
            Normalize into English/Hinglish
                           |
                           v
                    GNANI EVON v3.3
                Pretrained GGUF via AWS
                  Semantic Classification
                           |
             +-------------+-------------+
             |             |             |
             v             v             v
         HIGH RISK      SUSPICIOUS     LOW RISK
             |             |             |
             +-------------+-------------+
                           |
                           v
                 SAFETY DECISION ENGINE
               Risk Indicators + Warnings
                           |
                           v
                  WARNING LANGUAGE
                   Selection/Lookup
                           |
                           v
                    GNANI TIMBRE
                   Text-to-Speech
                           |
                           v
                   RESULTS DASHBOARD
             Risk + Transcript + Warning
```

### Key technical rule

Evon only needs to return **A, B or C** for the initial MVP:

- `A` → `HIGH_RISK`
- `B` → `SUSPICIOUS`
- `C` → `LOW_RISK`

We have already tested this classification approach successfully using the GGUF model on a CPU-only AWS EC2 instance.

Do not assume the classifier also returns explanations, confidence scores or highlighted evidence. Those are separate planned features.

---

## 3. Input Types

### Input A — Upload Recorded Audio

User provides:

| Field | Type | Details |
|---|---|---|
| Audio file | File | MP3, WAV, M4A, subject to transcription support |
| Conversation language | Selection | Auto-detect or choose available language |
| Preferred warning language | Selection | Hindi, English, etc. |
| Consent | Checkbox | Required for audio processing |

**Processing:** Audio → Prisma → transcript → optional translation → Evon → risk result.

### Input B — Paste Conversation Text

User provides:

| Field | Type | Details |
|---|---|---|
| Transcript | Text | Full conversation |
| Input language | Selection | Auto / Hindi / Hinglish / English / supported regional language |
| Warning language | Selection | Preferred language for safety audio |

Example:

Caller: Main SBI se bol raha hoon. Aapka account block hone wala hai.

Customer: Kya problem hai?

Caller: Verification ke liye OTP bataiye.

**Processing:** Text → optional translation → Evon → result.

This should be the first flow implemented because it uses the already-tested Evon inference API.

### Input C — Live Conversation Simulation

This is intended as a hackathon demonstration rather than actual interception of phone calls.

User selects a simulated scenario and starts the session.

Examples:
- Fake bank representative
- UPI refund scam
- Digital arrest impersonation
- Legitimate customer support call

Dialogue appears progressively, and the risk classification can update as new transcript segments arrive.

For the first version, this can be implemented with predefined synthetic scenarios.

---

## 4. Models and Exact Input/Output Responsibilities

| Component | Input | Output |
|---|---|---|
| **Gnani Prisma** | Call audio | Recognized speech/transcript |
| **Translation API** | Original transcript + input/target language | Normalized English/Hinglish transcript |
| **Gnani Evon** | Normalized transcript + system instructions | `A`, `B` or `C` |
| **Decision Engine** | Risk category + available indicators | Safety actions and warning template ID |
| **Translation/Localization** | Warning template + preferred language | Localized warning text |
| **Gnani Timbre** | Localized warning text + voice/language settings | Generated warning audio |
| **React** | Structured backend response | Results dashboard |

**Translation is optional.** English and Hinglish can be passed directly to Evon where appropriate. Other languages can be translated to English for analysis.

Prisma and Timbre still handle their respective speech tasks. The translation service handles text normalization; it does not replace the speech models.

---

## 5. Evon Inference — Important Details for Backend Developer

Evon is hosted on an AWS EC2 `r6i.xlarge` machine using the community-quantized GGUF checkpoint and a llama.cpp server.

**Current internal endpoint:**

`POST http://127.0.0.1:8080/v1/chat/completions`

This address works inside the EC2 instance, not from the public website.

The final backend must call the inference service through an authenticated, secured connection.

### Example request sent to Evon

```json
{
  "model": "evon",
  "messages": [
    {
      "role": "system",
      "content": "You are CallShield India's fraud-risk classifier. A = HIGH_RISK, B = SUSPICIOUS, C = LOW_RISK. Classify the conversation based on meaning and context. Output only A, B or C."
    },
    {
      "role": "user",
      "content": "Caller: Your account will be blocked. Tell me your OTP immediately."
    }
  ],
  "temperature": 0,
  "max_tokens": 8,
  "chat_template_kwargs": {
    "enable_thinking": false
  }
}
```

### Actual type of response received

```json
{
  "choices": [
    {
      "finish_reason": "stop",
      "message": {
        "role": "assistant",
        "content": "A"
      }
    }
  ]
}
```

The actual response also contains request metadata and token usage.

### Required backend conversion

```text
A → HIGH_RISK
B → SUSPICIOUS
C → LOW_RISK
anything else → UNDETERMINED
```

**Do not expose the raw llama.cpp endpoint or AWS credentials to React.**

The existing AWS instance is currently stopped. The final application will need the inference service running whenever analysis is requested.

---

## 6. Safety Decision Engine

The classification result is used to select a warning.

For the first version, warnings should come from predefined templates rather than requesting long Evon-generated responses.

| Risk | UI Heading | Example advice |
|---|---|---|
| HIGH_RISK | Strong Fraud Warning | Do not share OTP/PIN, transfer money or follow instructions until independently verified |
| SUSPICIOUS | Proceed With Caution | Verify the caller through official channels |
| LOW_RISK | No Significant Risk Detected | Stay alert; the system cannot guarantee a call is safe |
| UNDETERMINED | Unable to Determine Risk | Try again or manually verify the caller |

We can also add simple deterministic risk indicators such as OTP requests, urgent payment demands or account-blocking threats.

These must be clearly identified as rule-based indicators, not automatically presented as evidence produced by Evon.

---

## 7. User Interface Pages

### Page 1 — Landing Page

**Purpose:** Introduce the project and direct users to analysis.

Suggested headline:

**CallShield India — Detect the Scam Before You Trust the Call.**

Components:
- Main headline and subtitle
- Analyze a Call button
- Try Live Simulation button
- How It Works section
- Supported scam types
- Technology overview
- Privacy and consent statement

### Page 2 — Analysis Input

Two main tabs:

**Upload Audio**
- Drag-and-drop uploader
- File validation
- Audio preview
- Language selection
- Warning language preference
- Consent confirmation
- Analyze Call button

**Paste Transcript**
- Large conversation text area
- Optional speaker formatting
- Input language selection
- Warning language selection
- Analyze Transcript button

### Page 3 — Processing

Show meaningful processing stages:

1. Receiving input
2. Transcribing audio (audio mode only)
3. Normalizing language (when required)
4. Evon analyzing conversation
5. Preparing warning
6. Generating warning audio

Stages should show actual backend status, not arbitrary progress percentages.

### Page 4 — Analysis Results

Suggested desktop layout:

**Left side — Conversation**
- Original transcript
- Speaker labels
- Timestamps when available
- Audio player for uploaded call
- Original/translated transcript toggle
- Highlighted risk phrases when supported

**Right side — Fraud Analysis**
- Risk category
- Short summary
- Detected indicators if available
- Safety recommendations
- Warning language selector
- Play Warning button
- Analyze Another Call button

The model's risk category should be the primary result.

Any detailed explanation or evidence visualization should only appear when the backend supplies actual supporting data.

### Page 5 — Simulated Live Monitoring

Components:
- Select simulation scenario
- Start/Pause/Reset simulation
- Incoming dialogue bubbles
- Current risk status
- Risk progression timeline
- Contextual warning notifications
- Spoken warning playback

For the initial demo, use prerecorded or predefined conversations. Do not claim the application intercepts real calls.

### Page 6 — History (Secondary Priority)

- Past analyses
- Input type
- Detected risk
- Language
- Date/time
- View analysis

This page can initially use mock data until persistence is implemented.

---

## 8. Frontend ↔ Backend API Contracts

All endpoints below are **proposed Spring Boot application APIs** to be implemented by the development team.

### API A — Analyze Text

`POST /api/v1/analyze/text`

Request:

```json
{
  "transcript": "Caller: Aapka account block ho jayega. OTP bataiye.",
  "inputLanguage": "hi-Latn",
  "warningLanguage": "hi-IN"
}
```

Response:

```json
{
  "analysisId": "ana_001",
  "status": "PROCESSING"
}
```

### API B — Analyze Audio

`POST /api/v1/analyze/audio`

Request: `multipart/form-data`

Fields:
- `file`
- `inputLanguage`
- `warningLanguage`
- `consent`

Response:

```json
{
  "analysisId": "ana_002",
  "status": "PROCESSING"
}
```

### API C — Get Analysis Result

`GET /api/v1/analyze/{analysisId}`

Example completed response:

```json
{
  "analysisId": "ana_001",
  "status": "COMPLETED",
  "inputType": "TEXT",
  "originalLanguage": "hi-Latn",
  "originalTranscript": "Caller: Aapka account block ho jayega. OTP bataiye.",
  "normalizedLanguage": "en",
  "normalizedTranscript": "Caller: Your account will be blocked. Tell me your OTP.",
  "risk": {
    "code": "A",
    "level": "HIGH_RISK",
    "model": "gnani-evon-v3.3",
    "confidence": null
  },
  "warning": {
    "templateId": "OTP_FRAUD_WARNING",
    "language": "hi-IN",
    "text": "सावधान! अपना ओटीपी या पिन साझा न करें।",
    "audioUrl": null,
    "audioStatus": "NOT_GENERATED"
  },
  "recommendations": [
    "Do not share sensitive credentials.",
    "Contact the bank using its official number."
  ],
  "indicators": [],
  "createdAt": "2026-10-18T10:30:00Z"
}
```

This is a proposed response schema, not a claim that all supporting fields are already implemented.

### API D — Generate Warning Audio

`POST /api/v1/analyze/{analysisId}/warning-audio`

Request:

```json
{
  "language": "hi-IN"
}
```

Response when completed:

```json
{
  "status": "READY",
  "language": "hi-IN",
  "audioUrl": "/api/v1/analyze/ana_001/warning-audio"
}
```

The backend calls Timbre and provides the playable audio through a controlled endpoint.

### API E — Get Analysis History

`GET /api/v1/analyses`

Returns past analyses for the current user/session.

### API F — Live Simulation

`POST /api/v1/simulations`

Creates a simulated conversation session.

`POST /api/v1/simulations/{sessionId}/turns`

Adds a new synthetic conversation turn.

`GET /api/v1/simulations/{sessionId}/events`

Streams transcript updates, risk status and notifications.

---

## 9. Expected Input/Output Examples

### Scenario 1 — High-Risk Bank Scam

**Input:**

"Sir, aapka bank account 10 minute mein freeze hone wala hai. Abhi OTP bataiye."

**Expected output:**
- Risk: HIGH_RISK
- Display: Strong fraud-risk warning
- Warning text: Do not share OTP or banking credentials.
- Audio: Appropriate warning using Timbre.

### Scenario 2 — Legitimate Bank Safety Advice

**Input:**

"Please remember, our bank never asks customers to share their OTP or PIN."

**Expected output:**
- Risk: LOW_RISK
- Display: No major risk indicator detected
- Audio: Usually unnecessary.

### Scenario 3 — Unclear Verification Call

**Input:**

"Main bank se bol raha hoon. Aapki verification pending hai."

**Expected output:**
- Risk: SUSPICIOUS
- Display: Caller identity needs independent verification
- Audio: Cautionary warning if enabled.

These are target behaviors for testing. Evon predictions can differ, and classification accuracy must be evaluated.

---

## 10. Application States

Frontend should support:

- `IDLE`
- `UPLOADING`
- `TRANSCRIBING`
- `TRANSLATING`
- `CLASSIFYING`
- `PREPARING_WARNING`
- `GENERATING_AUDIO`
- `COMPLETED`
- `FAILED`

Not every request passes through every state.

For example, pasted text skips upload/transcription, and a low-risk result may skip audio generation.

The application should display helpful failure messages if transcription, translation, Evon inference or Timbre is unavailable.

---

## 11. Deployment Plan

### React frontend

Deploy through GitHub to Vercel, Netlify or another web hosting platform.

### Java Spring Boot backend

Deploy to an appropriate Java hosting platform or AWS.

### Gnani Evon inference

Run the community GGUF model through llama.cpp on AWS EC2.

**Current tested machine:**
- AWS EC2 `r6i.xlarge`
- CPU only
- 32 GiB RAM configuration
- Ubuntu Linux
- OpenAI-compatible llama.cpp server

Successful testing confirmed short inference without lengthy reasoning.

The AWS instance is currently stopped, so the final application cannot invoke it until it is started again.

Use a configurable backend setting such as:

`EVON_API_URL`

Never hardcode the EC2 IP address into the React frontend.

For deployment, the Java backend must communicate securely with the inference server through an authenticated HTTPS or private-network connection.

### Other API services

- Gnani Prisma for transcription
- Translation API for optional language normalization
- Gnani Timbre for voice warnings

Credentials stay in backend environment variables or a secrets manager, never in frontend code.

---

## 12. Suggested Development Order

**Phase 1 — Core functional application**

1. Build React landing page.
2. Build text analysis form.
3. Implement Spring Boot analysis endpoints.
4. Connect Spring Boot to Evon inference.
5. Display classifications and predefined warnings.
6. Deploy frontend and backend publicly.

**Phase 2 — Multilingual speech pipeline**

1. Add audio upload.
2. Connect Prisma speech recognition.
3. Add text translation/normalization.
4. Add warning language selection.
5. Connect Timbre for spoken warnings.
6. Implement audio player.

**Phase 3 — Hackathon demonstration enhancements**

1. Add simulated live call workflow.
2. Add risk progression.
3. Add rule-based supporting indicators.
4. Add original/normalized transcript toggle.
5. Improve responsive UI.
6. Test benign, suspicious and fraudulent scenarios.

**Fine-tuning is explicitly out of scope until the end-to-end application is built and tested.**

---

## 13. Responsibilities

### AI/ML Developer

- Maintain AWS Evon inference service
- Define and test classification prompts
- Integrate model-side language handling
- Support Prisma and Timbre configuration
- Evaluate classification behavior
- Coordinate input/output schemas

### Web/Java Developer

- Develop React interfaces
- Build Java Spring Boot APIs
- Implement upload and analysis workflows
- Manage status and error states
- Integrate external API responses
- Implement user sessions and optional history
- Manage frontend/backend deployment
- Build the interactive demo experience

### Shared responsibilities

- Agree on API response contracts
- Implement language and warning handling
- Verify end-to-end behavior
- Build synthetic demo scenarios
- Test privacy, security and error handling

---

## 14. Final Product Vision

The completed hackathon MVP should let a judge:

1. Open a public CallShield India website.
2. Paste an English/Hinglish conversation or upload supported audio.
3. Submit it for fraud analysis.
4. See the conversation transcribed/normalized where appropriate.
5. Receive a short Gnani Evon fraud-risk classification.
6. Read a safety recommendation.
7. Play a warning generated in their selected language using Gnani Timbre.

**The core innovation is connecting multilingual speech understanding, semantic risk classification and actionable voice warnings into one accessible fraud-prevention workflow.**

The first priority is a reliable working product using pretrained models. Additional model training and fine-tuning come later.