# Requirements Document

## Introduction

DevDebt is a web application that analyzes GitHub profiles to detect fake, padded, or inauthentic activity patterns using machine learning and behavioral analysis. The system evaluates GitHub profiles across multiple behavioral metrics to assign an authenticity score with confidence levels and detailed red flag warnings.

## Glossary

- **System**: The DevDebt GitHub Profile Authenticity Scorer application
- **Profile**: A GitHub user account with associated repositories and activity data
- **Authenticity_Score**: A numerical value from 0-100 indicating profile genuineness
- **Red_Flag**: A detected suspicious pattern with severity level
- **Feature**: A quantitative behavioral metric extracted from GitHub data
- **ML_Model**: The trained XGBoost classifier that generates authenticity predictions
- **GitHub_API**: GitHub's REST API v3 for fetching user and repository data
- **Cache**: MongoDB storage for analyzed profiles with 24-hour TTL

## Requirements

### Requirement 1: GitHub Data Collection

**User Story:** As a system administrator, I want to collect comprehensive GitHub profile data, so that the ML model has sufficient information for accurate authenticity assessment.

#### Acceptance Criteria

1. WHEN a username is provided, THE System SHALL fetch user profile data from GitHub API including account creation date, followers, following, and public repository count
2. WHEN fetching repository data, THE System SHALL retrieve up to 100 public repositories with pagination handling
3. WHEN analyzing repositories, THE System SHALL collect commit history (up to 100 recent commits per repository), language breakdown, stars, forks, and archived status
4. WHEN GitHub API rate limits are encountered, THE System SHALL implement exponential backoff and queue subsequent requests
5. WHEN API quota is exhausted, THE System SHALL return appropriate error messages with retry timing information

### Requirement 2: Feature Extraction and Analysis

**User Story:** As a data scientist, I want to extract quantitative behavioral features from raw GitHub data, so that the ML model can identify patterns indicative of authentic or suspicious activity.

#### Acceptance Criteria

1. THE System SHALL extract 23 numerical features categorized into temporal, code quality, repository, and engagement metrics
2. WHEN calculating temporal features, THE System SHALL compute account age, commit hour entropy, weekend commit ratio, commit time clustering, days since last commit, commit frequency variance, and burst activity score
3. WHEN analyzing code quality, THE System SHALL calculate average commit size, trivial commit ratio, large commit ratio, commit message length average, empty commit ratio, and commit size standard deviation
4. WHEN evaluating repositories, THE System SHALL determine fork ratio, original repository ratio, average stars per repository, zero-star repository ratio, archived repository ratio, and repository language diversity
5. WHEN measuring engagement, THE System SHALL compute follower-following ratio, contributions per repository, public gist count, and organization membership count
6. THE System SHALL normalize all features to [0, 1] range using min-max scaling with predefined bounds

### Requirement 3: Machine Learning Model Training

**User Story:** As a machine learning engineer, I want to train a classification model on labeled GitHub profile data, so that the system can accurately distinguish between authentic and suspicious profiles.

#### Acceptance Criteria

1. THE System SHALL collect training data with heuristic labels for 1000 authentic profiles and 1000 suspicious profiles
2. WHEN training the model, THE System SHALL use XGBoost classifier with specified hyperparameters including max depth of 6, 200 estimators, and learning rate of 0.05
3. WHEN handling class imbalance, THE System SHALL apply SMOTE oversampling to balance training data
4. THE System SHALL implement early stopping with 20 rounds on validation set to prevent overfitting
5. WHEN evaluating model performance, THE System SHALL achieve minimum AUC-ROC score of 0.85 on test set
6. THE System SHALL calibrate probabilities using isotonic regression to convert raw predictions to meaningful confidence scores

### Requirement 4: ML Model Inference Service

**User Story:** As a backend developer, I want a reliable inference service for the trained ML model, so that the API can generate authenticity scores and red flag warnings.

#### Acceptance Criteria

1. THE System SHALL load the trained XGBoost model and feature scaler on service startup
2. WHEN receiving feature vectors, THE System SHALL generate authenticity scores from 0-100 based on model probability predictions
3. WHEN calculating confidence, THE System SHALL use prediction margin formula: abs(probability - 0.5) × 200
4. THE System SHALL detect red flags using rule-based thresholds for suspicious patterns including timing anomalies, trivial commits, fork inflation, and engagement issues
5. WHEN red flags are detected, THE System SHALL assign severity levels of LOW, MEDIUM, or HIGH based on threshold violations
6. THE System SHALL respond to inference requests within 100 milliseconds for optimal user experience

### Requirement 5: Backend API Development

**User Story:** As a frontend developer, I want a robust REST API for profile analysis, so that the web application can request and receive authenticity assessments.

#### Acceptance Criteria

1. THE System SHALL provide a POST /api/analyze endpoint that accepts GitHub usernames and returns comprehensive analysis results
2. WHEN receiving analysis requests, THE System SHALL validate usernames to contain only alphanumeric characters and hyphens
3. WHEN processing requests, THE System SHALL check MongoDB cache for existing analyses less than 24 hours old
4. IF cache miss occurs, THEN THE System SHALL fetch fresh data from GitHub API, extract features, and invoke ML service
5. THE System SHALL implement rate limiting of 10 requests per minute per IP address to prevent abuse
6. WHEN errors occur, THE System SHALL return appropriate HTTP status codes with descriptive error messages for user not found (404), rate limit exceeded (429), and service unavailable (503)

### Requirement 6: Data Persistence and Caching

**User Story:** As a system administrator, I want efficient data storage and caching, so that the system can serve repeated requests quickly while managing storage costs.

#### Acceptance Criteria

1. THE System SHALL store analysis results in MongoDB with TTL indexing for automatic cleanup after 24 hours
2. WHEN saving profiles, THE System SHALL include raw GitHub data, extracted features, ML predictions, red flags, and chart data
3. THE System SHALL create unique indexes on username field to prevent duplicate analyses
4. WHEN cache hits occur, THE System SHALL return cached results without re-fetching from GitHub API or re-running ML inference
5. THE System SHALL maintain connection pooling with maximum 10 concurrent MongoDB connections for optimal performance

### Requirement 7: Frontend User Interface

**User Story:** As an end user, I want an intuitive web interface to analyze GitHub profiles, so that I can easily assess profile authenticity with visual feedback.

#### Acceptance Criteria

1. THE System SHALL provide a home page with prominent search functionality and example profiles for quick testing
2. WHEN users enter usernames, THE System SHALL validate input format and provide immediate feedback for invalid entries
3. WHEN analysis is in progress, THE System SHALL display loading states with progress indicators for fetching, analyzing, and calculating phases
4. THE System SHALL present results with a large radial score gauge showing color-coded authenticity levels: red (0-40), orange (40-70), green (70-100)
5. WHEN displaying results, THE System SHALL show key statistics including total commits, public repositories, followers, and original repository percentage
6. IF red flags are detected, THEN THE System SHALL list them with severity badges and descriptive explanations

### Requirement 8: Advanced Data Visualization

**User Story:** As a user analyzing profiles, I want detailed charts and visualizations, so that I can understand the behavioral patterns behind the authenticity assessment.

#### Acceptance Criteria

1. THE System SHALL provide tabbed navigation for commit patterns, code quality, and language breakdown visualizations
2. WHEN displaying commit patterns, THE System SHALL show line charts of commits over the last 12 months and bar charts of commits by day of week
3. WHEN showing code quality metrics, THE System SHALL present donut charts of commit size distribution and statistical cards for average commit size
4. THE System SHALL generate stacked bar charts for programming language breakdown with color-coded segments and percentage labels
5. WHEN generating chart data, THE System SHALL aggregate raw GitHub data into visualization-ready formats during the analysis phase

### Requirement 9: Error Handling and Monitoring

**User Story:** As a system administrator, I want comprehensive error handling and health monitoring, so that I can maintain system reliability and quickly identify issues.

#### Acceptance Criteria

1. THE System SHALL implement global error handling middleware that catches unhandled exceptions and returns sanitized error responses
2. WHEN GitHub users are not found, THE System SHALL return 404 errors with clear messaging
3. WHEN GitHub API rate limits are exceeded, THE System SHALL return 429 errors with retry-after timing information
4. THE System SHALL provide health check endpoints that verify database connectivity and ML service availability
5. WHEN system components fail, THE System SHALL log detailed error information while preventing sensitive data leakage to end users

### Requirement 10: Deployment and Infrastructure

**User Story:** As a DevOps engineer, I want containerized deployment with cloud hosting, so that the system can scale reliably while minimizing infrastructure costs.

#### Acceptance Criteria

1. THE System SHALL be containerized using Docker with separate containers for backend API, ML inference service, and frontend application
2. WHEN deploying to production, THE System SHALL use Railway for backend hosting and Vercel for frontend hosting to leverage free tier offerings
3. THE System SHALL connect to MongoDB Atlas M0 cluster (512MB free tier) for data persistence
4. THE System SHALL implement automated CI/CD pipeline using GitHub Actions for testing and deployment
5. WHEN health checks fail, THE System SHALL provide detailed status information for debugging and monitoring purposes