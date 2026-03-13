<![CDATA[# 🤝 How to Add Content to This System

## 📥 Method 1: Bulk Questions (Recommended)

1. Copy questions from LinkedIn/website/anywhere
2. Paste them in chat using the format from [`PROMPT_TEMPLATE.md`](./PROMPT_TEMPLATE.md)
3. The system will auto-organize them into correct categories with in-depth answers

## 📥 Method 2: Manual Addition

1. Go to the relevant category folder under `questions/`
2. Open the topic file (e.g., `arrays-strings.md`)
3. Copy the question template from inside the file (see HTML comment block)
4. Fill in your question, answer, code, and tags

## 📁 Folder Map

```
INTERVIEW_PREP/
├── README.md                    ← Main hub with navigation
├── PROMPT_TEMPLATE.md           ← Template for bulk adding questions
├── CONTRIBUTING.md              ← This file
├── .gitignore                   ← Git ignore config
│
├── questions/                   ← All interview questions
│   ├── dsa/                     ← Data Structures & Algorithms
│   ├── languages/               ← Programming Languages
│   ├── web-dev/                 ← Web Development
│   ├── database/                ← Database
│   ├── cloud-devops/            ← Cloud & DevOps
│   ├── system-design/           ← System Design
│   ├── oops-patterns/           ← OOPs & Design Patterns
│   ├── os-networking/           ← OS & Networking
│   ├── security/                ← Security
│   ├── testing/                 ← Testing
│   ├── mobile/                  ← Mobile Development
│   ├── ai-ml/                   ← AI/ML
│   ├── hr-behavioral/           ← HR & Behavioral
│   ├── aptitude/                ← Aptitude & Puzzles
│   └── company-specific/        ← Company-wise questions
│
├── tips-and-tricks/             ← Interview & career tips
├── roadmaps/                    ← Learning roadmaps
├── cheat-sheets/                ← Quick reference sheets
├── code-samples/                ← Reusable code snippets
├── images/                      ← Diagrams & visual assets
├── daily-updates/               ← What's new log
└── resources/                   ← External links & books
```

## 🏷️ Tagging Convention

Use these tags in questions:

- **Topic:** `#arrays` `#react` `#sql` `#docker`
- **Difficulty:** `#easy` `#medium` `#hard`
- **Company:** `#google` `#amazon` `#microsoft`
- **Priority:** `#must-know` `#frequently-asked` `#rare`
- **Source:** `#linkedin` `#leetcode` `#interview-experience`

## 📊 Difficulty Emojis

| Emoji | Meaning |
|-------|---------|
| 🟢 | Easy |
| 🟡 | Medium |
| 🔴 | Hard |
| ⭐ | Must Know |
| 🔥 | Frequently Asked |
]]>