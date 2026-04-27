# 18 – Azure AI & Machine Learning

---

## Table of Contents
1. [Azure AI Services Overview](#1-azure-ai-services-overview)
2. [Azure Computer Vision](#2-azure-computer-vision)
3. [Azure Speech Services](#3-azure-speech-services)
4. [Azure Language Services](#4-azure-language-services)
5. [Azure OpenAI Service](#5-azure-openai-service)
6. [Azure AI Search](#6-azure-ai-search)
7. [Azure Machine Learning (Azure ML)](#7-azure-machine-learning-azure-ml)
8. [Azure AI Foundry](#8-azure-ai-foundry)
9. [Responsible AI on Azure](#9-responsible-ai-on-azure)

---

## 1. Azure AI Services Overview

Azure AI Services (formerly Cognitive Services) are pre-built AI APIs—no ML expertise required.

### Service Categories

| Category | Services | What It Does |
|---|---|---|
| **Vision** | Computer Vision, Custom Vision, Face, Document Intelligence | Analyze images, OCR, face detection |
| **Speech** | Speech-to-Text, Text-to-Speech, Speaker Recognition, Translation | Voice AI |
| **Language** | Text Analytics, Translator, LUIS/CLU, QnA | NLP tasks |
| **Decision** | Anomaly Detector, Content Moderator, Personalizer | Recommendations |
| **Azure OpenAI** | GPT-4, GPT-4o, DALL-E 3, Whisper, Embeddings | Generative AI |

### Authentication

```python
# Key-based (simpler, less secure)
from azure.cognitiveservices.vision.computervision import ComputerVisionClient
from msrest.authentication import CognitiveServicesCredentials

client = ComputerVisionClient(
    endpoint="https://myvision.cognitiveservices.azure.com/",
    credentials=CognitiveServicesCredentials("<api-key>")
)

# Managed Identity (recommended for production)
from azure.identity import DefaultAzureCredential
from azure.ai.textanalytics import TextAnalyticsClient

credential = DefaultAzureCredential()
client = TextAnalyticsClient(
    endpoint="https://mylanguage.cognitiveservices.azure.com/",
    credential=credential
)
```

---

## 2. Azure Computer Vision

### Image Analysis (Vision 4.0)

```python
from azure.ai.vision.imageanalysis import ImageAnalysisClient
from azure.ai.vision.imageanalysis.models import VisualFeatures
from azure.identity import DefaultAzureCredential

client = ImageAnalysisClient(
    endpoint="https://myvision.cognitiveservices.azure.com/",
    credential=DefaultAzureCredential()
)

result = client.analyze_from_url(
    image_url="https://example.com/photo.jpg",
    visual_features=[
        VisualFeatures.CAPTION,
        VisualFeatures.DENSE_CAPTIONS,
        VisualFeatures.OBJECTS,
        VisualFeatures.TAGS,
        VisualFeatures.PEOPLE,
        VisualFeatures.SMART_CROPS,
        VisualFeatures.READ,  # OCR
    ],
    gender_neutral_caption=True,
    smart_crops_aspect_ratios=[0.9, 1.33],
)

print(f"Caption: {result.caption.text} (confidence={result.caption.confidence:.2f})")
for tag in result.tags.list:
    print(f"  Tag: {tag.name} ({tag.confidence:.2f})")
```

### Document Intelligence (Form Recognizer)

Pre-built models: invoice, receipt, business card, ID document, W-2, 1099, bank statement, vaccination card.

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest
from azure.identity import DefaultAzureCredential

client = DocumentIntelligenceClient(
    endpoint="https://mydi.cognitiveservices.azure.com/",
    credential=DefaultAzureCredential()
)

# Analyze an invoice
poller = client.begin_analyze_document(
    model_id="prebuilt-invoice",
    body=AnalyzeDocumentRequest(url_source="https://example.com/invoice.pdf")
)
result = poller.result()

for doc in result.documents:
    invoice_total = doc.fields.get("InvoiceTotal")
    vendor_name = doc.fields.get("VendorName")
    print(f"Vendor: {vendor_name.value_string}, Total: {invoice_total.value_currency.amount}")
```

---

## 3. Azure Speech Services

### Speech-to-Text

```python
import azure.cognitiveservices.speech as speechsdk

speech_config = speechsdk.SpeechConfig(
    subscription="<api-key>",
    region="eastus"
)
speech_config.speech_recognition_language = "en-US"

audio_config = speechsdk.audio.AudioConfig(filename="audio.wav")
recognizer = speechsdk.SpeechRecognizer(speech_config=speech_config, audio_config=audio_config)

result = recognizer.recognize_once_async().get()
if result.reason == speechsdk.ResultReason.RecognizedSpeech:
    print(f"Recognized: {result.text}")
```

### Text-to-Speech with SSML

```python
import azure.cognitiveservices.speech as speechsdk

speech_config = speechsdk.SpeechConfig(subscription="<key>", region="eastus")
speech_config.speech_synthesis_voice_name = "en-US-JennyNeural"

synthesizer = speechsdk.SpeechSynthesizer(speech_config=speech_config)

ssml = """
<speak version='1.0' xmlns='http://www.w3.org/2001/10/synthesis' xml:lang='en-US'>
  <voice name='en-US-JennyNeural'>
    <prosody rate='0.9' pitch='+5%'>
      Welcome to Azure AI Speech Services!
    </prosody>
  </voice>
</speak>
"""
result = synthesizer.speak_ssml_async(ssml).get()
```

### Voice Comparison

| Voice | Style | Use Case |
|---|---|---|
| en-US-JennyNeural | Conversational | Chatbots, assistants |
| en-US-GuyNeural | Narrative | Audiobooks, eLearning |
| en-US-AriaNeural | Professional | IVR, business |
| en-US-DavisNeural | Friendly | Customer service |

---

## 4. Azure Language Services

### Text Analytics

```python
from azure.ai.textanalytics import TextAnalyticsClient
from azure.identity import DefaultAzureCredential

client = TextAnalyticsClient(
    endpoint="https://mylang.cognitiveservices.azure.com/",
    credential=DefaultAzureCredential()
)

documents = [
    "Azure is amazing! The support team was very helpful.",
    "The deployment failed and nobody responded for 3 hours."
]

# Sentiment analysis with opinion mining
response = client.analyze_sentiment(documents, show_opinion_mining=True)
for doc in response:
    print(f"Sentiment: {doc.sentiment} (pos={doc.confidence_scores.positive:.2f})")
    for sentence in doc.sentences:
        for opinion in sentence.mined_opinions:
            print(f"  Target: {opinion.target.text}, Assessment: {opinion.assessments[0].text}")

# Key phrase extraction
response = client.extract_key_phrases(documents)
for doc in response:
    print(f"Key phrases: {doc.key_phrases}")

# Named Entity Recognition
response = client.recognize_entities(documents)
for doc in response:
    for entity in doc.entities:
        print(f"  Entity: {entity.text}, Category: {entity.category}, Confidence: {entity.confidence_score:.2f}")

# PII detection and redaction
response = client.recognize_pii_entities(documents)
for doc in response:
    print(f"Redacted: {doc.redacted_text}")
```

---

## 5. Azure OpenAI Service

### Available Models

| Model | Best For | Context Window |
|---|---|---|
| **gpt-4o** | Multimodal reasoning, vision, speed | 128K tokens |
| **gpt-4o-mini** | Cost-efficient chat | 128K tokens |
| **gpt-4** | Complex reasoning | 8K / 32K tokens |
| **gpt-35-turbo** | Fast chat, summarization | 16K tokens |
| **text-embedding-ada-002** | Embeddings for RAG/search | 8K tokens |
| **text-embedding-3-large** | High-quality embeddings | 8K tokens |
| **dall-e-3** | Image generation | – |
| **whisper** | Speech transcription | – |
| **tts** | Text-to-speech | – |

### Chat Completions

```python
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://myopenai.openai.azure.com/",
    api_key="<api-key>",          # or use DefaultAzureCredential
    api_version="2024-02-01"
)

response = client.chat.completions.create(
    model="gpt-4o",               # your deployment name
    messages=[
        {"role": "system", "content": "You are a helpful Azure expert."},
        {"role": "user", "content": "Explain the difference between Azure Blob and Azure Files."}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)
print(f"Tokens used: {response.usage.total_tokens}")
```

### Streaming Response

```python
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Write a haiku about Azure."}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content is not None:
        print(chunk.choices[0].delta.content, end="", flush=True)
print()
```

### Embeddings

```python
response = client.embeddings.create(
    model="text-embedding-ada-002",
    input=["Azure Blob Storage is for unstructured data", "Azure Files is SMB-compatible"]
)

embeddings = [item.embedding for item in response.data]
# Each embedding is a list of 1536 floats
print(f"Embedding dimensions: {len(embeddings[0])}")
```

### RAG Pattern (Retrieval Augmented Generation)

```python
import numpy as np
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint="https://myopenai.openai.azure.com/",
    api_key="<key>",
    api_version="2024-02-01"
)

def get_embedding(text: str) -> list[float]:
    response = client.embeddings.create(model="text-embedding-ada-002", input=text)
    return response.data[0].embedding

def cosine_similarity(a: list, b: list) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

# Your document chunks (in practice, load from Azure AI Search or vector DB)
documents = [
    "Azure Blob Storage stores unstructured data like images and videos.",
    "Azure Files provides SMB and NFS file shares in the cloud.",
    "Azure Queue Storage stores messages for asynchronous processing."
]
doc_embeddings = [get_embedding(d) for d in documents]

def rag_query(question: str) -> str:
    question_embedding = get_embedding(question)
    
    # Find top-3 most similar documents
    similarities = [cosine_similarity(question_embedding, de) for de in doc_embeddings]
    top_indices = sorted(range(len(similarities)), key=lambda i: similarities[i], reverse=True)[:3]
    context = "\n".join(documents[i] for i in top_indices)
    
    # Generate answer grounded in context
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": f"Answer using only this context:\n{context}"},
            {"role": "user", "content": question}
        ]
    )
    return response.choices[0].message.content

print(rag_query("What should I use for storing image files?"))
```

### Rate Limit Handling

```python
import time
from openai import RateLimitError

def call_with_retry(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except RateLimitError as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # exponential backoff: 1, 2, 4, 8, 16 seconds
                print(f"Rate limit hit. Retrying in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise
```

---

## 6. Azure AI Search

Azure AI Search is a cloud-native search service supporting full-text, semantic, and vector search.

### Core Concepts

```
Data Sources (Blob, SQL, Cosmos DB, SharePoint)
       │
   Indexers  (crawl and extract)
       │
  Skillsets  (AI enrichment: OCR, NER, translation)
       │
   Indexes   (searchable store of documents)
       │
  Query Engine → Search results
```

### Create an Index

```bash
# Create search service
az search service create \
  --name mysearch \
  --resource-group myrg \
  --sku Standard \
  --location eastus \
  --partition-count 1 \
  --replica-count 3

# Create index via REST API
curl -X POST "https://mysearch.search.windows.net/indexes?api-version=2023-11-01" \
  -H "Content-Type: application/json" \
  -H "api-key: <admin-key>" \
  -d '{
    "name": "hotels",
    "fields": [
      {"name": "HotelId", "type": "Edm.String", "key": true, "filterable": true},
      {"name": "HotelName", "type": "Edm.String", "searchable": true},
      {"name": "Description", "type": "Edm.String", "searchable": true, "analyzer": "en.microsoft"},
      {"name": "Category", "type": "Edm.String", "filterable": true, "facetable": true},
      {"name": "Rating", "type": "Edm.Double", "filterable": true, "sortable": true},
      {"name": "DescriptionVector", "type": "Collection(Edm.Single)", "searchable": true,
       "dimensions": 1536, "vectorSearchProfile": "myProfile"}
    ],
    "vectorSearch": {
      "algorithms": [{"name": "hnsw-1", "kind": "hnsw"}],
      "profiles": [{"name": "myProfile", "algorithm": "hnsw-1"}]
    },
    "semantic": {
      "configurations": [{
        "name": "my-semantic-config",
        "prioritizedFields": {
          "titleField": {"fieldName": "HotelName"},
          "contentFields": [{"fieldName": "Description"}]
        }
      }]
    }
  }'
```

### Query Types

```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
from azure.identity import DefaultAzureCredential

client = SearchClient(
    endpoint="https://mysearch.search.windows.net",
    index_name="hotels",
    credential=DefaultAzureCredential()
)

# 1. Keyword search
results = client.search(search_text="luxury hotel pool", top=5)

# 2. Semantic search (requires semantic configuration)
results = client.search(
    search_text="hotel near the beach",
    query_type="semantic",
    semantic_configuration_name="my-semantic-config",
    query_caption="extractive",
    top=5
)

# 3. Vector search (pure embedding-based)
query_vector = get_embedding("hotel near the beach")  # 1536-dim embedding
results = client.search(
    search_text=None,
    vector_queries=[VectorizedQuery(
        vector=query_vector,
        k_nearest_neighbors=5,
        fields="DescriptionVector"
    )]
)

# 4. Hybrid search (keyword + vector, best of both worlds)
results = client.search(
    search_text="luxury hotel",
    vector_queries=[VectorizedQuery(
        vector=get_embedding("luxury hotel"),
        k_nearest_neighbors=50,
        fields="DescriptionVector"
    )],
    query_type="semantic",
    semantic_configuration_name="my-semantic-config",
    top=5
)

for result in results:
    print(f"{result['HotelName']} – {result['@search.score']:.2f}")
```

---

## 7. Azure Machine Learning (Azure ML)

### Workspace Components

```
Azure ML Workspace
├── Compute
│   ├── Compute Instances  (dev/notebook VMs, always-on)
│   ├── Compute Clusters   (auto-scale training clusters)
│   └── Serverless Compute (managed, no pre-create needed)
├── Data
│   ├── Datastores (link to Storage, SQL, ADLSGen2)
│   └── Data Assets (versioned data references)
├── Assets
│   ├── Models      (versioned, registered models)
│   ├── Environments (Docker image + conda dependencies)
│   └── Components  (reusable pipeline steps)
├── Jobs
│   ├── Command Job  (single training script)
│   ├── Sweep Job    (hyperparameter tuning)
│   └── Pipeline Job (multi-step orchestration)
└── Endpoints
    ├── Online Endpoints  (real-time inference)
    └── Batch Endpoints   (batch inference)
```

### Training a Model (SDK v2)

```python
from azure.ai.ml import MLClient, command
from azure.ai.ml.entities import Environment, AmlCompute
from azure.identity import DefaultAzureCredential

ml_client = MLClient(
    credential=DefaultAzureCredential(),
    subscription_id="<sub-id>",
    resource_group_name="ml-rg",
    workspace_name="my-workspace"
)

# Define training job
job = command(
    code="./src",                    # directory with training code
    command="python train.py --data ${{inputs.training_data}} --epochs ${{inputs.epochs}}",
    inputs={
        "training_data": Input(type="uri_folder", path="azureml:my-dataset:1"),
        "epochs": 50
    },
    environment="AzureML-sklearn-1.5-ubuntu20.04-py38-cpu@latest",
    compute="cpu-cluster",           # name of compute cluster
    experiment_name="my-classifier",
    display_name="sklearn-training-run"
)

returned_job = ml_client.jobs.create_or_update(job)
print(f"Job URL: {returned_job.studio_url}")
```

### Register and Deploy a Model

```python
from azure.ai.ml.entities import Model, ManagedOnlineEndpoint, ManagedOnlineDeployment, CodeConfiguration

# Register model from job output
model = ml_client.models.create_or_update(
    Model(
        name="my-classifier",
        path="azureml://jobs/<job-id>/outputs/artifacts/paths/model/",
        type="mlflow_model",
        description="Sklearn classifier v1"
    )
)

# Create online endpoint
endpoint = ManagedOnlineEndpoint(
    name="my-classifier-endpoint",
    auth_mode="key"
)
ml_client.online_endpoints.begin_create_or_update(endpoint).wait()

# Deploy model to endpoint
deployment = ManagedOnlineDeployment(
    name="blue",
    endpoint_name="my-classifier-endpoint",
    model=f"azureml:my-classifier:1",
    instance_type="Standard_DS3_v2",
    instance_count=2
)
ml_client.online_deployments.begin_create_or_update(deployment).wait()

# Send traffic to deployment
endpoint.traffic = {"blue": 100}
ml_client.online_endpoints.begin_create_or_update(endpoint).wait()
```

### CLI v2

```bash
# Create workspace
az ml workspace create \
  --name my-workspace \
  --resource-group ml-rg \
  --location eastus

# Create compute cluster
az ml compute create \
  --name cpu-cluster \
  --type AmlCompute \
  --min-instances 0 \
  --max-instances 4 \
  --size Standard_DS3_v2 \
  --workspace-name my-workspace \
  --resource-group ml-rg

# Submit training job
az ml job create \
  --file train-job.yaml \
  --workspace-name my-workspace \
  --resource-group ml-rg

# Register model
az ml model create \
  --name my-classifier \
  --path runs:/<run-id>/model \
  --type mlflow_model \
  --workspace-name my-workspace \
  --resource-group ml-rg

# Create online endpoint
az ml online-endpoint create \
  --name my-endpoint \
  --workspace-name my-workspace \
  --resource-group ml-rg

# Deploy to endpoint
az ml online-deployment create \
  --file deployment.yaml \
  --workspace-name my-workspace \
  --resource-group ml-rg
```

### AutoML

```python
from azure.ai.ml import automl, Input

# Classification with AutoML
classification_job = automl.classification(
    compute="cpu-cluster",
    experiment_name="automl-classifier",
    training_data=Input(type="mltable", path="./data/train"),
    target_column_name="label",
    primary_metric="accuracy",
    n_cross_validations=5,
    enable_model_explainability=True,
    featurization="auto"
)
classification_job.set_limits(
    timeout_minutes=60,
    trial_timeout_minutes=15,
    max_trials=20
)
returned_job = ml_client.jobs.create_or_update(classification_job)
```

---

## 8. Azure AI Foundry

Azure AI Foundry (formerly Azure AI Studio) is the unified platform for building, evaluating, and deploying AI applications.

### Model Catalog

| Provider | Models |
|---|---|
| Azure OpenAI | GPT-4o, GPT-4o-mini, GPT-4, GPT-35-turbo, DALL-E 3, Whisper, Embeddings |
| Meta | Llama 3.1 (8B, 70B, 405B) |
| Microsoft | Phi-3 (mini, small, medium, vision variants) |
| Mistral AI | Mistral Large, Mistral Small, Mixtral |
| Cohere | Command R, Command R+, Embed v3 |
| AI21 | Jamba-Instruct |
| NVIDIA | Llama models on NIM |

### Prompt Flow

Prompt Flow is a visual development tool for LLM applications (prompt engineering + evaluation + deployment).

```yaml
# flow.dag.yaml
inputs:
  question:
    type: string
outputs:
  answer:
    type: string
    reference: ${chat.output}
nodes:
- name: retrieve
  type: python
  source: {type: code, path: retrieve.py}
  inputs:
    question: ${inputs.question}
- name: chat
  type: llm
  source: {type: code, path: chat.jinja2}
  inputs:
    context: ${retrieve.output}
    question: ${inputs.question}
  connection: my-aoai-connection
  api: chat
  model: gpt-4o
  parameters:
    temperature: 0.3
    max_tokens: 500
```

```bash
# Run prompt flow
az ml flow run create \
  --flow ./my-flow \
  --data ./test-data.jsonl \
  --workspace-name my-workspace \
  --resource-group ml-rg
```

---

## 9. Responsible AI on Azure

### Microsoft's Six Responsible AI Principles

| Principle | Description |
|---|---|
| **Fairness** | AI should treat all people fairly |
| **Reliability & Safety** | AI should perform reliably and safely |
| **Privacy & Security** | AI should be secure and respect privacy |
| **Inclusiveness** | AI should empower everyone |
| **Transparency** | AI should be understandable |
| **Accountability** | People should be accountable for AI systems |

### Azure Content Safety

```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeTextOptions, TextCategory
from azure.identity import DefaultAzureCredential

client = ContentSafetyClient(
    endpoint="https://mycontentsafety.cognitiveservices.azure.com/",
    credential=DefaultAzureCredential()
)

result = client.analyze_text(AnalyzeTextOptions(
    text="User-generated content to be moderated",
    categories=[TextCategory.HATE, TextCategory.SEXUAL, TextCategory.VIOLENCE, TextCategory.SELF_HARM]
))

for category in result.categories_analysis:
    print(f"{category.category}: severity={category.severity}")
    # severity 0 = safe, 2 = low, 4 = medium, 6 = high
```

### Azure ML Responsible AI Dashboard

The Responsible AI Dashboard consolidates:

| Component | Purpose |
|---|---|
| **Model Overview** | Performance metrics across demographic groups |
| **Error Analysis** | Find where model fails (cohort-based debugging) |
| **Data Analysis** | Distribution of features, label balance |
| **Fairness** | Disparity in performance across sensitive features |
| **Explainability** | Feature importance (global + per-sample) |
| **Causal Analysis** | What-if analysis: how changing a feature affects outcome |
| **Counterfactual** | Minimum change to flip prediction |

```python
from raiwidgets import ResponsibleAIDashboard
from responsibleai import RAIInsights

# Build RAI insights
rai_insights = RAIInsights(
    model=model,
    train=train_df,
    test=test_df,
    target_column='label',
    task_type='classification',
    sensitive_features=['gender', 'age_group']
)

# Add components
rai_insights.explainer.add()
rai_insights.error_analysis.add()
rai_insights.fairness.add(
    sensitive_features=['gender'],
    fairness_metrics=['accuracy_score', 'false_positive_rate']
)
rai_insights.causal.add(treatment_features=['income', 'education'])

rai_insights.compute()

# Save for Azure ML
rai_insights.save('/tmp/rai_insights')
```
