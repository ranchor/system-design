### System Design: Localization Feature for Facebook

### Problem Statement
Design a system that allows users to create posts in one language and provides an option to translate those posts into other languages of their choice. The system should efficiently store translation mappings and be scalable to handle Facebook's large user base.

### Clarification Questions
1. **Translation Scope**: Are translations needed for entire posts, or should we also support translations for individual words/phrases?
2. **Supported Languages**: How many languages should the system support?
3. **Usage Pattern**: What are the expected read/write patterns? Is it more read-heavy or write-heavy?
4. **Machine Learning**: Should we use machine learning models for translation, or rely on third-party services (e.g., Google Translate, Microsoft Translator)?

### Requirements
#### Functional Requirements
- Users can create posts in their preferred language.
- Users can choose to translate their posts into other languages.
- Store translations efficiently to avoid redundant translations.
- Provide accurate and context-aware translations.
- Allow users to manually correct translations if needed.

#### Non-Functional Requirements
- **Scalability**: Support millions of translations per day.
- **Low Latency**: Provide translations quickly, ideally within a few seconds.
- **Fault Tolerance**: Ensure the system is resilient to failures.
- **Extensibility**: Easily add support for new languages and translation models.

### Back of Envelope Estimations/Capacity Estimation & Constraints
1. **Users**: Assume 2 billion users.
2. **Posts**: Assume 1% of users create a post each day = 20 million posts/day.
3. **Translations**: Assume each post is translated into 3 languages on average = 60 million translations/day.

### High-Level System Design

#### Components
1. **API Gateway**: Handles incoming requests for post creation and translation.
2. **Translation Service**: Manages translation requests, either through ML models or third-party services.
3. **Translation Cache/Database**: Stores translations to avoid redundant translations.
4. **Post Storage**: Stores the original and translated posts.
5. **Language Model Storage**: Stores word/phrase translation mappings and ML models for translations.

### System Interface and Data Flow

1. **Post Creation**:
   - User creates a post in their preferred language.
   - The post is stored in the post storage.
   - The user can optionally request translations.

2. **Translation Request**:
   - User requests translation of their post into one or more languages.
   - The translation service checks the cache for existing translations.
   - If translation is not found in the cache, the service uses ML models or third-party APIs to generate translations.
   - The new translation is stored in the cache and post storage.

### High-Level Architecture

```plaintext
+--------------------+          +-------------------------+
|    API Gateway     | <------> |   Translation Service   |
+--------------------+          +-------------------------+
           |                              |
           v                              v
+--------------------+          +-------------------------+
|    Post Storage    | <------> | Translation Cache/DB    |
+--------------------+          +-------------------------+
           |                              |
           v                              v
+--------------------+          +-------------------------+
| Language Model/DB |          | Third-Party Translation |
+--------------------+          |      Services           |
                                           |
                                           v
                                +-------------------------+
                                |      ML Models          |
                                +-------------------------+
```

### Detailed Component Design

#### API Gateway
- **Endpoints**:
  - `POST /createPost`: For creating a new post.
  - `POST /translatePost`: For translating a post into another language.
- **Responsibilities**:
  - Authenticate requests.
  - Route requests to appropriate services.
  - Handle rate limiting and request validation.

#### Translation Service
- **Responsibilities**:
  - Manage translation requests.
  - Check translation cache/database for existing translations.
  - Generate translations using ML models or third-party APIs if needed.
  - Update the cache and post storage with new translations.

**Example Translation Flow**:
```python
class TranslationService:
    def __init__(self, cache, translation_api, ml_model):
        self.cache = cache
        self.translation_api = translation_api
        self.ml_model = ml_model

    def translate_post(self, post_id, target_language):
        # Check cache for existing translation
        cached_translation = self.cache.get(post_id, target_language)
        if cached_translation:
            return cached_translation

        # Fetch original post
        original_post = PostStorage.get_post(post_id)
        if not original_post:
            raise Exception("Post not found")

        # Generate translation using ML model or third-party API
        translation = self.ml_model.translate(original_post.text, target_language)
        if not translation:
            translation = self.translation_api.translate(original_post.text, target_language)

        # Store translation in cache and post storage
        self.cache.set(post_id, target_language, translation)
        PostStorage.save_translation(post_id, target_language, translation)

        return translation
```

#### Translation Cache/Database
- **Cache**:
  - Use a distributed cache (e.g., Redis) to store recent translations.
  - Key: `post_id:target_language`, Value: `translated_text`.

- **Database**:
  - Use a scalable database (e.g., Cassandra, DynamoDB) to store all translations.
  - Schema:
    - `post_id`: ID of the original post.
    - `language`: Language of the translation.
    - `translated_text`: Translated text.

**Example Cache Schema**:
```plaintext
+--------------+-----------------+-------------------+
|   post_id    |    language     |  translated_text  |
+--------------+-----------------+-------------------+
| post_123     |    en           | "Hello World"     |
| post_123     |    es           | "Hola Mundo"      |
| post_456     |    fr           | "Bonjour le monde"|
+--------------+-----------------+-------------------+
```

#### Post Storage
- **Responsibilities**:
  - Store original posts and their translations.
  - Provide efficient retrieval of posts and translations.

**Example Post Schema**:
```plaintext
+--------------+-----------------+-------------------+
|   post_id    |    language     |       text        |
+--------------+-----------------+-------------------+
| post_123     |    en           | "Hello World"     |
| post_123     |    es           | "Hola Mundo"      |
| post_456     |    fr           | "Bonjour le monde"|
+--------------+-----------------+-------------------+
```

#### Language Model Storage
- **Responsibilities**:
  - Store word/phrase translation mappings.
  - Store ML models for translation.
- **Implementation**:
  - Use a database or key-value store for word/phrase mappings.
  - Store ML models in a model registry (e.g., TensorFlow Serving, MLflow).

### Scaling the System

#### Read/Write Patterns
- **Read-Heavy**: The system is expected to be read-heavy, as users will frequently request translations.
- **Write Considerations**: Writes occur when new posts are created or translations are generated.

#### Scaling Strategies
- **Horizontal Scaling**: Scale the API Gateway, Translation Service, and Async Workers horizontally to handle increased load.
- **Caching**: Use distributed caching (e.g., Redis) to reduce database load and improve response times for frequently requested translations.
- **Sharding**: Shard the database to distribute the load and improve performance.
- **Load Balancing**: Use load balancers to distribute incoming requests evenly across servers.
- **Rate Limiting**: Implement rate limiting to prevent abuse and ensure fair usage of resources.

#### Machine Learning for Translation
- **Training Models**: Train custom ML models using large datasets of parallel texts.
- **Pre-Trained Models**: Use pre-trained models from frameworks like Hugging Face's Transformers.
- **Hybrid Approach**: Combine custom models with third-party translation APIs for improved accuracy.

**Example ML Translation Flow**:
```python
from transformers import MarianMTModel, MarianTokenizer

class MLTranslationModel:
    def __init__(self, model_name):
        self.tokenizer = MarianTokenizer.from_pretrained(model_name)
        self.model = MarianMTModel.from_pretrained(model_name)

    def translate(self, text, target_language):
        inputs = self.tokenizer(text, return_tensors="pt", padding=True)
        translated_tokens = self.model.generate(**inputs)
        translated_text = self.tokenizer.batch_decode(translated_tokens, skip_special_tokens=True)
        return translated_text[0]
```

### Summary
Designing a localization feature for Facebook involves handling large volumes of posts and translations efficiently. By using a combination of caching, distributed databases, and machine learning models, the system can provide accurate and scalable translations for millions of users. Key components include the API Gateway, Translation Service, Translation Cache/Database, Post Storage, and Language Model Storage. Horizontal scaling, caching, and sharding are crucial for handling the expected read-heavy load and ensuring low latency.