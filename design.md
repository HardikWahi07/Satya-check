# Design Document: Satya-Check Scam Detection

## Overview

Satya-Check is a serverless, event-driven scam detection system built on AWS that provides real-time analysis of forwarded content across multiple Indian languages. The system combines AWS AI/ML services (Comprehend, Rekognition, Bedrock) with a custom-built local truth intelligence layer to detect scams with regional context awareness.

The architecture follows a pipeline pattern where content flows through language detection, scam pattern analysis, local truth verification, and alert generation stages. Each stage is implemented as independent Lambda functions orchestrated through API Gateway, enabling horizontal scaling and fault isolation.

## Architecture

### High-Level Architecture

```
┌─────────────────┐
│  User/Client    │
│   (WhatsApp,    │
│   Telegram)     │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│              API Gateway (REST API)                      │
│  POST /analyze-content                                   │
│  GET  /scam-trends/{district}                           │
│  GET  /health                                            │
└────────┬────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│           Lambda: Content Orchestrator                   │
│  - Routes content by type (text/image/voice)            │
│  - Manages analysis workflow                             │
│  - Aggregates results                                    │
└────┬────────────────────────────────────────────────────┘
     │
     ├──────────────┬──────────────┬──────────────┐
     ▼              ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Lambda: │  │ Lambda:  │  │ Lambda:  │  │ Lambda:  │
│Language │  │  Image   │  │  Voice   │  │  Scam    │
│Detector │  │ Analyzer │  │Transcribe│  │ Detector │
└────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │            │              │              │
     │            ▼              ▼              │
     │       ┌─────────────────────┐           │
     │       │ Amazon Rekognition  │           │
     │       │ Amazon Transcribe   │           │
     │       └─────────────────────┘           │
     │                                          │
     ▼                                          ▼
┌──────────────────┐                  ┌─────────────────┐
│ Amazon Comprehend│                  │ Amazon Bedrock  │
│ - Language detect│                  │ - LLM reasoning │
│ - Sentiment      │                  │ - Pattern match │
│ - Entity extract │                  └────────┬────────┘
└────────┬─────────┘                           │
         │                                     │
         └──────────────┬──────────────────────┘
                        ▼
              ┌──────────────────┐
              │  Lambda: Truth   │
              │  Engine          │
              │  - Local verify  │
              │  - Cyber cell    │
              │  - Domain check  │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │    DynamoDB      │
              │ - Scam patterns  │
              │ - Regional data  │
              │ - Cyber reports  │
              └────────┬─────────┘
                       │
                       ▼
              ┌──────────────────┐
              │  Lambda: Alert   │
              │  Generator       │
              │  - Score calc    │
              │  - Localization  │
              │  - Format output │
              └──────────────────┘
```

### Data Flow

1. **Content Ingestion**: Client sends content via API Gateway
2. **Orchestration**: Content Orchestrator determines content type and routes to appropriate analyzers
3. **Parallel Analysis**: 
   - Language detection via Comprehend
   - Image analysis via Rekognition (if applicable)
   - Voice transcription via Transcribe (if applicable)
4. **Scam Detection**: Bedrock LLM analyzes content for scam patterns
5. **Truth Verification**: Truth Engine queries Regional Database for local context
6. **Scoring & Alert**: Alert Generator calculates probability score and formats response
7. **Response**: JSON response returned to client with score, classification, and reasons

## Components and Interfaces

### 1. API Gateway

**Endpoints:**

```typescript
// Content analysis endpoint
POST /analyze-content
Request: {
  content: string,
  contentType: "text" | "image" | "voice",
  imageData?: string,  // base64 encoded
  voiceData?: string,  // base64 encoded
  userDistrict?: string,
  userLanguage?: string
}
Response: {
  scamProbabilityScore: number,
  classification: "Likely Safe" | "Suspicious" | "High Scam Probability",
  reasons: string[],
  localContext?: {
    districtFlags: number,
    recentTrend: boolean,
    cyberCellAlert: boolean
  },
  alert: {
    language: string,
    message: string
  },
  processingTimeMs: number
}

// Scam trends endpoint
GET /scam-trends/{district}
Response: {
  district: string,
  activeCampaigns: number,
  topScamTypes: Array<{type: string, count: number}>,
  last7Days: Array<{date: string, count: number}>,
  heatmapData: Array<{subDistrict: string, intensity: number}>
}

// Health check
GET /health
Response: {
  status: "healthy" | "degraded",
  services: {
    comprehend: boolean,
    rekognition: boolean,
    bedrock: boolean,
    dynamodb: boolean
  }
}
```

### 2. Content Orchestrator Lambda

**Responsibility**: Routes content and manages analysis workflow

**Interface:**
```python
def lambda_handler(event, context):
    """
    Orchestrates the content analysis pipeline
    
    Args:
        event: API Gateway event with content data
        context: Lambda context
        
    Returns:
        Analysis result with scam probability and alert
    """
    pass

def route_content(content_type: str, content_data: dict) -> dict:
    """Routes content to appropriate analyzer"""
    pass

def aggregate_results(language_result, scam_result, truth_result) -> dict:
    """Combines results from all analysis stages"""
    pass
```

### 3. Language Detector Lambda

**Responsibility**: Detects language and extracts linguistic features

**Interface:**
```python
def detect_language(text: str) -> dict:
    """
    Detects language using Amazon Comprehend
    
    Args:
        text: Input text content
        
    Returns:
        {
            "language": str,  # ISO 639-1 code
            "confidence": float,
            "script": str,  # Devanagari, Tamil, etc.
            "sentiment": str,  # POSITIVE, NEGATIVE, NEUTRAL, MIXED
            "entities": list  # Extracted entities
        }
    """
    pass

def normalize_text(text: str, language: str) -> str:
    """Normalizes text for consistent processing"""
    pass
```

### 4. Image Analyzer Lambda

**Responsibility**: Extracts and analyzes visual content

**Interface:**
```python
def analyze_image(image_data: bytes) -> dict:
    """
    Analyzes image using Amazon Rekognition
    
    Args:
        image_data: Base64 decoded image bytes
        
    Returns:
        {
            "extractedText": str,
            "detectedLogos": list,
            "detectedLabels": list,
            "moderationLabels": list,
            "faceAnalysis": dict
        }
    """
    pass

def detect_forgery_indicators(image_data: bytes) -> dict:
    """Detects visual manipulation indicators"""
    pass
```

### 5. Voice Transcriber Lambda

**Responsibility**: Transcribes voice content to text

**Interface:**
```python
def transcribe_voice(voice_data: bytes, language_hint: str) -> dict:
    """
    Transcribes voice using Amazon Transcribe
    
    Args:
        voice_data: Audio file bytes
        language_hint: Expected language code
        
    Returns:
        {
            "transcript": str,
            "confidence": float,
            "detectedLanguage": str,
            "speakerLabels": list
        }
    """
    pass
```

### 6. Scam Detector Lambda

**Responsibility**: Identifies scam patterns using LLM reasoning

**Interface:**
```python
def detect_scam_patterns(content: str, language: str, metadata: dict) -> dict:
    """
    Detects scam patterns using Amazon Bedrock
    
    Args:
        content: Text content to analyze
        language: Detected language
        metadata: Additional context (entities, sentiment, etc.)
        
    Returns:
        {
            "patterns": list,  # Detected scam patterns
            "indicators": {
                "urgencyLanguage": bool,
                "otpRequest": bool,
                "impersonation": bool,
                "suspiciousUrls": list,
                "financialRequest": bool
            },
            "reasoning": str,  # LLM explanation
            "confidence": float
        }
    """
    pass

def extract_urls(content: str) -> list:
    """Extracts all URLs from content"""
    pass

def analyze_url(url: str) -> dict:
    """
    Analyzes URL for scam indicators
    
    Returns:
        {
            "domain": str,
            "domainAge": int,  # days
            "reputation": str,
            "isShortener": bool,
            "finalDestination": str,
            "typosquatting": bool,
            "trustScore": float
        }
    """
    pass
```

### 7. Truth Engine Lambda

**Responsibility**: Verifies claims against local truth sources

**Interface:**
```python
def verify_claim(content: str, district: str, language: str) -> dict:
    """
    Verifies content against regional database
    
    Args:
        content: Content to verify
        district: User's district
        language: Content language
        
    Returns:
        {
            "localFlags": int,  # Times flagged in district
            "cyberCellStatus": str,  # "reported" | "verified_fake" | "none"
            "trendingLocally": bool,
            "relatedPatterns": list,
            "officialWarnings": list,
            "lastSeenDate": str
        }
    """
    pass

def query_regional_database(pattern_hash: str, district: str) -> dict:
    """Queries DynamoDB for regional scam patterns"""
    pass

def calculate_local_weight(local_flags: int, trending: bool) -> float:
    """Calculates weight for local context in scoring"""
    pass
```

### 8. Alert Generator Lambda

**Responsibility**: Calculates final score and generates localized alerts

**Interface:**
```python
def calculate_scam_score(
    scam_patterns: dict,
    truth_verification: dict,
    url_analysis: list
) -> float:
    """
    Calculates final scam probability score (0-100)
    
    Scoring weights:
    - Scam patterns: 40%
    - Local truth verification: 35%
    - URL analysis: 15%
    - Sentiment/urgency: 10%
    """
    pass

def classify_score(score: float) -> str:
    """
    Classifies score into risk category
    
    0-30: "Likely Safe"
    30-70: "Suspicious"
    70-100: "High Scam Probability"
    """
    pass

def generate_alert(
    score: float,
    classification: str,
    reasons: list,
    language: str,
    local_context: dict
) -> dict:
    """
    Generates localized alert message
    
    Returns:
        {
            "language": str,
            "message": str,  # Localized alert text
            "actionable": list  # Suggested actions
        }
    """
    pass
```

## Data Models

### DynamoDB Tables

#### 1. ScamPatterns Table

```python
{
    "PK": "PATTERN#{pattern_hash}",  # Partition key
    "SK": "DISTRICT#{district_code}",  # Sort key
    "patternType": str,  # "text" | "image" | "voice" | "url"
    "content": str,  # Pattern content or description
    "language": str,
    "scamType": str,  # "otp_theft" | "impersonation" | "fake_offer" | etc.
    "firstSeen": str,  # ISO timestamp
    "lastSeen": str,
    "reportCount": int,
    "cyberCellVerified": bool,
    "status": str,  # "active" | "historical" | "resolved"
    "affectedDistricts": list,
    "severity": str,  # "low" | "medium" | "high" | "critical"
    "ttl": int  # DynamoDB TTL for auto-cleanup
}
```

**Indexes:**
- GSI1: `district_code-lastSeen-index` for trending queries
- GSI2: `scamType-firstSeen-index` for analytics
- GSI3: `language-reportCount-index` for language-specific patterns

#### 2. CyberCellReports Table

```python
{
    "PK": "REPORT#{report_id}",
    "SK": "TIMESTAMP#{timestamp}",
    "district": str,
    "state": str,
    "reportType": str,
    "description": str,
    "affectedUsers": int,
    "officialStatus": str,  # "investigating" | "confirmed" | "resolved"
    "relatedPatterns": list,  # Pattern hashes
    "publicWarning": str,  # Official warning text
    "language": str,
    "createdAt": str,
    "updatedAt": str
}
```

#### 3. DomainTrustScores Table

```python
{
    "PK": "DOMAIN#{domain}",
    "domainAge": int,  # days since registration
    "reputation": str,  # "trusted" | "unknown" | "suspicious" | "malicious"
    "trustScore": float,  # 0.0 to 1.0
    "scamReports": int,
    "lastChecked": str,
    "whoisData": dict,
    "sslValid": bool,
    "redirectChain": list,
    "ttl": int
}
```

#### 4. AnalyticsEvents Table

```python
{
    "PK": "EVENT#{event_id}",
    "SK": "TIMESTAMP#{timestamp}",
    "eventType": str,  # "analysis_request" | "scam_detected" | "alert_sent"
    "district": str,
    "language": str,
    "scamType": str,
    "scamScore": float,
    "processingTimeMs": int,
    "contentType": str,
    "timestamp": str,
    "ttl": int  # Keep for 90 days
}
```

### In-Memory Data Structures

```python
# Scam pattern matching result
class ScamAnalysisResult:
    patterns: List[str]
    indicators: Dict[str, bool]
    urls: List[URLAnalysis]
    reasoning: str
    confidence: float

# URL analysis result
class URLAnalysis:
    url: str
    domain: str
    domain_age_days: int
    reputation: str
    is_shortener: bool
    final_destination: str
    typosquatting_detected: bool
    trust_score: float

# Local truth verification result
class TruthVerificationResult:
    local_flags: int
    cyber_cell_status: str
    trending_locally: bool
    related_patterns: List[str]
    official_warnings: List[str]
    local_weight: float

# Final analysis output
class AnalysisOutput:
    scam_probability_score: float
    classification: str
    reasons: List[str]
    local_context: Optional[Dict]
    alert: Alert
    processing_time_ms: int

class Alert:
    language: str
    message: str
    actionable_steps: List[str]
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Language Detection and Processing Properties

**Property 1: Language detection accuracy**
*For any* text content in a supported Indian language (Hindi, Marathi, Tamil, Bengali, Hinglish, etc.), the Content_Scanner should correctly identify the language and process it using the appropriate language-specific NLP models.
**Validates: Requirements 1.1, 1.2**

**Property 2: Text normalization idempotence**
*For any* text with script variations, normalizing it once and normalizing it twice should produce identical results.
**Validates: Requirements 1.3**

**Property 3: Mixed language detection completeness**
*For any* content containing multiple languages, the Content_Scanner should identify all languages present with their respective segments.
**Validates: Requirements 1.4**

**Property 4: Extracted text pipeline consistency**
*For any* text extracted from images or transcribed from voice, it should be processed through the same language detection and scam analysis pipeline as direct text input.
**Validates: Requirements 5.5, 10.2**

### Scam Detection Properties

**Property 5: Urgency pattern detection**
*For any* content containing urgency language patterns (e.g., "act now", "limited time", "immediate action required"), the Scam_Detector should flag it as a suspicious indicator.
**Validates: Requirements 2.1**

**Property 6: Credential request detection**
*For any* content requesting OTP, password, PIN, or other authentication credentials, the Scam_Detector should classify it as high-risk behavior.
**Validates: Requirements 2.2**

**Property 7: Impersonation detection**
*For any* content impersonating government agencies, banks, or official entities, the Scam_Detector should detect and flag the impersonation attempt.
**Validates: Requirements 2.3**

**Property 8: URL extraction completeness**
*For any* content containing URLs (including shortened URLs), the Scam_Detector should extract all URLs for analysis.
**Validates: Requirements 2.4, 13.1**

**Property 9: Score calculation validity**
*For any* set of scam indicators, the calculated Scam_Probability_Score should be a valid number between 0 and 100 (inclusive).
**Validates: Requirements 2.5**

**Property 10: Score output completeness**
*For any* generated scam score, the output should include specific reasons listing the detected patterns and indicators.
**Validates: Requirements 4.4**

### Local Truth Intelligence Properties

**Property 11: Regional database query execution**
*For any* verifiable claim with district context, the Truth_Engine should query the Regional_Database with appropriate district-specific parameters.
**Validates: Requirements 3.1**

**Property 12: Cyber cell data retrieval**
*For any* content matching patterns reported by Cyber_Cell units, the Truth_Engine should retrieve and include the official status and warnings in the verification result.
**Validates: Requirements 3.2**

**Property 13: Local trend identification**
*For any* content matching patterns trending in a specific region, the Truth_Engine should identify the trend and include frequency data.
**Validates: Requirements 3.3**

**Property 14: Local source prioritization**
*For any* claim with both local and national verification sources available, the local sources should have higher weight in the final verification result.
**Validates: Requirements 3.4**

**Property 15: Database entry metadata completeness**
*For any* entry stored in the Regional_Database, it should include valid timestamps (firstSeen, lastSeen) and required metadata (district, language, scamType).
**Validates: Requirements 3.5, 7.1**

### Scoring and Classification Properties

**Property 16: Score classification consistency**
*For any* Scam_Probability_Score, the classification should be: "Likely Safe" for scores 0-30, "Suspicious" for scores 30-70, and "High Scam Probability" for scores 70-100.
**Validates: Requirements 4.1, 4.2, 4.3**

**Property 17: Local pattern weighting**
*For any* identical scam pattern, the score should be higher when local context (district flags, trending locally) is present compared to when it is absent.
**Validates: Requirements 4.5**

### Image Analysis Properties

**Property 18: Image text extraction**
*For any* image content, the Content_Scanner should attempt OCR text extraction using Rekognition and return extracted text (which may be empty for images without text).
**Validates: Requirements 5.1**

**Property 19: Logo verification execution**
*For any* image with detected logos or branding, the Content_Scanner should perform authenticity verification against known legitimate sources.
**Validates: Requirements 5.2**

**Property 20: Visual forgery detection**
*For any* image showing government notices, certificates, or official documents, the Scam_Detector should analyze for visual forgery indicators.
**Validates: Requirements 5.3**

**Property 21: Manipulation indicator detection**
*For any* image with deepfake or manipulation indicators, the Scam_Detector should flag potential manipulation in the analysis result.
**Validates: Requirements 5.4**

### Alert Generation Properties

**Property 22: Alert language consistency**
*For any* analyzed content in language X, the generated alert should also be in language X.
**Validates: Requirements 6.1**

**Property 23: High-risk alert completeness**
*For any* content classified as "High Scam Probability", the alert should include specific warning reasons explaining why it was flagged.
**Validates: Requirements 6.3**

**Property 24: Local context inclusion**
*For any* analysis with available local context (district flags, cyber cell alerts), the generated alert should include district-specific information.
**Validates: Requirements 6.4**

### Database Management Properties

**Property 25: Cyber cell update timeliness**
*For any* Cyber_Cell report received, the Regional_Database should be updated within 5 minutes of receipt.
**Validates: Requirements 7.2**

**Property 26: Inactive pattern status transition**
*For any* scam pattern marked as inactive, its status should change to "historical" but the record should remain in the database for trend analysis.
**Validates: Requirements 7.3**

**Property 27: Scam type segregation**
*For any* stored scam pattern, it should be tagged with its type (text, image, voice) to enable type-specific queries.
**Validates: Requirements 7.4**

**Property 28: Multi-dimensional query support**
*For any* query to the Regional_Database using district, state, language, or scam type parameters, appropriate results should be returned.
**Validates: Requirements 7.5**

### Resilience and Error Handling Properties

**Property 29: Exponential backoff retry**
*For any* failed AWS service API call, the system should retry up to 3 times with exponentially increasing delays between attempts.
**Validates: Requirements 8.6, 14.1**

**Property 30: Response time bounds**
*For any* text content analysis request, the system should return results within 3 seconds, and for image content within 5 seconds.
**Validates: Requirements 9.1, 9.2**

**Property 31: Database query performance**
*For any* DynamoDB query using appropriate indexes, the query execution time should be under 1 second.
**Validates: Requirements 9.5**

**Property 32: Language detection fallback**
*For any* content where language detection fails, the Content_Scanner should default to English processing and log the failure.
**Validates: Requirements 14.2**

**Property 33: Database unavailability graceful degradation**
*For any* request when the Regional_Database is unavailable, the system should use cached patterns and include a reduced-accuracy notification in the response.
**Validates: Requirements 14.3**

**Property 34: Timeout partial results**
*For any* analysis operation exceeding time thresholds, the system should return partial results with a timeout indicator rather than failing completely.
**Validates: Requirements 14.4**

**Property 35: Error logging completeness**
*For any* component failure or error condition, the system should log detailed error information including timestamp, component, error type, and context.
**Validates: Requirements 14.5**

### Voice Analysis Properties (Optional)

**Property 36: Voice transcription execution**
*For any* voice message content (when voice detection is enabled), the Content_Scanner should transcribe the audio and return transcript text.
**Validates: Requirements 10.1**

**Property 37: Regional language transcription**
*For any* voice content in a Regional_Language (when voice detection is enabled), the Content_Scanner should use language-specific transcription models.
**Validates: Requirements 10.3**

**Property 38: Voice manipulation detection**
*For any* voice content with synthetic speech or manipulation indicators (when voice detection is enabled), the Scam_Detector should flag the manipulation.
**Validates: Requirements 10.4**

### Analytics Properties

**Property 39: Heatmap district granularity**
*For any* analytics request, the generated heatmap should include district-level scam concentration data.
**Validates: Requirements 11.1**

**Property 40: Trend analysis time window**
*For any* trend analysis request, only scam patterns from the last 7 days should be included in emerging pattern identification.
**Validates: Requirements 11.2**

**Property 41: Report aggregation dimensions**
*For any* generated report, data should be aggregated by scam type, region, and language dimensions.
**Validates: Requirements 11.3**

**Property 42: Report data anonymization**
*For any* report containing user data, all personally identifiable information should be anonymized or removed.
**Validates: Requirements 11.5**

### Offline Mode Properties (Optional)

**Property 43: Offline cache fallback**
*For any* analysis request when internet connectivity is unavailable (with offline mode enabled), the system should use locally cached scam patterns for detection.
**Validates: Requirements 12.1**

**Property 44: Cache synchronization**
*For any* offline-to-online transition (with offline mode enabled), the local cache should synchronize with the Regional_Database to fetch latest patterns.
**Validates: Requirements 12.3**

**Property 45: Offline status indication**
*For any* alert generated in offline mode, it should include information indicating reduced functionality and offline status.
**Validates: Requirements 12.4**

### URL Analysis Properties

**Property 46: New domain flagging**
*For any* URL with a domain age less than 30 days, the Scam_Detector should flag it as suspicious.
**Validates: Requirements 13.2**

**Property 47: Domain reputation verification**
*For any* URL, the Scam_Detector should verify domain reputation against known malicious domain databases.
**Validates: Requirements 13.3**

**Property 48: URL shortener resolution**
*For any* URL using a shortener service, the Scam_Detector should resolve the final destination and analyze the target domain.
**Validates: Requirements 13.4**

**Property 49: Typosquatting detection**
*For any* URL with a domain similar to legitimate domains (typosquatting), the Scam_Detector should detect the similarity and flag it.
**Validates: Requirements 13.5**

**Property 50: Domain trust score persistence**
*For any* analyzed domain, a Trust_Score should be calculated and maintained for future reference.
**Validates: Requirements 13.6**

### Privacy and Security Properties

**Property 51: Content retention limits**
*For any* forwarded content received for analysis, the original content should not be stored beyond the analysis duration.
**Validates: Requirements 15.1**

**Property 52: Pattern storage anonymization**
*For any* scam pattern stored in the database, it should not contain personally identifiable information.
**Validates: Requirements 15.2**

**Property 53: Temporary content cleanup**
*For any* temporary content created during analysis, it should be deleted within 24 hours of analysis completion.
**Validates: Requirements 15.5**

## Error Handling

### Error Categories

1. **AWS Service Errors**
   - Comprehend API failures (rate limits, service unavailable)
   - Rekognition API failures (invalid image format, size limits)
   - Bedrock API failures (model unavailable, token limits)
   - DynamoDB errors (throttling, provisioned throughput exceeded)
   - Transcribe errors (unsupported audio format, duration limits)

2. **Data Validation Errors**
   - Invalid content format
   - Missing required fields
   - Content size exceeds limits
   - Unsupported language
   - Malformed URLs

3. **Processing Errors**
   - Language detection failure
   - OCR extraction failure
   - Transcription failure
   - Pattern matching timeout
   - Score calculation errors

4. **Database Errors**
   - Connection failures
   - Query timeouts
   - Data inconsistency
   - Index unavailable

### Error Handling Strategies

**Retry with Exponential Backoff:**
```python
def retry_with_backoff(func, max_attempts=3, base_delay=1):
    """
    Retries function with exponential backoff
    
    Delays: 1s, 2s, 4s
    """
    for attempt in range(max_attempts):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_attempts - 1:
                raise
            delay = base_delay * (2 ** attempt)
            time.sleep(delay)
            log_retry(func.__name__, attempt + 1, delay)
```

**Graceful Degradation:**
```python
def analyze_with_fallback(content, district):
    """
    Provides degraded service when components fail
    """
    try:
        # Try full analysis with all services
        return full_analysis(content, district)
    except DatabaseUnavailable:
        # Fall back to cached patterns
        log_warning("Database unavailable, using cache")
        return cached_analysis(content, reduced_accuracy=True)
    except LanguageDetectionFailed:
        # Default to English processing
        log_warning("Language detection failed, defaulting to English")
        return analyze_as_english(content)
```

**Partial Results:**
```python
def analyze_with_timeout(content, timeout_seconds=3):
    """
    Returns partial results if processing exceeds timeout
    """
    result = AnalysisResult()
    
    with timeout(timeout_seconds):
        try:
            result.language = detect_language(content)
            result.scam_patterns = detect_patterns(content)
            result.truth_verification = verify_truth(content)
            result.score = calculate_score(result)
        except TimeoutError:
            result.partial = True
            result.timeout_indicator = True
            log_warning("Analysis timeout, returning partial results")
    
    return result
```

**Error Response Format:**
```python
{
    "success": False,
    "error": {
        "code": "LANGUAGE_DETECTION_FAILED",
        "message": "Unable to detect content language",
        "details": "Comprehend API returned error: ...",
        "fallback_used": True,
        "retry_after": 60  # seconds
    },
    "partial_result": {
        # Any partial analysis completed before error
    }
}
```

### Circuit Breaker Pattern

For external service calls, implement circuit breaker to prevent cascading failures:

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitBreakerOpen("Service temporarily unavailable")
        
        try:
            result = func()
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
            return result
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
            raise
```

## Testing Strategy

### Dual Testing Approach

The Satya-Check system requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests** focus on:
- Specific examples of scam patterns (known phishing messages)
- Edge cases (empty content, malformed URLs, unsupported formats)
- Error conditions (API failures, timeouts, invalid inputs)
- Integration points (AWS service mocking, database operations)
- Specific language examples (Hindi urgency phrases, Tamil impersonation)

**Property-Based Tests** focus on:
- Universal properties across all inputs (score ranges, language consistency)
- Comprehensive input coverage through randomization
- Invariants that must hold (idempotence, round-trips, data integrity)
- Performance bounds across varied inputs
- Security properties (PII removal, data retention)

### Property-Based Testing Configuration

**Framework**: Use `hypothesis` for Python (Lambda functions)

**Configuration**:
```python
from hypothesis import given, settings, strategies as st

# Minimum 100 iterations per property test
@settings(max_examples=100, deadline=5000)  # 5 second deadline per example
@given(
    content=st.text(min_size=10, max_size=1000),
    language=st.sampled_from(['hi', 'mr', 'ta', 'bn', 'en'])
)
def test_property_language_consistency(content, language):
    """
    Feature: satya-check-scam-detection, Property 22: Alert language consistency
    
    For any analyzed content in language X, the generated alert should also be in language X.
    """
    result = analyze_content(content, language)
    assert result.alert.language == language
```

**Test Organization**:
- Each correctness property maps to ONE property-based test
- Tests are tagged with feature name and property number
- Tests use appropriate generators for Indian languages, scam patterns, URLs
- Tests verify properties hold across randomized inputs

**Custom Generators**:
```python
# Generator for Indian language text
@st.composite
def indian_language_text(draw):
    language = draw(st.sampled_from(['hi', 'mr', 'ta', 'bn', 'en']))
    # Generate text in specified language
    return generate_text_in_language(language)

# Generator for scam patterns
@st.composite
def scam_content(draw):
    pattern_type = draw(st.sampled_from(['urgency', 'otp_request', 'impersonation']))
    language = draw(st.sampled_from(['hi', 'mr', 'ta', 'bn', 'en']))
    return generate_scam_pattern(pattern_type, language)

# Generator for URLs with various characteristics
@st.composite
def url_with_properties(draw):
    domain_age = draw(st.integers(min_value=0, max_value=365))
    is_shortener = draw(st.booleans())
    is_typosquat = draw(st.booleans())
    return generate_url(domain_age, is_shortener, is_typosquat)
```

### Unit Testing Strategy

**Test Categories**:

1. **Language Detection Tests**
   - Test specific Hindi, Marathi, Tamil, Bengali, Hinglish examples
   - Test mixed language content
   - Test script normalization edge cases
   - Test language detection failure handling

2. **Scam Pattern Tests**
   - Test known urgency phrases in each language
   - Test OTP/password request variations
   - Test government/bank impersonation examples
   - Test URL extraction from various formats

3. **Scoring Tests**
   - Test boundary conditions (scores at 0, 30, 70, 100)
   - Test classification transitions
   - Test local weighting calculations
   - Test score with missing components

4. **Database Tests**
   - Test CRUD operations for scam patterns
   - Test query performance with indexes
   - Test data consistency
   - Test TTL cleanup

5. **Integration Tests**
   - Test end-to-end content analysis flow
   - Test AWS service integration (with mocks)
   - Test error propagation through pipeline
   - Test timeout handling

6. **Security Tests**
   - Test PII removal from stored patterns
   - Test content deletion after analysis
   - Test data anonymization in reports
   - Test TLS connection requirements

**Example Unit Tests**:
```python
def test_hindi_urgency_detection():
    """Test detection of Hindi urgency phrases"""
    content = "तुरंत कार्रवाई करें! आपका खाता बंद हो जाएगा!"
    result = detect_scam_patterns(content, language='hi')
    assert result.indicators['urgencyLanguage'] == True

def test_score_classification_boundary():
    """Test classification at boundary scores"""
    assert classify_score(30.0) == "Suspicious"
    assert classify_score(29.9) == "Likely Safe"
    assert classify_score(70.0) == "High Scam Probability"
    assert classify_score(69.9) == "Suspicious"

def test_url_extraction_edge_cases():
    """Test URL extraction from various formats"""
    content = "Visit http://example.com or bit.ly/abc123 or www.test.com"
    urls = extract_urls(content)
    assert len(urls) == 3
    assert "http://example.com" in urls
    assert "bit.ly/abc123" in urls
    assert "www.test.com" in urls

def test_database_unavailable_fallback():
    """Test graceful degradation when database is down"""
    with mock_database_unavailable():
        result = analyze_content("test content", district="Mumbai")
        assert result.success == True
        assert result.reduced_accuracy == True
        assert result.fallback_used == True
```

### Performance Testing

While not part of unit/property tests, performance requirements should be validated:

- Load testing: 1000 concurrent requests
- Response time: <3s for text, <5s for images
- Database query time: <1s with indexes
- Memory usage under sustained load
- Lambda cold start optimization

### Test Coverage Goals

- Unit test coverage: >80% for core logic
- Property test coverage: All 53 properties implemented
- Integration test coverage: All API endpoints and workflows
- Error path coverage: All error handling branches tested

