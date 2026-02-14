# Implementation Plan: Satya-Check Scam Detection

## Overview

This implementation plan breaks down the Satya-Check scam detection system into incremental coding tasks. The system will be built using JavaScript (Node.js) for AWS Lambda functions, with AWS SDK v3 for service integration. The implementation follows a bottom-up approach, starting with core utilities and data models, then building individual Lambda functions, and finally wiring everything together through API Gateway.

## Tasks

- [ ] 1. Set up project structure and core dependencies
  - Create directory structure for Lambda functions
  - Initialize package.json with AWS SDK v3 dependencies (@aws-sdk/client-comprehend, @aws-sdk/client-rekognition, @aws-sdk/client-bedrock-runtime, @aws-sdk/client-dynamodb, @aws-sdk/client-transcribe)
  - Set up shared utilities directory for common code
  - Configure ESLint and testing framework (Jest with @fast-check/jest for property-based testing)
  - Create DynamoDB table schemas (ScamPatterns, CyberCellReports, DomainTrustScores, AnalyticsEvents)
  - _Requirements: 8.4, 8.5_

- [ ] 2. Implement core data models and utilities
  - [ ] 2.1 Create data model classes
    - Implement ScamAnalysisResult, URLAnalysis, TruthVerificationResult, AnalysisOutput, Alert classes
    - Add validation methods for each model
    - _Requirements: All requirements (foundational)_
  
  - [ ] 2.2 Create URL extraction and parsing utility
    - Implement extractUrls() function with regex for various URL formats
    - Handle shortened URLs, www prefixes, and protocol-less URLs
    - _Requirements: 2.4, 13.1_
  
  - [ ]* 2.3 Write property test for URL extraction
    - **Property 8: URL extraction completeness**
    - **Validates: Requirements 2.4, 13.1**
  
  - [ ] 2.4 Create text normalization utilities
    - Implement normalizeText() for Devanagari, Tamil, Bengali scripts
    - Handle Unicode normalization and whitespace cleanup
    - _Requirements: 1.3_
  
  - [ ]* 2.5 Write property test for text normalization
    - **Property 2: Text normalization idempotence**
    - **Validates: Requirements 1.3**
  
  - [ ] 2.6 Create error handling utilities
    - Implement retry with exponential backoff function
    - Create circuit breaker class for AWS service calls
    - Implement error logging with structured format
    - _Requirements: 8.6, 14.1, 14.5_
  
  - [ ]* 2.7 Write property test for retry logic
    - **Property 29: Exponential backoff retry**
    - **Validates: Requirements 8.6, 14.1**

- [ ] 3. Implement Language Detector Lambda function
  - [ ] 3.1 Create languageDetector Lambda handler
    - Implement detectLanguage() using AWS Comprehend DetectDominantLanguage API
    - Implement detectSentiment() using Comprehend DetectSentiment API
    - Implement detectEntities() using Comprehend DetectEntities API
    - Add support for Hindi, Marathi, Tamil, Bengali, English
    - _Requirements: 1.1, 1.2_
  
  - [ ] 3.2 Add mixed language detection
    - Implement logic to segment content by language
    - Detect all languages present in mixed content
    - _Requirements: 1.4_
  
  - [ ]* 3.3 Write property tests for language detection
    - **Property 1: Language detection accuracy**
    - **Property 3: Mixed language detection completeness**
    - **Validates: Requirements 1.1, 1.2, 1.4**
  
  - [ ] 3.4 Add error handling and fallback
    - Implement fallback to English when detection fails
    - Add error logging
    - _Requirements: 14.2_
  
  - [ ]* 3.5 Write property test for language detection fallback
    - **Property 32: Language detection fallback**
    - **Validates: Requirements 14.2**

- [ ] 4. Implement URL Analyzer module
  - [ ] 4.1 Create URL analysis functions
    - Implement analyzeDomain() to check domain age via WHOIS or external API
    - Implement checkDomainReputation() using threat intelligence APIs
    - Implement resolveShortUrl() to expand shortened URLs
    - Implement detectTyposquatting() using Levenshtein distance
    - _Requirements: 13.2, 13.3, 13.4, 13.5_
  
  - [ ] 4.2 Implement domain trust score calculation
    - Calculate trust score based on age, reputation, SSL validity
    - Store and retrieve trust scores from DynamoDB
    - _Requirements: 13.6_
  
  - [ ]* 4.3 Write property tests for URL analysis
    - **Property 46: New domain flagging**
    - **Property 47: Domain reputation verification**
    - **Property 48: URL shortener resolution**
    - **Property 49: Typosquatting detection**
    - **Property 50: Domain trust score persistence**
    - **Validates: Requirements 13.2, 13.3, 13.4, 13.5, 13.6**

- [ ] 5. Implement Scam Detector Lambda function
  - [ ] 5.1 Create scamDetector Lambda handler
    - Implement detectScamPatterns() using AWS Bedrock (Claude or Llama model)
    - Create prompt template for scam detection in multiple languages
    - Parse LLM response to extract patterns and reasoning
    - _Requirements: 2.1, 2.2, 2.3, 2.6_
  
  - [ ] 5.2 Implement scam indicator detection
    - Detect urgency language patterns (regex + LLM)
    - Detect OTP/password requests (keyword matching + context)
    - Detect impersonation attempts (entity matching)
    - Integrate URL analysis from URL Analyzer module
    - _Requirements: 2.1, 2.2, 2.3, 2.4_
  
  - [ ] 5.3 Implement score calculation
    - Calculate base scam probability score from indicators
    - Ensure score is always between 0-100
    - Include detected patterns in output
    - _Requirements: 2.5, 4.4_
  
  - [ ]* 5.4 Write property tests for scam detection
    - **Property 5: Urgency pattern detection**
    - **Property 6: Credential request detection**
    - **Property 7: Impersonation detection**
    - **Property 9: Score calculation validity**
    - **Property 10: Score output completeness**
    - **Validates: Requirements 2.1, 2.2, 2.3, 2.5, 4.4**

- [ ] 6. Checkpoint - Ensure core detection logic works
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Implement DynamoDB integration layer
  - [ ] 7.1 Create Regional Database query functions
    - Implement queryScamPatterns() with district and language filters
    - Implement storeSc amPattern() with metadata
    - Implement updateCyberCellReport() with timestamp tracking
    - Implement queryDomainTrustScore() and storeDomainTrustScore()
    - Use appropriate partition keys and GSIs for efficient queries
    - _Requirements: 3.1, 7.1, 7.2, 7.5, 13.6_
  
  - [ ] 7.2 Implement pattern status management
    - Implement markPatternInactive() to change status to "historical"
    - Ensure inactive patterns remain in database
    - _Requirements: 7.3_
  
  - [ ] 7.3 Add scam type tagging
    - Tag patterns as "text", "image", or "voice"
    - Enable type-specific queries
    - _Requirements: 7.4_
  
  - [ ]* 7.4 Write property tests for database operations
    - **Property 15: Database entry metadata completeness**
    - **Property 25: Cyber cell update timeliness**
    - **Property 26: Inactive pattern status transition**
    - **Property 27: Scam type segregation**
    - **Property 28: Multi-dimensional query support**
    - **Validates: Requirements 3.1, 3.5, 7.1, 7.2, 7.3, 7.4, 7.5**
  
  - [ ]* 7.5 Write property test for query performance
    - **Property 31: Database query performance**
    - **Validates: Requirements 9.5**

- [ ] 8. Implement Truth Engine Lambda function
  - [ ] 8.1 Create truthEngine Lambda handler
    - Implement verifyClaim() to query Regional Database
    - Implement queryRegionalDatabase() with pattern hashing
    - Calculate local flags count for district
    - _Requirements: 3.1_
  
  - [ ] 8.2 Add Cyber Cell data retrieval
    - Query CyberCellReports table for matching patterns
    - Retrieve official status and warnings
    - _Requirements: 3.2_
  
  - [ ] 8.3 Implement local trend detection
    - Query patterns by district and time range
    - Identify trending patterns (high frequency in last 7 days)
    - _Requirements: 3.3_
  
  - [ ] 8.4 Implement local source prioritization
    - Calculate local weight based on flags and trending status
    - Prioritize local sources over national in verification result
    - _Requirements: 3.4_
  
  - [ ]* 8.5 Write property tests for truth verification
    - **Property 11: Regional database query execution**
    - **Property 12: Cyber cell data retrieval**
    - **Property 13: Local trend identification**
    - **Property 14: Local source prioritization**
    - **Validates: Requirements 3.1, 3.2, 3.3, 3.4**
  
  - [ ] 8.6 Add database unavailability handling
    - Implement cache fallback when database is down
    - Add reduced accuracy notification
    - _Requirements: 14.3_
  
  - [ ]* 8.7 Write property test for graceful degradation
    - **Property 33: Database unavailability graceful degradation**
    - **Validates: Requirements 14.3**

- [ ] 9. Implement Alert Generator Lambda function
  - [ ] 9.1 Create alertGenerator Lambda handler
    - Implement calculateScamScore() combining scam patterns, truth verification, URL analysis
    - Use weighted scoring: patterns 40%, local truth 35%, URLs 15%, sentiment 10%
    - _Requirements: 4.5_
  
  - [ ] 9.2 Implement score classification
    - Implement classifyScore() with boundaries: 0-30 Safe, 30-70 Suspicious, 70-100 High Risk
    - _Requirements: 4.1, 4.2, 4.3_
  
  - [ ]* 9.3 Write property tests for scoring
    - **Property 16: Score classification consistency**
    - **Property 17: Local pattern weighting**
    - **Validates: Requirements 4.1, 4.2, 4.3, 4.5**
  
  - [ ] 9.4 Implement alert message generation
    - Create alert templates for each supported language
    - Implement generateAlert() to format localized messages
    - Ensure alert language matches content language
    - _Requirements: 6.1_
  
  - [ ] 9.5 Add conditional alert content
    - Include warning reasons for "High Scam Probability" classification
    - Include district-specific information when local context available
    - _Requirements: 6.3, 6.4_
  
  - [ ]* 9.6 Write property tests for alert generation
    - **Property 22: Alert language consistency**
    - **Property 23: High-risk alert completeness**
    - **Property 24: Local context inclusion**
    - **Validates: Requirements 6.1, 6.3, 6.4**

- [ ] 10. Implement Image Analyzer Lambda function
  - [ ] 10.1 Create imageAnalyzer Lambda handler
    - Implement analyzeImage() using AWS Rekognition DetectText API
    - Implement detectLogos() using Rekognition DetectLabels API
    - Implement detectModerationLabels() for inappropriate content
    - _Requirements: 5.1, 5.2_
  
  - [ ] 10.2 Add forgery detection
    - Analyze image for manipulation indicators
    - Detect fake government notices and certificates
    - _Requirements: 5.3, 5.4_
  
  - [ ] 10.3 Integrate extracted text with language pipeline
    - Pass extracted text through language detection
    - Ensure same processing as direct text input
    - _Requirements: 5.5_
  
  - [ ]* 10.4 Write property tests for image analysis
    - **Property 18: Image text extraction**
    - **Property 19: Logo verification execution**
    - **Property 20: Visual forgery detection**
    - **Property 21: Manipulation indicator detection**
    - **Property 4: Extracted text pipeline consistency (partial)**
    - **Validates: Requirements 5.1, 5.2, 5.3, 5.4, 5.5**

- [ ] 11. Implement Voice Transcriber Lambda function (Optional)
  - [ ] 11.1 Create voiceTranscriber Lambda handler
    - Implement transcribeVoice() using AWS Transcribe StartTranscriptionJob API
    - Support Hindi, Marathi, Tamil, Bengali, English
    - Poll for transcription completion
    - _Requirements: 10.1, 10.3_
  
  - [ ] 11.2 Add voice manipulation detection
    - Analyze audio for synthetic speech indicators
    - Flag potential deepfake audio
    - _Requirements: 10.4_
  
  - [ ] 11.3 Integrate transcript with text pipeline
    - Pass transcript through language detection and scam detection
    - Ensure same processing as direct text
    - _Requirements: 10.2_
  
  - [ ]* 11.4 Write property tests for voice analysis
    - **Property 36: Voice transcription execution**
    - **Property 37: Regional language transcription**
    - **Property 38: Voice manipulation detection**
    - **Property 4: Extracted text pipeline consistency (partial)**
    - **Validates: Requirements 10.1, 10.2, 10.3, 10.4**

- [ ] 12. Implement Content Orchestrator Lambda function
  - [ ] 12.1 Create contentOrchestrator Lambda handler
    - Implement main lambda_handler() to receive API Gateway events
    - Parse request body and extract content, contentType, district, language
    - _Requirements: All requirements (orchestration)_
  
  - [ ] 12.2 Implement content routing logic
    - Route text content to Language Detector → Scam Detector
    - Route image content to Image Analyzer → Language Detector → Scam Detector
    - Route voice content to Voice Transcriber → Language Detector → Scam Detector
    - _Requirements: All requirements (orchestration)_
  
  - [ ] 12.3 Implement result aggregation
    - Collect results from Language Detector, Scam Detector, Truth Engine
    - Pass aggregated data to Alert Generator
    - Format final response with score, classification, reasons, alert
    - _Requirements: All requirements (orchestration)_
  
  - [ ] 12.4 Add timeout handling
    - Implement timeout for long-running operations
    - Return partial results with timeout indicator
    - _Requirements: 14.4_
  
  - [ ]* 12.5 Write property test for timeout handling
    - **Property 34: Timeout partial results**
    - **Validates: Requirements 14.4**
  
  - [ ] 12.6 Add processing time tracking
    - Track start and end time for analysis
    - Include processingTimeMs in response
    - _Requirements: 9.1, 9.2_
  
  - [ ]* 12.7 Write property test for response time
    - **Property 30: Response time bounds**
    - **Validates: Requirements 9.1, 9.2**

- [ ] 13. Checkpoint - Ensure orchestration and integration works
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 14. Implement Analytics Lambda function
  - [ ] 14.1 Create analytics Lambda handler
    - Implement generateHeatmap() to query AnalyticsEvents by district
    - Aggregate scam counts by sub-district for heatmap visualization
    - _Requirements: 11.1_
  
  - [ ] 14.2 Implement trend analysis
    - Query patterns from last 7 days
    - Identify emerging scam types and patterns
    - _Requirements: 11.2_
  
  - [ ] 14.3 Implement report generation
    - Aggregate data by scam type, region, and language
    - Generate summary statistics
    - _Requirements: 11.3_
  
  - [ ] 14.4 Add data anonymization
    - Remove PII from reports
    - Anonymize user identifiers
    - _Requirements: 11.5_
  
  - [ ]* 14.5 Write property tests for analytics
    - **Property 39: Heatmap district granularity**
    - **Property 40: Trend analysis time window**
    - **Property 41: Report aggregation dimensions**
    - **Property 42: Report data anonymization**
    - **Validates: Requirements 11.1, 11.2, 11.3, 11.5**

- [ ] 15. Implement Offline Mode support (Optional)
  - [ ] 15.1 Create local cache management
    - Implement cacheScamPatterns() to store 1000 most recent patterns locally
    - Implement loadCachedPatterns() for offline detection
    - _Requirements: 12.1, 12.2_
  
  - [ ] 15.2 Add cache synchronization
    - Implement syncCache() to update local cache when online
    - Detect connectivity changes
    - _Requirements: 12.3_
  
  - [ ] 15.3 Add offline status indication
    - Include offline mode flag in alerts
    - Add reduced functionality message
    - _Requirements: 12.4_
  
  - [ ]* 15.4 Write property tests for offline mode
    - **Property 43: Offline cache fallback**
    - **Property 44: Cache synchronization**
    - **Property 45: Offline status indication**
    - **Validates: Requirements 12.1, 12.3, 12.4**

- [ ] 16. Implement privacy and security measures
  - [ ] 16.1 Add content retention controls
    - Ensure original content is not stored beyond analysis
    - Implement automatic cleanup after processing
    - _Requirements: 15.1_
  
  - [ ] 16.2 Implement PII anonymization
    - Scan patterns for PII before storing
    - Remove phone numbers, emails, names, addresses
    - _Requirements: 15.2_
  
  - [ ] 16.3 Add temporary content cleanup
    - Implement TTL-based cleanup for temporary data
    - Set 24-hour expiration on temporary content
    - _Requirements: 15.5_
  
  - [ ]* 16.4 Write property tests for privacy
    - **Property 51: Content retention limits**
    - **Property 52: Pattern storage anonymization**
    - **Property 53: Temporary content cleanup**
    - **Validates: Requirements 15.1, 15.2, 15.5**

- [ ] 17. Set up API Gateway and Lambda integration
  - [ ] 17.1 Create API Gateway REST API
    - Define POST /analyze-content endpoint
    - Define GET /scam-trends/{district} endpoint
    - Define GET /health endpoint
    - Configure CORS for web clients
    - _Requirements: 8.5_
  
  - [ ] 17.2 Configure Lambda integrations
    - Connect /analyze-content to Content Orchestrator Lambda
    - Connect /scam-trends to Analytics Lambda
    - Connect /health to health check Lambda
    - Set up Lambda proxy integration
    - _Requirements: 8.5_
  
  - [ ] 17.3 Add request validation
    - Validate request body schema
    - Validate required fields (content, contentType)
    - Return 400 for invalid requests
    - _Requirements: All requirements (API layer)_
  
  - [ ] 17.4 Configure Lambda environment variables
    - Set DynamoDB table names
    - Set AWS region
    - Set Bedrock model ID
    - Set timeout values
    - _Requirements: All requirements (configuration)_

- [ ] 18. Create deployment infrastructure
  - [ ] 18.1 Create AWS SAM or CloudFormation template
    - Define all Lambda functions with appropriate memory and timeout
    - Define DynamoDB tables with GSIs
    - Define API Gateway with endpoints
    - Define IAM roles and policies for Lambda execution
    - _Requirements: All requirements (infrastructure)_
  
  - [ ] 18.2 Add deployment scripts
    - Create deploy.sh script for SAM/CloudFormation deployment
    - Create package.sh script to bundle Lambda functions
    - Add environment-specific configuration (dev, staging, prod)
    - _Requirements: All requirements (deployment)_

- [ ] 19. Write integration tests
  - [ ]* 19.1 Test end-to-end text analysis flow
    - Send text content through API
    - Verify response structure and score
    - Test multiple languages
    - _Requirements: All requirements (integration)_
  
  - [ ]* 19.2 Test end-to-end image analysis flow
    - Send image content through API
    - Verify OCR extraction and scam detection
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_
  
  - [ ]* 19.3 Test error handling flows
    - Test AWS service failures with mocks
    - Test database unavailability
    - Test timeout scenarios
    - Verify graceful degradation
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5_
  
  - [ ]* 19.4 Test analytics endpoints
    - Test heatmap generation
    - Test trend analysis
    - Test report generation
    - _Requirements: 11.1, 11.2, 11.3_

- [ ] 20. Write unit tests for edge cases
  - [ ]* 20.1 Test empty content handling
    - Test empty string input
    - Test whitespace-only input
    - Test null/undefined input
    - _Requirements: All requirements (edge cases)_
  
  - [ ]* 20.2 Test malformed URL handling
    - Test invalid URL formats
    - Test URLs without protocols
    - Test internationalized domain names
    - _Requirements: 13.1, 13.2, 13.3, 13.4, 13.5_
  
  - [ ]* 20.3 Test unsupported language handling
    - Test languages outside supported set
    - Test mixed unsupported languages
    - Verify fallback to English
    - _Requirements: 1.1, 1.2, 14.2_
  
  - [ ]* 20.4 Test score boundary conditions
    - Test score exactly at 0, 30, 70, 100
    - Test score just below and above boundaries
    - Verify classification correctness
    - _Requirements: 4.1, 4.2, 4.3_

- [ ] 21. Final checkpoint - Complete system validation
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties (minimum 100 iterations each)
- Unit tests validate specific examples and edge cases
- Integration tests validate end-to-end workflows
- Use AWS SDK v3 for all AWS service integrations
- Use Jest with @fast-check/jest for property-based testing
- Lambda functions should be organized in separate directories for modularity
- Shared utilities should be in a common layer or shared directory
- DynamoDB tables should use appropriate partition keys and GSIs for performance
- All Lambda functions should have appropriate IAM permissions
- Consider using Lambda layers for shared dependencies to reduce deployment size
