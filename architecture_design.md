```markdown
# Technical Architecture Document: Stock Price Tracker

## 1. System Architecture Overview

### 1.1 Architecture Pattern
- **Frontend**: Single Page Application (SPA) with component-based architecture
- **Backend**: API Gateway + Microservices
- **Data Flow**: Real-time WebSocket connections for live data, REST APIs for historical data
- **Deployment**: Containerized services on cloud platform

### 1.2 High-Level Architecture Diagram
```
┌─────────────────────────────────────────────────────────────┐
│                    Client Devices                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   Desktop   │  │   Tablet    │  │   Mobile    │         │
│  │   Browser   │  │   Browser   │  │   Browser   │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                 │                 │                │
└─────────┼─────────────────┼─────────────────┼────────────────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                    HTTPS (TLS 1.3)
                            │
                 ┌──────────▼──────────┐
                 │   Cloudflare CDN    │
                 │  (Static Assets)    │
                 └──────────┬──────────┘
                            │
                 ┌──────────▼──────────┐
                 │   API Gateway        │
                 │  (NGINX + Kong)     │
                 └──────────┬──────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼──────┐   ┌───────▼──────┐   ┌───────▼──────┐
│   WebSocket  │   │   REST API   │   │   Auth       │
│   Service    │   │   Service    │   │   Service    │
│   (Node.js)  │   │   (Python)   │   │   (Future)   │
└───────┬──────┘   └───────┬──────┘   └──────────────┘
        │                   │
        │         ┌─────────▼─────────┐
        │         │   Data Processing │
        │         │   Service         │
        │         │   (Python)        │
        │         └─────────┬─────────┘
        │                   │
        │         ┌─────────▼─────────┐
        │         │   Cache Layer     │
        │         │   (Redis)         │
        │         └─────────┬─────────┘
        │                   │
        └───────────────────┤
                            │
                 ┌──────────▼──────────┐
                 │   External APIs     │
                 │  • Alpha Vantage    │
                 │  • Yahoo Finance    │
                 │  • B3 Feed          │
                 └─────────────────────┘
```

## 2. Technology Stack

### 2.1 Frontend
- **Framework**: React 18 with TypeScript
- **State Management**: Zustand (lightweight alternative to Redux)
- **UI Library**: Chakra UI (accessible, mobile-first components)
- **Charts**: Recharts (lightweight, responsive charting)
- **Real-time**: Socket.IO client
- **Build Tool**: Vite
- **Testing**: Jest + React Testing Library + Cypress (E2E)

### 2.2 Backend Services
- **API Gateway**: Kong + NGINX
- **WebSocket Service**: Node.js + Socket.IO
- **REST API Service**: Python FastAPI
- **Data Processing**: Python + Pandas + NumPy
- **Task Queue**: Celery + Redis (for scheduled data fetching)
- **Authentication**: JWT (future implementation)

### 2.3 Data Layer
- **Cache**: Redis 7.x (for real-time data and session storage)
- **Time-series Data**: InfluxDB 2.x (for historical price data)
- **Relational Database**: PostgreSQL 15 (for user data, future use)
- **Object Storage**: AWS S3 / Cloudflare R2 (for static assets)

### 2.4 Infrastructure & DevOps
- **Containerization**: Docker + Docker Compose
- **Orchestration**: Kubernetes (production) / Docker Swarm (staging)
- **Cloud Platform**: AWS (EC2, RDS, ElastiCache) or DigitalOcean
- **CI/CD**: GitHub Actions
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack (Elasticsearch, Logstash, Kibana)
- **Error Tracking**: Sentry

## 3. Data Models

### 3.1 Stock Data Model (Redis/InfluxDB)
```typescript
interface StockQuote {
  symbol: string;           // e.g., "PETR4.SA"
  name: string;            // "Petrobras PN"
  price: number;           // Current price in BRL
  change: number;          // Absolute change
  changePercent: number;   // Percentage change
  volume: number;          // Trading volume
  timestamp: ISOString;    // Last update timestamp
  
  // Additional details
  open: number;
  high: number;
  low: number;
  previousClose: number;
  marketCap?: number;
  peRatio?: number;
  sector?: string;
}

interface StockHistoricalData {
  symbol: string;
  timeframe: '1D' | '1W' | '1M';
  data: Array<{
    timestamp: ISOString;
    open: number;
    high: number;
    low: number;
    close: number;
    volume: number;
  }>;
}
```

### 3.2 User Data Model (PostgreSQL - Future)
```sql
-- For MVP, using localStorage only
-- Future schema for user accounts:

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  last_login TIMESTAMP
);

CREATE TABLE watchlists (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE watchlist_stocks (
  watchlist_id UUID REFERENCES watchlists(id) ON DELETE CASCADE,
  stock_symbol VARCHAR(20) NOT NULL,
  added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (watchlist_id, stock_symbol)
);

CREATE TABLE price_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  stock_symbol VARCHAR(20) NOT NULL,
  threshold_price DECIMAL(10,2) NOT NULL,
  condition VARCHAR(10) CHECK (condition IN ('ABOVE', 'BELOW')),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  triggered_at TIMESTAMP
);
```

### 3.3 Cache Structure (Redis)
```
# Real-time stock data (5-second TTL)
stock:quote:{symbol} -> JSON string of StockQuote
stock:quotes:latest -> Hash of all 10 stocks

# Historical data (longer TTL)
stock:history:{symbol}:{timeframe} -> JSON string of historical data

# Session data
session:{sessionId}:watchlist -> Set of stock symbols
session:{sessionId}:alerts -> Hash of price alerts

# Market summary
market:summary -> JSON string of Ibovespa data
market:top_gainers -> Sorted set of symbols by changePercent
market:top_losers -> Sorted set of symbols by changePercent
```

## 4. API Design

### 4.1 REST API Endpoints (FastAPI)

#### Stock Data
```
GET /api/v1/stocks
- Returns list of top 10 Ibovespa stocks
- Query params: ?fields=basic|full

GET /api/v1/stocks/{symbol}
- Returns detailed stock information

GET /api/v1/stocks/{symbol}/history
- Returns historical data
- Query params: ?timeframe=1d|1w|1m

GET /api/v1/market/summary
- Returns Ibovespa index data and market overview
```

#### User Data (LocalStorage-based for MVP)
```
POST /api/v1/watchlist (Future)
- Add stock to watchlist

DELETE /api/v1/watchlist/{symbol} (Future)
- Remove stock from watchlist

POST /api/v1/alerts (Future)
- Create price alert

GET /api/v1/alerts (Future)
- List user's alerts
```

### 4.2 WebSocket Events (Socket.IO)

#### Client → Server
```javascript
{
  // Subscribe to stock updates
  event: 'subscribe',
  data: {
    symbols: ['PETR4.SA', 'VALE3.SA'],
    interval: 5000 // ms
  }
}

{
  // Unsubscribe from stock updates
  event: 'unsubscribe',
  data: {
    symbols: ['PETR4.SA']
  }
}

{
  // Request historical data
  event: 'history',
  data: {
    symbol: 'PETR4.SA',
    timeframe: '1d'
  }
}
```

#### Server → Client
```javascript
{
  // Real-time stock updates
  event: 'stock_update',
  data: {
    symbol: 'PETR4.SA',
    quote: StockQuote,
    timestamp: '2024-01-15T10:30:00Z'
  }
}

{
  // Historical data response
  event: 'history_data',
  data: StockHistoricalData
}

{
  // Market summary update
  event: 'market_update',
  data: MarketSummary
}

{
  // Alert notification
  event: 'alert_triggered',
  data: {
    symbol: 'PETR4.SA',
    threshold: 35.50,
    currentPrice: 35.75,
    condition: 'ABOVE'
  }
}
```

### 4.3 External API Integration
```python
# Data fetching service architecture
class DataFetcher:
    def __init__(self):
        self.sources = {
            'primary': AlphaVantageAPI(api_key=env.ALPHA_VANTAGE_KEY),
            'fallback': YahooFinanceAPI(),
            'backup': B3Scraper()  # If licensed
        }
    
    async def fetch_stock_data(self, symbols: List[str]) -> Dict[str, StockQuote]:
        # Try primary source, fall back if fails
        # Implement circuit breaker pattern
        # Cache results in Redis with 5-second TTL
```

## 5. File Structure

### 5.1 Frontend (React + TypeScript)
```
src/
├── components/
│   ├── common/
│   │   ├── Header/
│   │   ├── Footer/
│   │   ├── LoadingSpinner/
│   │   └── ErrorBoundary/
│   ├── stocks/
│   │   ├── StockCard/
│   │   ├── StockTable/
│   │   ├── StockChart/
│   │   └── StockDetails/
│   ├── market/
│   │   ├── MarketSummary/
│   │   ├── GainersLosers/
│   │   └── SectorOverview/
│   └── user/
│       ├── Watchlist/
│       ├── AlertModal/
│       └── Preferences/
├── hooks/
│   ├── useStocks.ts
│   ├── useWebSocket.ts
│   ├── useWatchlist.ts
│   └── useAlerts.ts
├── services/
│   ├── api/
│   │   ├── stockAPI.ts
│   │   ├── marketAPI.ts
│   │   └── websocket.ts
│   ├── storage/
│   │   ├── localStorage.ts
│   │   └── sessionStorage.ts
│   └── notifications/
│       └── browserNotifications.ts
├── stores/
│   ├── stockStore.ts
│   ├── marketStore.ts
│   ├── userStore.ts
│   └── uiStore.ts
├── utils/
│   ├── formatters.ts
│   ├── validators.ts
│   └── constants.ts
├── types/
│   ├── stock.types.ts
│   ├── market.types.ts
│   └── user.types.ts
├── styles/
│   ├── theme.ts
│   ├── global.css
│   └── responsive.css
└── App.tsx
```

### 5.2 Backend Services
```
backend/
├── api-gateway/
│   ├── kong.yml
│   └── nginx.conf
├── websocket-service/
│   ├── src/
│   │   ├── handlers/
│   │   │   ├── stockHandler.ts
│   │   │   └── marketHandler.ts
│   │   ├── services/
│   │   │   ├── dataService.ts
│   │   │   └── cacheService.ts
│   │   └── server.ts
│   ├── Dockerfile
│   └── package.json
├── rest-api/
│   ├── src/
│   │   ├── api/
│   │   │   ├── v1/
│   │   │   │   ├── stocks.py
│   │   │   │   ├── market.py
│   │   │   │   └── users.py
│   │   │   └── dependencies.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   └── security.py
│   │   ├── models/
│   │   │   ├── stock.py
│   │   │   └── user.py
│   │   ├── services/
│   │   │   ├── data_fetcher.py
│   │   │   └── cache_manager.py
│   │   └── main.py
│   ├── Dockerfile
│   └── requirements.txt
├── data-processor/
│   ├── src/
│   │   ├── tasks/
│   │   │   ├── fetch_stocks.py
│   │   │   └── update_cache.py
│   │   └── scheduler.py
│   ├── Dockerfile
│   └── requirements.txt
└── docker-compose.yml
```

## 6. Performance Optimization

### 6.1 Frontend Optimizations
- **Code Splitting**: React.lazy() for route-based splitting
- **Image Optimization**: WebP format with responsive images
- **Bundle Optimization**: Tree-shaking, minification, compression
- **Caching Strategy**: Service Worker for offline capability
- **Virtualization**: React-window for large lists (future)

### 6.2 Backend Optimizations
- **Connection Pooling**: Database and Redis connection pools
- **Query Optimization**: Indexed queries, prepared statements
- **Compression**: Gzip/Brotli for API responses
- **CDN**: Static assets served via Cloudflare
- **Load Balancing**: Horizontal scaling of stateless services

### 6.3 Real-time Data Pipeline
```
External API → Data Processor → Redis Cache → WebSocket Service → Clients
     ↓              ↓               ↓
   Fallback     Validation      Pub/Sub
     ↓              ↓               ↓
   Backup       Enrichment     Broadcasting
```

## 7. Security Implementation

### 7.1 Network Security
- HTTPS enforcement with HSTS headers
- CORS configuration for API endpoints
- Rate limiting (100 requests/minute per IP)
- WebSocket connection validation

### 7.2 Data Security
- Input validation and sanitization
- SQL injection prevention (parameterized queries)
- XSS protection (Content Security Policy)
- LocalStorage encryption for sensitive data

### 7.3 Compliance
- LGPD compliance for Brazilian users
- Data retention policies (30 days for historical data)
- Privacy policy and cookie consent

## 8. Monitoring & Alerting

### 8.1 Application Metrics
- **Performance**: Response times, error rates, uptime
- **Business**: Active users, session duration, feature usage
- **System**: CPU, memory, disk I/O, network traffic

### 8.2 Alert Rules
- API latency > 500ms for 5 minutes
- Error rate > 1% for 10 minutes
- WebSocket disconnect rate > 5%
- Data freshness > 10 seconds during market hours

### 8.3 Logging Strategy
- Structured logging (JSON format)
- Correlation IDs for request tracing
- Log levels: DEBUG, INFO, WARN, ERROR
- Centralized log aggregation

## 9. Deployment Strategy

### 9.1 Environment Configuration
```
environments/
├── development/
│   ├── .env.development
│   └── docker-compose.dev.yml
├── staging/
│   ├── .env.staging
│   └── kubernetes/
└── production/
    ├── .env.production
    └── kubernetes/
```

### 9.2 CI/CD Pipeline
```yaml
# GitHub Actions workflow
name: Deploy
on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - Run unit tests
      - Run integration tests
      - Run E2E tests
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - Build Docker images
      - Push to container registry
      - Scan for vulnerabilities
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - Deploy to Kubernetes
      - Run smoke tests
      - Update monitoring dashboards
```

## 10. Scalability Considerations

### 10.1 Horizontal Scaling
- Stateless API services auto-scale based on CPU/RAM
- Redis cluster for cache scaling
- PostgreSQL read replicas for future user data
- WebSocket service with sticky sessions

### 10.2 Database Scaling
- **Redis**: Cluster mode with sharding
- **InfluxDB**: Single node for MVP, cluster for >100K users
- **PostgreSQL**: Read replicas, connection pooling

### 10.3 Cost Optimization
