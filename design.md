# Design Document: Rural Supply Chain and Price Intelligence System

## Overview

The Rural Supply Chain and Price Intelligence System is a cloud-based AI platform that provides farmers with actionable market intelligence through three core capabilities: mandi price prediction, crop demand forecasting, and optimal market recommendations. The system leverages machine learning models trained on historical market data, weather patterns, and agricultural statistics to generate predictions and recommendations.

The architecture follows a microservices pattern with clear separation between data ingestion, ML model serving, business logic, and presentation layers. This design enables independent scaling, easier maintenance, and flexibility to update individual components without affecting the entire system.

### Key Design Principles

1. **Mobile-First**: Optimized for smartphone users with limited bandwidth
2. **Offline-Capable**: Critical information cached locally for offline access
3. **Scalable**: Horizontal scaling to support growing user base
4. **Resilient**: Graceful degradation when external data sources are unavailable
5. **Multilingual**: Support for 10 Indian languages with easy extensibility
6. **Privacy-Focused**: Minimal data collection with strong encryption

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Web App    │  │ Android App  │  │   iOS App    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  Authentication │ Rate Limiting │ Request Routing      │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Application Services Layer                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Price      │  │   Demand     │  │   Market     │          │
│  │  Prediction  │  │  Forecasting │  │Recommendation│          │
│  │   Service    │  │   Service    │  │   Service    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    Alert     │  │     User     │  │    Data      │          │
│  │   Service    │  │   Service    │  │  Integration │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ML Model Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    Price     │  │   Demand     │  │ Optimization │          │
│  │    Model     │  │    Model     │  │    Model     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘

                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Data Layer                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Time-Series │  │  Relational  │  │    Cache     │          │
│  │   Database   │  │   Database   │  │    (Redis)   │          │
│  │  (TimescaleDB)│ │  (PostgreSQL)│  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   External Data Sources                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Agmarknet   │  │     IMD      │  │  Ministry of │          │
│  │   (Prices)   │  │  (Weather)   │  │ Agriculture  │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

**Frontend:**
- Web: React with TypeScript, Progressive Web App (PWA) capabilities
- Mobile: React Native for cross-platform iOS and Android
- State Management: Redux Toolkit
- Offline Storage: IndexedDB (web), AsyncStorage (mobile)

**Backend:**
- API Gateway: Kong or AWS API Gateway
- Services: Python (FastAPI) for ML services, Node.js (Express) for business logic
- Message Queue: RabbitMQ for async processing
- Cache: Redis for session management and frequently accessed data

**ML/AI:**
- Framework: TensorFlow/Keras for deep learning models
- Time Series: Prophet, ARIMA, LSTM networks
- Model Serving: TensorFlow Serving or TorchServe
- Feature Store: Feast for feature management

**Data:**
- Time-Series DB: TimescaleDB (PostgreSQL extension) for price and weather data
- Relational DB: PostgreSQL for user data, market info, configurations
- Object Storage: S3-compatible storage for model artifacts and backups
- Data Pipeline: Apache Airflow for ETL orchestration

**Infrastructure:**
- Container Orchestration: Kubernetes
- Cloud Provider: AWS, Azure, or GCP (cloud-agnostic design)
- Monitoring: Prometheus + Grafana
- Logging: ELK Stack (Elasticsearch, Logstash, Kibana)

## Components and Interfaces

### 1. Price Prediction Service

**Responsibility:** Generate price predictions for crop-mandi combinations using ML models.

**Key Operations:**
- `predictPrice(cropId, mandiId, predictionWindow)`: Returns price predictions with confidence scores
- `getPriceHistory(cropId, mandiId, startDate, endDate)`: Returns historical price data
- `comparePrices(cropId, mandiIds[], date)`: Compares prices across multiple mandis

**Interfaces:**
```typescript
interface PricePredictionRequest {
  cropId: string;
  mandiId: string;
  predictionWindows: number[]; // [7, 15, 30] days
  userId: string;
}

interface PricePredictionResponse {
  cropId: string;
  mandiId: string;
  predictions: PricePrediction[];
  metadata: PredictionMetadata;
}

interface PricePrediction {
  date: string;
  minPrice: number;
  maxPrice: number;
  mostLikelyPrice: number;
  confidenceScore: number; // 0-100
  predictionWindow: number; // days ahead
}

interface PredictionMetadata {
  modelVersion: string;
  generatedAt: string;
  dataFreshness: string;
  factorsConsidered: string[];
}
```

**ML Model Architecture:**
- Ensemble approach combining LSTM, Prophet, and XGBoost
- Input features: historical prices (3 years), weather data, seasonal patterns, festival calendar, production statistics
- Output: Price distribution (min, max, most likely) with confidence intervals
- Training: Weekly retraining on latest data
- Validation: Walk-forward validation with 80-20 train-test split

### 2. Demand Forecasting Service

**Responsibility:** Forecast future demand trends for crops at regional and national levels.

**Key Operations:**
- `forecastDemand(cropId, regionId, forecastWindow)`: Returns demand forecasts
- `getDemandTrends(cropId, regionId, historicalPeriod)`: Returns historical demand patterns
- `identifyDemandDrivers(cropId, regionId)`: Returns factors influencing demand

**Interfaces:**
```typescript
interface DemandForecastRequest {
  cropId: string;
  regionId: string;
  forecastWindows: number[]; // [30, 60, 90] days
  userId: string;
}

interface DemandForecastResponse {
  cropId: string;
  regionId: string;
  forecasts: DemandForecast[];
  metadata: ForecastMetadata;
}

interface DemandForecast {
  date: string;
  demandLevel: 'Low' | 'Medium' | 'High' | 'Very High';
  demandIndex: number; // 0-100 normalized score
  confidenceScore: number;
  forecastWindow: number;
}

interface ForecastMetadata {
  modelVersion: string;
  generatedAt: string;
  seasonalFactors: string[];
  upcomingEvents: string[]; // festivals, holidays
}
```

**ML Model Architecture:**
- Time series forecasting with seasonal decomposition
- Input features: historical consumption, population, festival calendar, weather, export-import data
- Output: Demand level classification with confidence scores
- Training: Monthly retraining
- Validation: MAPE < 15% on test set

### 3. Market Recommendation Service

**Responsibility:** Recommend optimal markets for selling produce based on prices, costs, and logistics.

**Key Operations:**
- `recommendMarkets(cropId, quantity, userLocation, preferences)`: Returns ranked market recommendations
- `calculateNetProfit(cropId, quantity, sourceLocation, targetMandi)`: Calculates expected profit
- `getTransportationCost(sourceLocation, targetLocation, quantity, vehicleType)`: Estimates transport costs

**Interfaces:**
```typescript
interface MarketRecommendationRequest {
  cropId: string;
  quantity: number; // in kg
  userLocation: GeoLocation;
  preferences: UserPreferences;
  userId: string;
}

interface MarketRecommendationResponse {
  recommendations: MarketRecommendation[];
  generatedAt: string;
  validUntil: string;
}

interface MarketRecommendation {
  mandiId: string;
  mandiName: string;
  location: GeoLocation;
  distance: number; // km
  currentPrice: number;
  predictedPrice: number;
  transportationCost: number;
  marketFees: number;
  estimatedSpoilage: number;
  netProfit: number;
  profitMargin: number; // percentage
  rank: number;
  warnings: string[]; // congestion, capacity issues
  confidenceScore: number;
}

interface UserPreferences {
  maxDistance: number; // km
  preferredVehicleType: string;
  riskTolerance: 'Low' | 'Medium' | 'High';
}

interface GeoLocation {
  latitude: number;
  longitude: number;
  address?: string;
}
```

**Optimization Algorithm:**
- Multi-objective optimization: maximize net profit, minimize distance, minimize risk
- Constraints: vehicle capacity, market capacity, spoilage rates
- Scoring function: weighted combination of profit (60%), distance (25%), reliability (15%)
- Real-time updates based on current market conditions

### 4. Alert Service

**Responsibility:** Monitor market conditions and send notifications to users.

**Key Operations:**
- `createAlert(userId, alertConfig)`: Creates a new price alert
- `checkAlerts()`: Periodic job to evaluate alert conditions
- `sendNotification(userId, alertType, message)`: Sends notifications via configured channels

**Interfaces:**
```typescript
interface AlertConfig {
  alertId?: string;
  userId: string;
  cropId: string;
  mandiId: string;
  alertType: 'TARGET_PRICE' | 'PRICE_INCREASE' | 'PRICE_DECREASE' | 'DEMAND_SPIKE';
  targetPrice?: number;
  thresholdPercentage?: number;
  notificationChannels: ('SMS' | 'PUSH' | 'EMAIL')[];
  isActive: boolean;
}

interface AlertNotification {
  alertId: string;
  userId: string;
  message: string;
  timestamp: string;
  data: {
    cropId: string;
    mandiId: string;
    currentPrice: number;
    previousPrice?: number;
    changePercentage?: number;
  };
}
```

**Implementation:**
- Background job runs every 30 minutes during market hours
- Uses Redis for fast alert condition checking
- Notification delivery via Twilio (SMS), Firebase Cloud Messaging (Push)
- Retry logic with exponential backoff for failed deliveries

### 5. User Service

**Responsibility:** Manage user accounts, profiles, preferences, and authentication.

**Key Operations:**
- `registerUser(userDetails)`: Creates new user account
- `authenticateUser(credentials)`: Validates credentials and issues JWT token
- `updateProfile(userId, profileData)`: Updates user profile
- `getUserPreferences(userId)`: Retrieves user preferences and saved items

**Interfaces:**
```typescript
interface UserProfile {
  userId: string;
  phoneNumber: string;
  name: string;
  location: GeoLocation;
  preferredLanguage: string;
  favoriteCrops: string[]; // max 5
  preferredMandis: string[]; // max 3
  farmSize: number; // hectares
  createdAt: string;
  lastLoginAt: string;
}

interface AuthenticationRequest {
  phoneNumber: string;
  password?: string;
  otp?: string;
}

interface AuthenticationResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  user: UserProfile;
}
```

**Security:**
- Password hashing: bcrypt with salt rounds = 12
- JWT tokens: RS256 algorithm, 1-hour expiry for access tokens, 7-day expiry for refresh tokens
- OTP-based authentication for users without passwords
- Rate limiting: 5 failed login attempts trigger 15-minute lockout

### 6. Data Integration Service

**Responsibility:** Fetch, validate, and integrate data from external sources.

**Key Operations:**
- `fetchAgmarknetData()`: Retrieves daily price data from Agmarknet
- `fetchWeatherData()`: Retrieves weather data from IMD
- `fetchProductionData()`: Retrieves crop production statistics
- `validateAndStore(data, source)`: Validates and stores integrated data

**Interfaces:**
```typescript
interface DataIntegrationJob {
  jobId: string;
  source: 'AGMARKNET' | 'IMD' | 'AGRICULTURE_MINISTRY';
  status: 'PENDING' | 'RUNNING' | 'SUCCESS' | 'FAILED';
  startTime: string;
  endTime?: string;
  recordsProcessed: number;
  recordsFailed: number;
  errors: string[];
}

interface DataValidationResult {
  isValid: boolean;
  errors: ValidationError[];
  warnings: ValidationWarning[];
}

interface ValidationError {
  field: string;
  value: any;
  reason: string;
  severity: 'ERROR' | 'WARNING';
}
```

**Data Pipeline:**
- Scheduled jobs via Apache Airflow
- Agmarknet: Daily at 7:00 AM IST
- Weather: Every 6 hours
- Production stats: Monthly on 1st day
- Validation: Statistical outlier detection, completeness checks, format validation
- Error handling: Retry 3 times, fallback to previous valid data, alert administrators

## Data Models

### Price Data Model

```typescript
interface PriceRecord {
  priceId: string;
  cropId: string;
  mandiId: string;
  date: string; // ISO 8601
  minPrice: number;
  maxPrice: number;
  modalPrice: number; // most common price
  arrivals: number; // quantity arrived in quintals
  source: string; // 'AGMARKNET', 'MANUAL'
  dataQuality: 'HIGH' | 'MEDIUM' | 'LOW';
  createdAt: string;
  updatedAt: string;
}
```

**Storage:** TimescaleDB with hypertables partitioned by date
**Indexes:** Composite index on (cropId, mandiId, date)
**Retention:** 5 years, then archived to cold storage

### Crop Data Model

```typescript
interface Crop {
  cropId: string;
  name: string;
  localNames: Record<string, string>; // language code -> local name
  category: 'CEREAL' | 'PULSE' | 'VEGETABLE' | 'FRUIT' | 'SPICE' | 'OTHER';
  perishability: 'HIGH' | 'MEDIUM' | 'LOW';
  typicalShelfLife: number; // days
  seasonality: SeasonInfo[];
  averageYield: number; // kg per hectare
}

interface SeasonInfo {
  season: 'KHARIF' | 'RABI' | 'ZAID';
  sowingMonths: number[];
  harvestMonths: number[];
  regions: string[];
}
```

### Mandi Data Model

```typescript
interface Mandi {
  mandiId: string;
  name: string;
  location: GeoLocation;
  state: string;
  district: string;
  facilities: string[]; // 'COLD_STORAGE', 'GRADING', 'PACKAGING'
  operatingHours: string;
  marketFeePercentage: number;
  averageDailyVolume: number; // quintals
  supportedCrops: string[];
  congestionLevel: 'LOW' | 'MEDIUM' | 'HIGH';
  lastUpdated: string;
}
```

### Weather Data Model

```typescript
interface WeatherRecord {
  weatherId: string;
  location: GeoLocation;
  date: string;
  temperature: {
    min: number;
    max: number;
    avg: number;
  };
  rainfall: number; // mm
  humidity: number; // percentage
  windSpeed: number; // km/h
  conditions: string; // 'CLEAR', 'RAIN', 'STORM', etc.
  source: string;
}

interface WeatherForecast {
  forecastId: string;
  location: GeoLocation;
  forecastDate: string;
  generatedAt: string;
  predictions: WeatherRecord[];
  confidenceScore: number;
}
```

### User Activity Model

```typescript
interface UserActivity {
  activityId: string;
  userId: string;
  activityType: 'PREDICTION' | 'RECOMMENDATION' | 'ALERT' | 'COMPARISON';
  timestamp: string;
  details: Record<string, any>;
  deviceInfo: {
    platform: string;
    appVersion: string;
    osVersion: string;
  };
}
```

**Storage:** PostgreSQL with partitioning by month
**Retention:** 1 year for detailed logs, aggregated metrics retained indefinitely
