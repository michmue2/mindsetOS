You are executing the /archetypeQuizCreation skill.

The user has provided a PDF file path: $ARGUMENTS

Your job is to extract principled thinking lessons from this book, produce a structured quiz JSON file in the MindsetOS format, anonymise the main character's name as "MM", and add the result to the local quiz library at `/Users/Michael_1/Desktop/MindsetOS/`.

Follow these steps precisely, in order.

---

## STEP 1 — Extract and inspect the PDF

Run:
```bash
pdftotext "$ARGUMENTS" /tmp/archetype_source.txt 2>/dev/null && wc -l /tmp/archetype_source.txt
```

Then read the first 300 lines to find:
- Book title and author name
- Table of contents (chapter names and structure)
- The book's overall subject matter

```bash
head -300 /tmp/archetype_source.txt
```

---

## STEP 2 — Identify the main character and all name variants

From what you read, identify:
- The full name of the author/main character (e.g. "Charlie Munger")
- All variants that may appear in the text: full name, first name only, last name only, possessives (e.g. "Charlie Munger", "Charlie", "Munger", "Charlie's", "Munger's")

You will replace ALL of these with "MM" in the final output. Keep this list in mind throughout.

---

## STEP 3 — Map chapter boundaries

Find the line numbers where each major chapter or section begins:
```bash
grep -n "CHAPTER\|PART \|^[A-Z][A-Z ]\{5,\}$\|^[0-9]\+\." /tmp/archetype_source.txt | head -60
```

Use these to define logical content blocks. Group chapters into **5–7 batches** of roughly equal size, each suitable for one parallel agent. Record the line ranges (e.g. 400–1200, 1201–2100, etc.).

Note which chapters are primarily **biographical/introductory** — instruct the agent for those sections to produce fewer questions (1–2 per principle) since they contain less principled thinking.

---

## STEP 4 — Determine the next available question ID

Check the existing quiz library to find the highest existing ID so new questions don't collide:
```bash
python3 -c "
import glob, json
max_id = 0
for f in glob.glob('/Users/Michael_1/Desktop/MindsetOS/app/data/*.json'):
    try:
        data = json.load(open(f))
        for q in data.get('questions', []):
            max_id = max(max_id, q.get('id', 0))
    except: pass
print('Next ID:', max_id + 1)
"
```

---

## STEP 5 — Derive a safe file slug from the book title

Create a short, lowercase, hyphenated slug for the book (e.g. "poor-charlie-almanack" → `charlie`). This will be used as:
- Output filename: `/Users/Michael_1/Desktop/MindsetOS/app/data/<slug>.json`
- Individual part files: `/Users/Michael_1/Desktop/MindsetOS/<slug>_quiz_part1.json`, etc.

---

## STEP 6 — Launch parallel agents (one per chapter batch)

Launch **one agent per content batch** simultaneously using the Agent tool with `run_in_background: true`.

Each agent receives:
- The line range to read: `sed -n 'START,ENDp' /tmp/archetype_source.txt`
- The chapter/section names it covers
- The starting question ID for its batch (pre-assigned so IDs don't overlap)
- The name variants to replace with "MM"
- The output filename for its part: e.g. `/Users/Michael_1/Desktop/MindsetOS/<slug>_quiz_part2.json`

Each agent must:

1. Read its assigned line range from `/tmp/archetype_source.txt`
2. Identify the distinct principles, mental models, or lessons in that section
3. Create **3–4 non-overlapping quiz questions per principle** (1–2 for biographical/introductory sections)
4. Replace ALL identified author name variants with "MM" in every field
5. Write a valid JSON file with this exact structure:

```json
{
  "title": "<Book Title> — Mental Model Quiz: <Section Name>",
  "source": "<Book Title> by MM (Year)",
  "part": "<Part/Chapter range description>",
  "questions": [
    {
      "id": 47,
      "principle": "The principle or mental model name",
      "question": "A specific, thought-provoking question that tests genuine understanding",
      "short_answer": "1–2 sentence core answer",
      "elaborated_answer": "3–5 sentences explaining the mental model deeply, with context and implications",
      "chapter": "Part X: Chapter Name",
      "section": "Chapter Name — Sub-section Name"
    }
  ]
}
```

**Question quality rules:**
- Questions must test understanding of the principle, not just recall of a quote
- Prefer counterintuitive insights over obvious ones
- short_answer should be usable as a standalone flashcard answer
- elaborated_answer should explain the *why* and *implications*, not just restate the answer
- Do NOT use real author names anywhere — always use "MM"

---

## STEP 7 — Wait for all agents to complete, then merge

Once all agents finish, merge all part files into a single complete file:

```python
import json, glob

slug = "<slug>"  # replace with actual slug
base = "/Users/Michael_1/Desktop/MindsetOS"

part_files = sorted(glob.glob(f"{base}/{slug}_quiz_part*.json"))
all_questions = []
for f in part_files:
    data = json.load(open(f))
    all_questions.extend(data["questions"])

# Re-number sequentially from the next available ID
next_id = <next_id>  # from Step 4
for i, q in enumerate(all_questions):
    q["id"] = next_id + i

complete = {
    "title": "<Book Title> — Mental Model Quiz: Complete",
    "source": "<Book Title> by MM (<Year>)",
    "total_questions": len(all_questions),
    "questions": all_questions
}

out_path = f"{base}/app/data/{slug}.json"
json.dump(complete, open(out_path, "w"), indent=2)
print(f"Saved {len(all_questions)} questions to {out_path}")
```

---

## STEP 8 — Run a final name-scrub pass on the output file

Even after agent anonymisation, run a final replacement pass on the merged file to catch any missed variants:

```python
import json

slug = "<slug>"
path = f"/Users/Michael_1/Desktop/MindsetOS/app/data/{slug}.json"

# Build replacement list from the name variants identified in Step 2
# Order: longest/most specific first, then last names, then first names
REPLACEMENTS = [
    ("Full Name's", "MM's"),
    ("Full Name",   "MM"),
    ("LastName's",  "MM's"),
    ("LastName",    "MM"),
    ("FirstName's", "MM's"),
    ("FirstName",   "MM"),
]

with open(path) as f:
    content = f.read()

for old, new in REPLACEMENTS:
    content = content.replace(old, new)

with open(path, "w") as f:
    f.write(content)

# Verify
remaining = [old for old, _ in REPLACEMENTS if old in content]
if remaining:
    print("WARNING — still found:", remaining)
else:
    print("✓ Clean — no real names remain")
```

---

## STEP 9 — Report to the user

Tell the user:
- Book title and author (as detected)
- Total questions created
- Output file path: `app/data/<slug>.json`
- The archetype config block they need to add to `app/index.html` to make it appear in the app:

```javascript
{
  id: '<slug>',
  name: 'The <Archetype Name>',      // suggest a fitting archetype label
  emoji: '<fitting emoji>',
  color: '<hex>',                     // suggest a colour that fits the book's theme
  bg: '<light hex>',
  darkBg: '<dark hex>',
  description: '<2–3 word descriptors>',
  dataFile: 'data/<slug>.json',
},
```

Ask the user to confirm the archetype name, emoji, and colours before you add it to `index.html`. Once confirmed, insert it into the `ARCHETYPES` array in `app/index.html` and commit + push to GitHub.

---

## Key constraints (apply throughout)

- Never use the real author name in any output field — always "MM"
- Maintain the exact JSON structure used by existing files
- IDs must be unique across the entire library (use the next available ID from Step 4)
- Focus on **principled thinking** — skip biographical trivia unless it directly reveals a principle
- Each question must be self-contained (answerable without context from adjacent cards)
