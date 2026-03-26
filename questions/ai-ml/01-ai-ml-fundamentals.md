# 🤖 AI/ML & Generative AI — Interview Fundamentals

> Consolidated from AI Engineer / ML-focused interview experiences. Covers Machine Learning basics, Deep Learning, NLP/Transformer architecture, LLMs, prompt engineering, AI agents, and Python ecosystem — essential for any AI/ML engineering role.

> **Questions**: Q1–Q22 | **Difficulty**: Intermediate

---

<a id="q1"></a>
## Q1. Agentic AI vs Generative AI — what's the difference?

### 📝 One-Liner
**Generative AI** creates content (text, images, code) from prompts — it reacts to input. **Agentic AI** can plan, reason, use tools, and take autonomous actions toward a goal — it acts independently with loops of thought-action-observation.

### 🔑 Quick Answer
| Feature | Generative AI | Agentic AI |
|---------|--------------|------------|
| Behavior | Reactive (prompt → response) | Autonomous (goal → plan → act) |
| Decision making | Single-shot | Multi-step reasoning |
| Tool usage | No (text in → text out) | Yes (APIs, search, code execution) |
| Memory | Context window only | Can maintain working memory |
| Examples | ChatGPT, DALL-E, Copilot | AutoGPT, LangChain agents, CrewAI |

(Generative AI content create karta hai — Agentic AI khud sochta hai, plan banata hai, tools use karta hai, aur autonomously kaam karta hai.)

### ⚡ Remember
> **Generative** = content creation (single response) | **Agentic** = goal-oriented, multi-step, tool-using | Agentic wraps generative models with reasoning loops | ReAct pattern: Reason → Act → Observe → Repeat | LangChain, CrewAI, AutoGen = agentic frameworks

---

<a id="q2"></a>
## Q2. Machine Learning vs Deep Learning

### 📝 One-Liner
**ML** = algorithms that learn patterns from data (needs feature engineering). **DL** = subset of ML using neural networks with multiple layers (auto-extracts features). DL needs more data and compute but handles complex patterns (images, text, audio).

### 🔑 Quick Answer
| Feature | Machine Learning | Deep Learning |
|---------|-----------------|---------------|
| Feature engineering | Manual (human selects features) | Automatic (network learns) |
| Data requirement | Works with small-medium data | Needs large datasets |
| Compute | CPU sufficient | GPU/TPU required |
| Interpretability | More interpretable | Black box |
| Examples | Linear Regression, SVM, Random Forest | CNN, RNN, Transformer |
| Best for | Tabular data, structured problems | Images, text, audio, video |

### ⚡ Remember
> ML = broader field | DL = subset (neural networks with depth) | DL: more data + more compute = better performance | ML: better for small data, structured/tabular data | Transfer learning bridges the gap (pretrained DL models)

---

<a id="q3"></a>
## Q3. Fine-tuning — what, when, and how

### 📝 One-Liner
Fine-tuning takes a **pre-trained model** and further trains it on a domain-specific dataset — transferring general knowledge while adapting to your specific task. Much cheaper and faster than training from scratch.

### 🔑 Quick Answer
**When to fine-tune**: (1) Pre-trained model doesn't perform well enough on your domain. (2) You have domain-specific data (medical, legal, financial). (3) You need consistent output format/style. (4) Prompt engineering alone isn't sufficient.

**Types**: (1) **Full fine-tuning** — update all weights (expensive). (2) **LoRA/QLoRA** — update small adapter layers only (efficient). (3) **Instruction tuning** — train on instruction-response pairs. (4) **RLHF** — human feedback to align outputs.

(Fine-tuning ek pre-trained model ko apne specific data pe further train karna hai — pura model scratch se nahi banana padta.)

### ⚡ Remember
> **Fine-tuning** = adapt pre-trained model to specific domain | **LoRA** = parameter-efficient fine-tuning (update small adapters) | Much cheaper than training from scratch | Need: quality labeled data for your domain | Alternatives: RAG (no training needed), prompt engineering (simplest)

---

<a id="q4"></a>
## Q4. Tokenization — how text becomes numbers

### 📝 One-Liner
Tokenization converts text into numerical tokens that models can process. Methods: **Word-level** (each word = token), **Subword** (BPE/WordPiece — splits rare words into pieces), **Character-level** (each char = token). Modern LLMs use subword tokenization.

### 🔑 Quick Answer
```
Input: "unhappiness is overwhelming"

Word-level:    ["unhappiness", "is", "overwhelming"]  → [4521, 12, 8934]
Subword (BPE): ["un", "happiness", "is", "over", "whelm", "ing"]  → [234, 1567, 12, 445, 8821, 293]
Character:     ["u", "n", "h", "a", "p", ...]  → [117, 110, 104, ...]
```

**Why subword?** Handles unknown words (splits into known pieces), balances vocabulary size and sequence length, handles morphology (un + happiness).

### ⚡ Remember
> **BPE** (GPT) and **WordPiece** (BERT) = industry standard subword tokenization | 1 token ≈ 4 characters / ¾ word in English | Vocabulary size: ~50K-100K tokens | Affects: context window usage, API costs, multilingual support | Tokenization varies by model

---

<a id="q5"></a>
## Q5. Large Language Models (LLMs) — architecture and key concepts

### 📝 One-Liner
LLMs are **Transformer-based** neural networks trained on massive text corpora using **self-supervised learning** (predict next token). They learn language patterns, facts, and reasoning capabilities through scale (GPT-4: ~1.8T parameters, trained on ~13T tokens).

### 🔑 Quick Answer
**Key concepts**: (1) **Transformer architecture** — self-attention mechanism for processing sequences. (2) **Pre-training** — next-token prediction on massive data. (3) **Fine-tuning** — adapt to specific tasks. (4) **RLHF** — align with human preferences. (5) **Context window** — max tokens in input + output. (6) **Emergent abilities** — capabilities that appear only at scale (reasoning, code generation).

### ⚡ Remember
> LLM = Transformer + massive data + massive compute | Pre-training: next token prediction (self-supervised) | Fine-tuning: task-specific adaptation | RLHF: human preference alignment | Context window = input + output token limit | Key models: GPT-4, Claude, Llama, Gemini

---

<a id="q6"></a>
## Q6. Hallucination — why LLMs make things up

### 📝 One-Liner
Hallucination occurs when an LLM generates **factually incorrect, fabricated, or nonsensical** content that sounds confident and plausible. Caused by: training data gaps, statistical pattern matching (not understanding), and the model's tendency to always produce fluent output.

### 🔑 Quick Answer
**Mitigation strategies**: (1) **RAG** (Retrieval-Augmented Generation) — ground responses in retrieved documents. (2) **Temperature = 0** — reduce randomness. (3) **System prompts** — instruct model to say "I don't know". (4) **Fact-checking** — verify against knowledge base. (5) **Fine-tuning** — train on accurate domain data. (6) **Human-in-the-loop** — review before production use.

### ⚡ Remember
> Hallucination = confident but wrong output | RAG = #1 mitigation (ground in real data) | Lower temperature = less creative, more factual | Never trust LLM output for critical facts without verification | Fine-tuning on quality data reduces domain hallucination

---

<a id="q7"></a>
## Q7. Data preprocessing for ML — essential steps

### 📝 One-Liner
Data preprocessing transforms raw data into a format suitable for ML models: **Clean** (handle missing values, remove duplicates), **Transform** (normalize/scale, encode categoricals), **Engineer** features, **Split** (train/test/validation).

### 🔑 Quick Answer
**Pipeline steps**: (1) Handle missing values (impute mean/median/mode, or drop). (2) Remove duplicates and outliers. (3) Encode categorical variables (One-Hot, Label Encoding). (4) Scale numerical features (StandardScaler, MinMaxScaler). (5) Feature selection (correlation, importance). (6) Train/test split (80/20 or 70/15/15 with validation). (7) Balance classes (SMOTE, undersampling for imbalanced data).

### ⚡ Remember
> **Garbage In = Garbage Out** — preprocessing is 60-80% of ML work | Scale features for distance-based algorithms (KNN, SVM) | One-Hot for nominal categories, Label Encoding for ordinal | Train/test split BEFORE any preprocessing to avoid data leakage | Use pipelines (sklearn.Pipeline) for reproducibility

---

<a id="q8"></a>
## Q8. Feature engineering — creating meaningful features

### 📝 One-Liner
Feature engineering creates new input features from raw data to improve model performance — domain knowledge + creativity. Examples: date → day_of_week/is_weekend, address → lat/lng, text → TF-IDF/embeddings.

### 🔑 Quick Answer
**Common techniques**: (1) **Binning** — age → age_group (young/middle/senior). (2) **Interaction features** — price × quantity = total_cost. (3) **Time features** — timestamp → hour, day_of_week, is_holiday. (4) **Text features** — word count, sentiment score, TF-IDF. (5) **Aggregation** — user → avg_spend_last_30_days. (6) **Encoding** — target encoding, frequency encoding. (7) **Polynomial features** — x² for non-linear relationships.

### ⚡ Remember
> Feature engineering is often more impactful than model selection | Domain knowledge is key | Automated: featuretools, sklearn.PolynomialFeatures | Validate: check feature importance post-training | Avoid data leakage: no future information in features

---

<a id="q9"></a>
## Q9. Self-attention mechanism in Transformers

### 📝 One-Liner
Self-attention computes a **weighted relationship between every pair of tokens** in a sequence, allowing each token to "attend to" all other tokens. Uses Query (Q), Key (K), Value (V) matrices: Attention(Q,K,V) = softmax(QK^T / √d_k) × V.

### 🔑 Quick Answer
**Step by step**: (1) Each token generates Q, K, V vectors via learned weight matrices. (2) Compute attention scores: dot product of Q with all K vectors. (3) Scale by √d_k to prevent gradient issues. (4) Softmax to get attention weights (probabilities). (5) Weighted sum of V vectors = context-aware representation.

**Why it matters**: Captures long-range dependencies (unlike RNNs), parallelizable (unlike sequential processing), and forms the basis of all modern LLMs.

### ⚡ Remember
> **Q·K = relevance score** (how much to attend) | **V = the actual information** | Scale by √d_k for stable gradients | **Multi-head attention** = multiple attention patterns in parallel | O(n²) complexity with sequence length | Self-attention is what makes Transformers powerful

---

<a id="q10"></a>
## Q10. Transformer architecture — encoder, decoder, and full model

### 📝 One-Liner
**Encoder** (BERT) = processes input, creates representations (good for understanding — classification, NER). **Decoder** (GPT) = generates output auto-regressively (good for generation). **Encoder-Decoder** (T5, original Transformer) = input → output translation.

### 🔑 Quick Answer
| Architecture | Attention | Task | Models |
|-------------|-----------|------|--------|
| Encoder-only | Bidirectional self-attention | Understanding (classification, NER, QA) | BERT, RoBERTa, DeBERTa |
| Decoder-only | Causal (masked) self-attention | Generation (text, code) | GPT-4, Claude, Llama |
| Encoder-Decoder | Cross-attention + self-attention | Seq2Seq (translation, summarization) | T5, BART, mBART |

### ⚡ Remember
> **BERT** = encoder (bidirectional, understanding) | **GPT** = decoder (left-to-right, generation) | **T5** = encoder-decoder (translation, summarization) | Modern LLMs are mostly decoder-only | Encoder = bidirectional context, Decoder = causal masking

---

<a id="q11"></a>
## Q11. Positional encoding — why Transformers need it

### 📝 One-Liner
Transformers process all tokens simultaneously (no sequential order like RNNs), so they need **positional encoding** to inject information about token positions. Original: sinusoidal functions. Modern: learned positional embeddings or RoPE (Rotary Position Embedding).

### 🔑 Quick Answer
Without positional encoding, "cat sat on mat" and "mat sat on cat" would produce the same representation. **Sinusoidal encoding** uses sin/cos at different frequencies — each position gets a unique pattern, and the model can learn relative distances. **RoPE** (used in Llama, GPT-NeoX) encodes positions directly into the attention computation, enabling better length generalization.

### ⚡ Remember
> Transformers are permutation-invariant without position info | Sinusoidal = original (fixed, no training) | Learned embeddings = trainable (BERT, GPT-2) | RoPE = modern (relative positions, better extrapolation) | ALiBi = another alternative (attention bias)

---

<a id="q12"></a>
## Q12. AI Agents — architecture and tool usage

### 📝 One-Liner
AI Agents are systems that use LLMs as a **reasoning engine** to plan, execute actions using tools (APIs, databases, code execution), observe results, and iterate until a goal is achieved. Framework: LLM + Memory + Tools + Planning.

### 🔑 Quick Answer
**Agent architecture**: (1) **LLM Core** — reasoning and planning. (2) **Tools** — functions the agent can call (search, calculator, API, code interpreter). (3) **Memory** — short-term (conversation) + long-term (vector DB). (4) **Planning** — break goals into subtasks. (5) **Observation** — process tool results and decide next action.

**ReAct pattern**: Thought → Action → Observation → Thought → ... until goal achieved.

### ⚡ Remember
> Agent = LLM + Tools + Memory + Planning | **ReAct** = Reason + Act loop | Frameworks: LangChain, CrewAI, AutoGen | Tools: function calling, RAG, code execution | Memory: conversation buffer + vector DB for long-term | Guard rails needed (limit actions, human approval for sensitive ops)

---

<a id="q13"></a>
## Q13. Python libraries for ML/AI development

### 📝 One-Liner
**Core**: NumPy (arrays), Pandas (dataframes), Matplotlib/Seaborn (viz). **ML**: scikit-learn (classical ML), XGBoost (boosting). **DL**: PyTorch, TensorFlow/Keras. **NLP**: Hugging Face Transformers, spaCy. **LLM**: LangChain, OpenAI API, llamaindex.

### 🔑 Quick Answer
| Category | Library | Purpose |
|----------|---------|---------|
| Data manipulation | Pandas, NumPy | DataFrames, array operations |
| Visualization | Matplotlib, Seaborn, Plotly | Charts, statistical plots, interactive |
| Classical ML | scikit-learn | Classification, regression, clustering |
| Deep Learning | PyTorch, TensorFlow | Neural networks, GPU training |
| NLP | Hugging Face, spaCy, NLTK | Tokenization, embeddings, NER |
| LLM/GenAI | LangChain, OpenAI, llamaindex | Agent framework, API access, RAG |
| Computer Vision | OpenCV, torchvision | Image processing, object detection |
| Experiment tracking | MLflow, Weights & Biases | Model versioning, metrics tracking |

### ⚡ Remember
> **scikit-learn** for classical ML | **PyTorch** is industry standard for DL (overtook TensorFlow) | **Hugging Face** for pre-trained models + fine-tuning | **LangChain** for LLM application development | Know the ecosystem map — interviewers test breadth

---

<a id="q14"></a>
## Q14. FastAPI for ML model deployment

### 📝 One-Liner
FastAPI is a high-performance Python web framework ideal for ML model serving — async support, automatic Swagger docs, Pydantic validation, and easy Docker deployment.

### 💻 Code
```python
from fastapi import FastAPI
from pydantic import BaseModel
import joblib

app = FastAPI()
model = joblib.load("model.pkl")

class PredictionRequest(BaseModel):
    features: list[float]

class PredictionResponse(BaseModel):
    prediction: float
    confidence: float

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    prediction = model.predict([request.features])[0]
    confidence = max(model.predict_proba([request.features])[0])
    return PredictionResponse(prediction=prediction, confidence=confidence)

# Run: uvicorn main:app --host 0.0.0.0 --port 8000
# Docs: http://localhost:8000/docs (auto-generated Swagger)
```

### ⚡ Remember
> **FastAPI** = async + auto-docs + validation | **Pydantic** for request/response schemas | Load model at startup (not per request) | Docker + uvicorn for production | Add health check endpoint | Consider model versioning and A/B testing

---

<a id="q15"></a>
## Q15. Types of ML algorithms — supervised, unsupervised, reinforcement

### 📝 One-Liner
**Supervised** = labeled data (input → known output) — classification, regression. **Unsupervised** = no labels (find patterns) — clustering, dimensionality reduction. **Reinforcement** = agent learns from environment rewards — game playing, robotics.

### 🔑 Quick Answer
| Type | Data | Goal | Algorithms |
|------|------|------|-----------|
| Supervised | Labeled (X → y) | Predict outcomes | Linear/Logistic Regression, SVM, Random Forest, XGBoost |
| Unsupervised | Unlabeled (X only) | Find patterns | K-Means, DBSCAN, PCA, Autoencoders |
| Semi-supervised | Mix of labeled + unlabeled | Leverage both | Label propagation, self-training |
| Reinforcement | Reward signals | Maximize reward | Q-Learning, PPO, DQN |
| Self-supervised | Pseudo-labels from data | Pre-train representations | BERT (masked LM), GPT (next token) |

### ⚡ Remember
> **Supervised** = most common (classification + regression) | **Unsupervised** = clustering + anomaly detection | **Self-supervised** = how LLMs are pre-trained | Choose based on: labeled data availability, task type, interpretability needs | Start simple (Linear/Logistic) before complex models

---

<a id="q16"></a>
## Q16. Neural network types — CNN, RNN, LSTM, Transformer

### 📝 One-Liner
**CNN** = spatial patterns (images). **RNN** = sequential data (time series, text). **LSTM/GRU** = RNN with memory gates (solves vanishing gradient). **Transformer** = attention-based, parallel processing (replaced RNNs for most NLP tasks).

### 🔑 Quick Answer
| Architecture | Strength | Weakness | Use Case |
|-------------|----------|----------|----------|
| CNN | Local pattern detection | Fixed receptive field | Image classification, object detection |
| RNN | Sequential processing | Vanishing gradient, slow | Simple sequences (replaced by Transformers) |
| LSTM | Long-term dependencies | Sequential (slow) | Time series, legacy NLP |
| Transformer | Long-range, parallelizable | O(n²) memory for attention | NLP, code, multimodal, everything modern |

### ⚡ Remember
> **CNN** = still best for images | **Transformer** = dominant for text/code/multimodal | **LSTM** = replaced by Transformers for most NLP | Vision Transformer (ViT) = Transformers for images too | Know the evolution: RNN → LSTM → Transformer

---

<a id="q17"></a>
## Q17. Prompt engineering — techniques and best practices

### 📝 One-Liner
Prompt engineering designs effective inputs for LLMs to get desired outputs. Key techniques: **Zero-shot** (no examples), **Few-shot** (provide examples), **Chain-of-Thought** (step-by-step reasoning), **ReAct** (reasoning + actions), **System prompts** (set behavior).

### 🔑 Quick Answer
**Techniques**: (1) **Zero-shot**: "Classify this review as positive/negative: [text]". (2) **Few-shot**: Provide 2-3 examples before the actual task. (3) **CoT (Chain-of-Thought)**: "Think step by step..." — improves reasoning. (4) **System prompts**: Define role, tone, constraints. (5) **Output format**: "Respond in JSON format with fields: ...". (6) **Self-consistency**: Generate multiple answers, vote on best.

### ⚡ Remember
> **Few-shot > zero-shot** for complex tasks | **CoT** dramatically improves reasoning/math | Be specific about format and constraints | System prompt sets the "character" | Temperature controls creativity vs consistency | Iterate and test prompts systematically

---

<a id="q18"></a>
## Q18. Vector databases — what and why

### 📝 One-Liner
Vector databases store and efficiently search **high-dimensional vector embeddings** using approximate nearest neighbor (ANN) algorithms. Essential for **RAG** (Retrieval-Augmented Generation), semantic search, and recommendation systems.

### 🔑 Quick Answer
**How it works**: (1) Convert text/images to vector embeddings (using embedding models like OpenAI ada-002, sentence-transformers). (2) Store vectors in specialized database. (3) Query: convert query to vector → find nearest neighbors → return similar items.

**Popular vector DBs**: Pinecone (managed), Weaviate (open-source), Chroma (lightweight), Milvus (scalable), pgvector (PostgreSQL extension), FAISS (Facebook library).

### ⚡ Remember
> **Embedding** = dense vector representation of text/image | **ANN** = approximate nearest neighbor (fast, not exact) | Used in RAG: store documents → retrieve relevant chunks → feed to LLM | Pinecone = managed, Chroma = local dev, pgvector = if already using Postgres

---

<a id="q19"></a>
## Q19. Model evaluation metrics — classification and regression

### 📝 One-Liner
**Classification**: Accuracy, Precision, Recall, F1-Score, AUC-ROC. **Regression**: MSE, RMSE, MAE, R². Choose based on problem: imbalanced data → F1/AUC, business cost → precision vs recall trade-off.

### 🔑 Quick Answer
| Metric | Formula / Meaning | When to Use |
|--------|-------------------|-------------|
| Accuracy | Correct / Total | Balanced classes |
| Precision | TP / (TP + FP) | Cost of false positives high (spam detection) |
| Recall | TP / (TP + FN) | Cost of false negatives high (cancer detection) |
| F1-Score | 2 × (P×R)/(P+R) | Balance precision and recall |
| AUC-ROC | Area under ROC curve | Overall model performance, threshold-independent |
| RMSE | √(mean(error²)) | Regression, penalizes large errors |
| R² | 1 - SS_res/SS_tot | How well model explains variance |

### ⚡ Remember
> **Accuracy misleading** for imbalanced data (99% accuracy if 99% is one class) | **Precision** = of predicted positives, how many are correct | **Recall** = of actual positives, how many were found | **F1** = harmonic mean of P and R | Use **confusion matrix** to visualize

---

<a id="q20"></a>
## Q20. Context window and token limits in LLMs

### 📝 One-Liner
The context window is the **maximum number of tokens** (input + output) an LLM can process in a single interaction. Larger windows enable longer documents but cost more compute. GPT-4 Turbo: 128K, Claude: 200K, Gemini: 1M+.

### 🔑 Quick Answer
**Context management strategies**: (1) **Summarization** — condense long history. (2) **Chunking** — split documents into overlapping chunks for RAG. (3) **Sliding window** — keep recent messages, summarize older ones. (4) **Retrieval** — only include relevant context (RAG). (5) **Token budgeting** — allocate tokens for system prompt, context, and output.

### ⚡ Remember
> Context window = input + output tokens combined | Larger context ≠ better (model struggles with middle portions — "lost in the middle" problem) | RAG = retrieve only relevant chunks instead of stuffing everything | Token cost = proportional to context size | 1 token ≈ 4 characters in English

---

<a id="q21"></a>
## Q21. Temperature and sampling parameters in LLMs

### 📝 One-Liner
**Temperature** controls randomness: 0 = deterministic (greedy), 0.7 = balanced, 1.0+ = creative/random. **Top-p** (nucleus sampling) limits token selection to smallest set with cumulative probability p. **Top-k** limits to k most likely tokens.

### 🔑 Quick Answer
| Parameter | Low Value | High Value |
|-----------|-----------|------------|
| Temperature (0-2) | Deterministic, focused, repetitive | Creative, diverse, sometimes incoherent |
| Top-p (0-1) | Fewer token choices, focused | More token choices, diverse |
| Top-k (1-100) | Very restricted vocabulary | Wider vocabulary |

**Use cases**: Temperature 0 = code generation, factual QA. Temperature 0.3-0.7 = general conversation. Temperature 0.8-1.0 = creative writing, brainstorming.

### ⚡ Remember
> **Temperature 0** = same output every time (code, facts) | **Temperature 0.7** = good balance | **Top-p 0.9** = common default | Don't set both temp and top-p aggressively | Temperature applies BEFORE top-p/top-k filtering | Higher temp = more hallucination risk

---

<a id="q22"></a>
## Q22. RAG (Retrieval-Augmented Generation) — architecture and implementation

### 📝 One-Liner
RAG combines **retrieval** (search relevant documents from a knowledge base) with **generation** (LLM produces answers grounded in retrieved context). Solves: hallucination, knowledge cutoff, and domain-specific questions.

### 🔑 Quick Answer
**RAG pipeline**: (1) **Ingestion**: Document → Chunk → Embed → Store in vector DB. (2) **Retrieval**: User query → Embed → Vector search → Top-K relevant chunks. (3) **Generation**: Prompt = System instruction + Retrieved chunks + User query → LLM → Grounded answer.

```
User Query → Embed → Vector DB (similarity search) → Top-K Chunks
                                                          ↓
                                                  LLM Prompt = Context + Query
                                                          ↓
                                                  Grounded Answer
```

**Key decisions**: Chunk size (512-1024 tokens), overlap (10-20%), embedding model (OpenAI ada-002, sentence-transformers), retrieval method (semantic, hybrid with keyword), number of chunks (3-5).

### ⚡ Remember
> **RAG** = retrieve then generate (no fine-tuning needed) | Cheaper than fine-tuning | Easily updatable (add/remove documents) | Chunk size affects quality (too small = no context, too large = noise) | Hybrid search (semantic + keyword) > pure semantic | Evaluate: faithfulness, relevance, answer quality
