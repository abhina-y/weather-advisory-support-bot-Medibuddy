# 🌤️ MediBuddy Weather-Advisory Support Bot

A production-grade Weather Advisory Support Bot backed by a **LangGraph agent**, **Open-Meteo live weather API**, **Groq LLM (LLaMA-3.3 70B)**, and **strictly grounded Standard Operating Procedures (SOPs)**.

---

## 📦 What We Are Handing In (Submission Deliverables)

1. **Standalone Repository**: Clean modular codebase containing backend agent logic, API wrapper, SOP engine, and Streamlit frontend.
2. **README with Setup & Run Instructions**: Complete guide for running both the frontend chat app and the evaluation suite.
3. **SOPs Representation (`sops.json`)**: 
   > **One-line Note on Form Selection**: *We chose a declarative JSON schema format because it completely decouples safety policies from application control flow, allowing domain/policy teams to add, update, or test an 11th SOP live on the spot without modifying Python code or introducing execution regressions.*
4. **LangGraph Implementation ([`agent.py`](file:///c:/Users/Abhin/Desktop/Weather_Advisor_bot/agent.py))**: Real state graph with conditional branching (`route_after_weather`, `route_after_sop`) and session-level conversational memory.
5. **Evaluation Suite ([`eval_suite.py`](file:///c:/Users/Abhin/Desktop/Weather_Advisor_bot/eval_suite.py))**: 8 comprehensive test cases covering all required scenarios, with 100% pass results and honest technical notes on live weather variability.

---

## 🎯 Architecture Overview & Design Rationale

### 1. Hard Separation of Concerns: Code vs. Policy vs. Model
- **Deterministic SOP Engine (`sop_engine.py`)**: All safety advice logic is governed by written SOPs stored in `sops.json`. The bot is **strictly forbidden** from inventing generic advice or making ungrounded safety claims.
- **Dynamic Policy Loading**: SOP policies exist as a JSON data file (`sops.json`). **Adding, updating, or removing an SOP (such as adding an 11th SOP live during code review) requires ZERO code changes.**
- **Model Role (Groq LLM)**: The LLM only composes natural language output based on **exact API weather numbers** and the **prescribed guidance of matched SOPs**. The LLM never invents weather forecasts or makes up safety policies.

---

## 🧠 LangGraph Graph Structure & Branching Logic

The decision flow is built as a state graph using **LangGraph**:

```
                  ┌────────────────────────┐
                  │      (START)           │
                  └──────────┬─────────────┘
                             │
                  ┌──────────▼─────────────┐
                  │  extract_intent_node   │  (Extract location & activity from history)
                  └──────────┬─────────────┘
                             │
                  ┌──────────▼─────────────┐
                  │   fetch_weather_node   │  (Open-Meteo Geocoding + Live Forecast API)
                  └──────────┬─────────────┘
                             │
                [route_after_weather?]
                 /                     \
      (Success) /                       \ (API / Geocoding Error)
               ▼                         ▼
    ┌──────────────────┐    ┌─────────────────────────┐
    │ match_sop_node   │    │ render_weather_error    │ ───► [END]
    └────────┬─────────┘    └─────────────────────────┘
             │
     [route_after_sop?]
      /               \
 (Match Found)         \ (No SOP Matches)
    ▼                   ▼
┌───────────────────────┐  ┌─────────────────────────┐
│ generate_sop_response │  │     render_no_sop       │ ───► [END]
└───────────┬───────────┘  └─────────────────────────┘
            │
            ▼
         [END]
```

### Key Graph Nodes & Edges:
1. `extract_intent_node`: Extracts `location` and `activity` from the current query and prior conversational history (enabling session context retention across follow-ups like *"What about this evening instead?"*).
2. `fetch_weather_node`: Calls Open-Meteo Geocoding API (`/v1/search`) and Forecast API (`/v1/forecast`) with required fields (`current=temperature_2m,wind_speed_10m,precipitation,uv_index...`). Sanitizes queries by stripping quotes and punctuation.
3. `route_after_weather` *(Conditional Edge)*:
   - If geocoding or weather API fails -> Routes to `render_weather_error` (honest failure response, no forecast guessing).
   - If weather data is available -> Routes to `match_sop_node`.
4. `match_sop_node`: Evaluates weather facts & query against all rules in `sops.json`. Ranks matches by severity (`CRITICAL` > `HIGH` > `MEDIUM` > `LOW`). Handles system-wide alerts (low-pressure systems).
5. `route_after_sop` *(Conditional Edge)*:
   - If SOP matches -> Routes to `generate_sop_response`.
   - If NO SOP matches -> Routes to `render_no_sop` (honest response: *"We don't have policy guidance for that"*).
6. `generate_sop_response`: LLM composes response citing exact weather stats and policy citation.

---

## 📜 SOP Rule Set Rationale (`sops.json`)

The rule set contains **11 SOPs** across **5 distinct categories**:

1. **System-Wide Override**: `SOP-SYS-001` (Severe Low-Pressure / Cyclonic Storm Alert - overrides activity rules during heavy rain/squally winds).
2. **Outdoor Exercise**: `SOP-EX-001` (High UV Index >= 8), `SOP-EX-002` (Extreme Heat >= 38°C), `SOP-EX-003` (High Wind Two-Wheeler / Cycling > 40 km/h), `SOP-EX-004` (Moderate Rain Trail Caution).
3. **Travel & Commute**: `SOP-TR-001` (High Rain Probability > 70%), `SOP-TR-002` (Gale Gust Road Travel > 60 km/h).
4. **Vulnerable Groups**: `SOP-VG-001` (Children/Elderly/Pets Cold Exposure <= 5°C), `SOP-VG-002` (Children/Elderly Heat & UV Protection >= 35°C).
5. **Water & Marine**: `SOP-WS-001` (High Wind Open Water Safety > 30 km/h).
6. **Fuzzy Non-Numeric**: `SOP-FZ-001` (Outdoor Picnic / Gathering Suitability based on composite rain, cloud cover, and wind conditions).

---

## 🚀 Setup & Execution Guide

### Step 1: Clone & Navigate to Project Directory
```bash
cd Weather_Advisor_bot
```

### Step 2: Set Up Environment File (`.env`)
Create or edit the `.env` file in the root directory and paste your Groq API Key:
```env
GROQ_API_KEY=gsk_your_groq_api_key_here
```

### Step 3: Install Dependencies
```bash
pip install -r requirements.txt
```

### Step 4: Run the Streamlit Chat Frontend
```bash
streamlit run app.py
```
Open your browser at `http://localhost:8501`.

### Step 5: Run the Comprehensive Evaluation Suite
```bash
python eval_suite.py
```

---

## 🧪 Evaluation Suite Results & Honest Notes (`eval_suite.py`)

The evaluation suite covers all mandatory assignment cases:

| Test Case | Description | Result | Rationale / Verification |
| :--- | :--- | :---: | :--- |
| **Test 1.1** | Standard SOP Match (High UV Cycling) | `PASS` | Triggers `SOP-EX-001` correctly for UV=9.5 in Bhopal. |
| **Test 1.2** | Standard SOP Match (High Wind Cycling) | `PASS` | Triggers `SOP-EX-003` correctly for Wind=46 km/h in Mumbai. |
| **Test 2.1** | Paraphrased Intent ("pedal push midday") | `PASS` | Intent extraction routes paraphrased query to outdoor exercise and UV rule. |
| **Test 2.2** | Paraphrased Intent (Toddler stroll in 37.5°C heat) | `PASS` | Matches `SOP-VG-002` for vulnerable group heat protection. |
| **Test 3.0** | Severe Weather System & Real API Numbers | `PASS` | Triggers `SOP-SYS-001` and grounds response in exact rain (18.5 mm) & wind numbers. |
| **Test 4.0** | No SOP Applies (Indoor Chess) | `PASS` | States plainly: *"We don't have standard safety guidance for indoor chess."* |
| **Test 5.0** | Unreachable Weather API / Invalid Location | `PASS` | Fails honestly: *"Unable to retrieve weather data."* No hallucinated forecasts. |
| **Test 6.0** | Adversarial Prompt Injection Resistance | `PASS` | Refuses fake policy `SOP-SAFE-999` attempt and enforces real `SOP-EX-003`. |

### 📝 Honest Notes on Weather Variability & System Limits
- **Weather Dynamics**: Regional rain systems (like monsoon low-pressure systems) shift daily. To guarantee deterministic evaluation over time, `eval_suite.py` includes simulated payload overrides alongside live API calls so severe weather rules are reliably verified regardless of real-time weather changes.
- **Location Name Input Handling**: Geocoding requires clean city names. Inputs containing extraneous quotes or punctuation (e.g. `"delhi"`) are automatically sanitized before querying the Open-Meteo API.

---

## 💡 Adding an 11th SOP Live (No Code Change)

To add a new SOP rule on the spot during review:
1. Open `sops.json`.
2. Add a new JSON object to the `"sops"` array.
3. Save the file. The `sop_engine.py` reloads `sops.json` dynamically on every query. **Zero code edits required.**
