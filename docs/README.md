# Poetiq ARC-AGI Solver — Documentation

Welcome! This folder contains everything you need to understand how the Poetiq ARC-AGI Solver works — even if you have never seen the codebase before.

---

## 📚 Documents in this Folder

| File | What it covers |
|---|---|
| [architecture.md](architecture.md) | Big-picture system design: what the components are and how they fit together |
| [workflow.md](workflow.md) | Step-by-step walkthrough of a complete solver run, from launch to Kaggle submission |
| [data_flow.md](data_flow.md) | How data (grids, prompts, code, scores) travels through the system |
| [modules.md](modules.md) | Detailed reference for every Python file in the `arc_agi/` package |
| [configuration.md](configuration.md) | Every configuration knob explained in plain English |
| [glossary.md](glossary.md) | Plain-English definitions of all key terms used in the codebase |

---

## 🧭 Recommended Reading Order

1. **[glossary.md](glossary.md)** — understand the vocabulary first
2. **[architecture.md](architecture.md)** — see the big picture
3. **[workflow.md](workflow.md)** — follow one complete run end-to-end
4. **[data_flow.md](data_flow.md)** — trace exactly what happens to each piece of data
5. **[modules.md](modules.md)** — deep dive into each module
6. **[configuration.md](configuration.md)** — learn how to tune the system

---

## 🚀 Quick-Start Reminder

```bash
# 1. Install dependencies
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 2. Add your API keys
echo "GEMINI_API_KEY=your_key_here" > .env

# 3. Run the solver
python main.py
```

Results are written to `output/submission_<timestamp>.json` in Kaggle format.
