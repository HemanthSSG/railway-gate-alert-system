# Railway Gate Train Prediction & Alert System - Architecture

## 1. Project Overview

This is a production-grade, full-stack real-time system that:
- Detects user GPS location
- Identifies the nearest railway level crossing/gate
- Tracks live train positions
- Predicts train arrival at the railway gate
- Sends real-time alerts and notifications

---

## 2. Technical Architecture

### 2.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend (React)                        │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐          │
│  │ Map Display  │  │ Notification│  │ User Location│          │
│  │  (Leaflet)   │  │   System    │  │  Tracking    │          │
│  └──────┬───────┘  └──────┬──────┘  └──────┬───────┘          │
└─────────┼────────────────┼──────────────────┼──────────────────┘
          │                │                  │
          └────────────────┼──────────────────┘
                          │
              WebSocket / Server-Sent Events
                          │
┌─────────────────────────▼──────────────────────────────────────┐
│                    Backend (FastAPI)                           │
│  ┌───────────────┐  ┌────────────────┐  ┌──────────────────┐ │
│  │ REST API      │  │ WebSocket      │  │ Real-time Event  │ │
│  │ Endpoints     │  │ Handler        │  │ Manager          │ │
│  └───────┬───────┘  └────────┬───────┘  └────────┬─────────┘ │
│          │                   │                   │            │
│  ┌───────▼───────────────────▼───────────────────▼──────────┐ │
│  │           Prediction & Processing Engine                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │ Nearest Gate │  │ Train Approach│  │ ETA Engine   │  │ │
│  │  │  Calculator  │  │  Detector     │  │              │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │ │
│  └──────────────────────────────────────────────────────────┘ │
│          │                                                     │
│  ┌───────▼─────────────────────────────────────────────────┐ │
│  │         Train Data Service Layer                        │ │
│  │  ┌──────────────────────────────────────────────────┐  │ │
│  │  │  Train Provider Interface (Abstract)             │  │ │
│  │  │  ├── Live API Provider (when available)          │  │ │
│  │  │  └── Mock Provider (for development)             │  │ │
│  │  └──────────────────────────────────────────────────┘  │ │
│  └────────────────────────┬─────────────────────────────────┘ │
└─────────────────────────┬──────────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼────┐    ┌──────▼──────┐   ┌─────▼──────┐
   │PostgreSQL│    │External API │   │Mock Data   │
   │  +PostGIS│    │(Railway)    │   │ Provider   │
   └──────────┘    └─────────────┘   └────────────┘
```

### 2.2 Data Flow

```
User Location (GPS)
    ↓
[Nearest Gate Calculator] → Haversine Distance
    ↓
Railway Gate Selected
    ↓
[Fetch Approaching Trains] → Train Data Service
    ↓
[Train Direction Detector] → Analyze movement toward gate
    ↓
[ETA Prediction Engine] → Calculate arrival time
    ↓
[Real-time WebSocket] → Push to Frontend
    ↓
[Notification System] → Alert user
    ↓
Frontend: Update Map & Display ETA
```

---

## 3. Technology Stack

### Frontend
- **React.js** (18.x)
- **TypeScript** (for type safety)
- **Leaflet** (interactive maps)
- **Axios** (HTTP client)
- **Socket.IO Client** (WebSocket)
- **React Query** (data fetching)
- **Tailwind CSS** (styling)

### Backend
- **Python 3.11+**
- **FastAPI** (async web framework)
- **SQLAlchemy** (ORM)
- **Pydantic** (data validation)
- **PostgreSQL** (database)
- **PostGIS** (geospatial queries)
- **python-socketio** (WebSocket)
- **APScheduler** (scheduled tasks)

### Infrastructure
- **Docker & Docker Compose** (containerization)
- **PostgreSQL 14+** (database)
- **Redis** (optional: caching, real-time message queue)

### External Services
- **OpenStreetMap / Leaflet** (map tiles)
- **Railway API** (to be verified and integrated)

---

## 4. Database Schema

### Tables Structure

#### `railway_stations`
```sql
CREATE TABLE railway_stations (
    id SERIAL PRIMARY KEY,
    station_code VARCHAR(20) UNIQUE NOT NULL,
    station_name VARCHAR(255) NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    railway_line VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_station_location ON railway_stations USING GIST (
    ST_Point(longitude, latitude)
);
```

#### `railway_gates`
```sql
CREATE TABLE railway_gates (
    id SERIAL PRIMARY KEY,
    gate_name VARCHAR(255) NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    station_id INTEGER REFERENCES railway_stations(id),
    railway_line VARCHAR(100) NOT NULL,
    station_distance_km DECIMAL(8, 3),
    estimated_travel_time_seconds INTEGER,
    gate_direction VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_gate_location ON railway_gates USING GIST (
    ST_Point(longitude, latitude)
);
```

#### `trains`
```sql
CREATE TABLE trains (
    id SERIAL PRIMARY KEY,
    train_number VARCHAR(20) UNIQUE NOT NULL,
    train_name VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'RUNNING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `train_positions`
```sql
CREATE TABLE train_positions (
    id SERIAL PRIMARY KEY,
    train_id INTEGER REFERENCES trains(id),
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    speed_kmph DECIMAL(8, 2),
    direction VARCHAR(50),
    timestamp TIMESTAMP NOT NULL,
    source VARCHAR(50) DEFAULT 'API',
    data_freshness_score DECIMAL(3, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_train_position_location ON train_positions USING GIST (
    ST_Point(longitude, latitude)
);
CREATE INDEX idx_train_position_timestamp ON train_positions(timestamp DESC);
```

#### `train_predictions`
```sql
CREATE TABLE train_predictions (
    id SERIAL PRIMARY KEY,
    train_id INTEGER REFERENCES trains(id),
    gate_id INTEGER REFERENCES railway_gates(id),
    eta_seconds INTEGER,
    eta_timestamp TIMESTAMP,
    current_distance_km DECIMAL(8, 3),
    approaching BOOLEAN DEFAULT FALSE,
    direction_to_gate VARCHAR(50),
    confidence_score DECIMAL(3, 2),
    calculated_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_prediction_gate ON train_predictions(gate_id);
CREATE INDEX idx_prediction_train ON train_predictions(train_id);
```

#### `station_gate_calibration`
```sql
CREATE TABLE station_gate_calibration (
    id SERIAL PRIMARY KEY,
    station_id INTEGER REFERENCES railway_stations(id),
    gate_id INTEGER REFERENCES railway_gates(id),
    distance_km DECIMAL(8, 3),
    average_travel_time_seconds INTEGER,
    sample_count INTEGER DEFAULT 1,
    last_calibrated TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_station_gate_pair ON station_gate_calibration(station_id, gate_id);
```

---

## 5. Key Algorithms

### 5.1 Nearest Gate Calculator
```
Input: User Latitude, User Longitude
Output: Nearest Railway Gate

Algorithm:
1. Query all railway gates from database
2. Calculate Haversine distance from user to each gate
3. Filter gates within reasonable radius (e.g., 5 km)
4. Return gate with minimum distance
5. Also return top 3 alternatives

Haversine Formula:
a = sin²(Δφ/2) + cos φ1 ⋅ cos φ2 ⋅ sin²(Δλ/2)
c = 2 ⋅ atan2( √a, √(1−a) )
d = R ⋅ c

where:
- φ is latitude, λ is longitude, R is earth's radius (6371 km)
```

### 5.2 Train Direction Detector
```
Input: Previous Position, Current Position, Gate Position
Output: Is train approaching gate?

Algorithm:
1. Calculate vector from previous position to current position
2. Calculate vector from current position to gate
3. Calculate dot product of vectors
4. If dot product < 0: train moving toward gate (approaching)
5. If dot product > 0: train moving away from gate
6. Calculate cross product for perpendicular distance
7. Consider railway line geometry (if available)

Decision Tree:
if train_on_gate_railway_line:
    if distance_to_gate < last_distance_to_gate:
        return APPROACHING
    else:
        return MOVING_AWAY
else:
    return NOT_ON_TRACK
```

### 5.3 ETA Prediction Engine
```
Input: Train Position, Gate Position, Train Speed, Historical Data
Output: ETA in seconds

Algorithm - Primary (Speed-based):
if train_speed > 0 and recent_speed_data:
    distance_km = haversine(train_pos, gate_pos)
    eta_seconds = (distance_km / speed_kmph) * 3600
    confidence = HIGH
else:
    Algorithm - Secondary (Historical):
    eta_seconds = fetch_calibrated_travel_time(station, gate)
    confidence = MEDIUM

Stale Data Handling:
if (now - last_update) > STALE_THRESHOLD:
    confidence = REDUCED
    mark prediction as "potentially outdated"
    adjust eta_seconds with uncertainty margin (+30%)

Output Format:
{
    eta_seconds: int,
    eta_timestamp: ISO8601,
    confidence: ENUM[HIGH, MEDIUM, LOW],
    is_stale: bool,
    calculated_at: ISO8601
}
```

### 5.4 Waiting Time Calculation
```
Input: ETA to Gate, Historical Gate Clearance Data
Output: Estimated Waiting Time

If gate_clearance_data_available:
    waiting_time = gate_arrival_eta + gate_clearance_time
    confidence = MEDIUM
Else:
    waiting_time = null
    message = "Gate clearance data unavailable"
    confidence = NONE

Display to User:
"Train arrival: ~{eta} seconds"
"Gate clearance: Unknown (follow official signals)"
```

---

## 6. Real-Time Communication Strategy

### WebSocket Events

#### Frontend → Backend
```
CONNECT: /ws/gate/{gate_id}
    → Subscribe to real-time updates for specific gate

USER_LOCATION: { latitude, longitude, accuracy, timestamp }
    → Update user location for distance calculations

DISCONNECT:
    → Unsubscribe from gate updates
```

#### Backend → Frontend
```
GATE_STATUS: { gate_id, approaching_trains, nearest_train_eta }
    → General gate status update

TRAIN_APPROACHING: { train_number, eta_seconds, distance_km, confidence }
    → Alert when train is approaching

TRAIN_UPDATE: { train_number, latitude, longitude, speed, timestamp }
    → Regular train position updates

TRAIN_PASSED: { train_number, passed_at }
    → Notification after train passes gate

WARNING_LEVEL_CHANGE: { level, message, eta_seconds }
    → Warning state change (NO_TRAIN → DETECTED → APPROACHING → IMMEDIATE → PASSED)
```

### Update Frequency
- **Train Position Updates**: Every 30-60 seconds
- **ETA Recalculation**: Every 15-30 seconds (when train approaching)
- **User Location Updates**: Every 30-60 seconds
- **Prediction Updates**: Continuous (calculated server-side)

---

## 7. Error Handling Strategy

| Error Scenario | Handling |
|---|---|
| GPS unavailable | Display error message, suggest enabling location |
| GPS permission denied | Request permission or show static map |
| Poor GPS accuracy | Show accuracy indicator, allow manual refinement |
| API rate limit | Implement exponential backoff, queue requests |
| API timeout | Use cached data, mark as potentially stale |
| Invalid coordinates | Reject, request new location |
| No nearby gates | Display message, expand search radius |
| No approaching trains | Show "safe to cross" (if no trains) |
| Train data stale (>10 min) | Mark ETA as unreliable, show uncertainty |
| Database connection failure | Return cached response or error |
| WebSocket disconnect | Auto-reconnect with exponential backoff |

---

## 8. Security Measures

1. **API Key Management**
   - Store all keys in `.env` file (never in code)
   - Use `python-dotenv` to load secrets
   - Validate environment on startup

2. **Input Validation**
   - Validate latitude/longitude ranges
   - Sanitize all string inputs
   - Rate limit API endpoints

3. **Privacy**
   - Do not permanently store user location
   - Clear location data from memory after processing
   - Provide `.env.example` without secrets

4. **Frontend Security**
   - Never expose API keys in frontend code
   - Use CORS properly
   - Implement CSP headers

5. **Backend Security**
   - Validate all external API responses
   - Implement request signing
   - Use HTTPS only in production

---

## 9. Deployment Strategy

### Development
- Local PostgreSQL + PostGIS
- Docker Compose for local dev environment
- Mock train data provider

### Production
- Containerized backend (Docker)
- Managed PostgreSQL (AWS RDS, Google Cloud SQL, or similar)
- PostGIS extension enabled
- Redis for caching/sessions
- CDN for frontend assets
- WebSocket load balancing (if needed)

### Scaling Considerations
- Horizontally scale backend with load balancer
- Use Redis for distributed WebSocket state
- Implement database connection pooling
- Cache frequently accessed data (gates, stations)

---

## 10. Testing Strategy

### Unit Tests
- Haversine distance calculation
- Train direction detection
- ETA prediction engine
- Confidence scoring

### Integration Tests
- Database operations
- API endpoints
- WebSocket connections
- Train data provider integration

### End-to-End Tests
- User location → nearest gate flow
- Train detection → prediction → notification flow
- Multiple trains handling
- Error recovery scenarios

### Test Scenarios (Covered)
1. User near railway gate
2. User far from gates
3. No approaching train
4. One approaching train
5. Multiple approaching trains
6. Train moving toward gate
7. Train moving away from gate
8. Train without scheduled stop
9. Train with scheduled stop
10. Stale train data
11. API failure
12. GPS failure
13. Invalid coordinates
14. Multiple railway lines

---

## 11. Extensibility Design

### Easy to Add
- **New Railway Gates**: Add to database, no code changes
- **New Railway Stations**: Add to database, no code changes
- **New Regions/States**: Add data, deploy, no architecture change

### Provider Abstraction
```python
class TrainDataProvider(ABC):
    @abstractmethod
    async def fetch_train_positions(self) -> List[Train]:
        pass

class LiveAPIProvider(TrainDataProvider):
    async def fetch_train_positions(self):
        # Connect to verified live API
        pass

class MockProvider(TrainDataProvider):
    async def fetch_train_positions(self):
        # Return mock data for testing
        pass
```

This allows easy switching between providers.

---

## 12. Development Phases

**Phase 1**: Frontend + Backend foundation, GPS, Map display  
**Phase 2**: Database, Railway gates/stations data  
**Phase 3**: Mock train data provider, basic API integration  
**Phase 4**: Train direction detection, ETA prediction  
**Phase 5**: Real-time WebSocket, notifications  
**Phase 6**: Testing, error handling, documentation  

---

## 13. Important Disclaimers

- **Not a Safety-Critical System**: This is informational only
- **Not a Railway Signalling System**: Do not rely for safety decisions
- **Always Follow Official Signals**: Users must obey railway barriers/signals
- **Prediction Uncertainty**: ETA is an estimate with confidence scoring
- **Data Source Verification**: Only use verified, authorized APIs

---

This architecture is designed for:
- ✅ Production readiness
- ✅ Scalability
- ✅ Maintainability
- ✅ Security
- ✅ Extensibility
- ✅ Clear data flow
- ✅ Error resilience
- ✅ Real-time capability
