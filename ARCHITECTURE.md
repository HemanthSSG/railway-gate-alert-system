# Railway Gate Train Prediction & Alert System - Architecture Document

## System Overview

This document describes the architecture, design decisions, and implementation details of the Railway Gate Train Prediction & Alert System.

## 1. Technical Stack

### Frontend
- **React.js** - UI framework
- **TypeScript/JavaScript** - Programming language
- **Leaflet** - Map display and management
- **Axios** - HTTP client
- **WebSocket API** - Real-time communication
- **CSS3** - Styling

### Backend
- **Python 3.9+** - Programming language
- **FastAPI** - Web framework
- **Uvicorn** - ASGI server
- **SQLAlchemy** - ORM
- **Pydantic** - Data validation
- **Python-socketio** - WebSocket support

### Database
- **PostgreSQL 12+** - Relational database
- **PostGIS** - Geospatial extension
- **pgAdmin** - Database management tool

### Infrastructure
- **Docker** - Containerization
- **Docker Compose** - Multi-container orchestration
- **Redis** (Optional) - Caching and pub/sub

## 2. Data Flow

### Real-time Train Tracking Flow

```
┌─────────────┐
│ User Device │
│ (Browser)   │
└──────┬──────┘
       │ 1. Share GPS Location
       │ 2. Get Nearest Gate
       ▼
┌─────────────────────────────┐
│ Frontend (React)            │
│ - GPS Geolocation API       │
│ - Leaflet Map Display       │
│ - WebSocket Connection      │
└──────┬──────────────────────┘
       │ 3. REST API Calls
       │ 4. WebSocket Subscribe
       ▼
┌─────────────────────────────┐
│ Backend (FastAPI)           │
│ - Gate Management API       │
│ - WebSocket Manager         │
│ - Prediction Engine         │
└──────┬──────────────────────┘
       │ 5. Query Database
       │ 6. Fetch Train Data
       ▼
┌─────────────────────────────┐
│ Services Layer              │
│ - Gate Service              │
│ - Train Data Service        │
│ - Prediction Engine         │
│ - Geospatial Calculator     │
└──────┬──────────────────────┘
       │ 7. Database Query
       │ 8. Mock/Real API Call
       ▼
┌─────────────────────────────┐
│ Data Layer                  │
│ - PostgreSQL + PostGIS      │
│ - Train Data Provider       │
└─────────────────────────────┘
```

## 3. Database Schema

### Tables

#### railway_gates
```sql
CREATE TABLE railway_gates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    geom GEOMETRY(Point, 4326) NOT NULL,
    station_id UUID NOT NULL REFERENCES railway_stations(id),
    railway_line VARCHAR(100) NOT NULL,
    station_distance FLOAT,  -- Distance from station to gate in KM
    estimated_travel_time INT,  -- Estimated travel time in seconds
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_geom (geom),
    INDEX idx_railway_line (railway_line)
);
```

#### railway_stations
```sql
CREATE TABLE railway_stations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    geom GEOMETRY(Point, 4326) NOT NULL,
    railway_line VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_geom (geom),
    INDEX idx_railway_line (railway_line)
);
```

#### trains
```sql
CREATE TABLE trains (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    train_number VARCHAR(50) UNIQUE NOT NULL,
    train_name VARCHAR(255) NOT NULL,
    railway_line VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_train_number (train_number),
    INDEX idx_railway_line (railway_line)
);
```

#### train_positions
```sql
CREATE TABLE train_positions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    train_id UUID NOT NULL REFERENCES trains(id),
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    geom GEOMETRY(Point, 4326) NOT NULL,
    speed_kmph FLOAT,
    direction VARCHAR(50),  -- N, S, E, W, NE, NW, SE, SW
    timestamp TIMESTAMP NOT NULL,
    is_stale BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_train_id (train_id),
    INDEX idx_timestamp (timestamp),
    INDEX idx_geom (geom)
);
```

## 4. API Endpoint Design

### REST Endpoints

#### Gates
```
GET /api/v1/gates
  Query Parameters: limit, offset, railway_line
  Response: [GateResponse]
  
GET /api/v1/gates/{gate_id}
  Response: GateResponse
  
GET /api/v1/gates/nearest
  Query Parameters: latitude, longitude, radius_km
  Response: GateResponse
```

#### Stations
```
GET /api/v1/stations
  Query Parameters: limit, offset, railway_line
  Response: [StationResponse]
  
GET /api/v1/stations/{station_id}
  Response: StationResponse
```

#### Trains
```
GET /api/v1/trains/nearby
  Query Parameters: gate_id, radius_km
  Response: [TrainResponse]
  
GET /api/v1/trains/{train_number}
  Response: TrainResponse
```

#### Predictions
```
GET /api/v1/predictions/{gate_id}
  Response: PredictionResponse
  
GET /api/v1/predictions/{gate_id}/trains
  Response: [TrainPredictionResponse]
```

#### Health
```
GET /api/v1/health
  Response: {status: "healthy", timestamp: ISO8601}
```

### WebSocket Endpoints

```
WS /ws/gate/{gate_id}
  Subscribe to real-time updates for a specific gate
  Message Types:
    - train_approaching
    - train_entered
    - train_passed
    - prediction_update
    - no_train_detected
```

## 5. Prediction Engine

### ETA Calculation Algorithm

```python
def calculate_eta(train_position, gate_position, train_speed, historical_travel_time):
    """
    Calculate estimated time of arrival at the gate.
    
    Args:
        train_position: (lat, lon)
        gate_position: (lat, lon)
        train_speed: km/h (may be None)
        historical_travel_time: seconds (from station to gate)
    
    Returns:
        eta_seconds: Estimated seconds until train reaches gate
    """
    
    # Calculate straight-line distance using Haversine formula
    distance_km = haversine_distance(train_position, gate_position)
    
    # Method 1: Use current speed if available and reliable
    if train_speed and train_speed > 0 and is_speed_reliable(train_speed):
        # Convert speed from km/h to km/s
        speed_km_per_second = train_speed / 3600
        eta_seconds = distance_km / speed_km_per_second
        confidence = "HIGH"
        
    # Method 2: Use historical travel time as fallback
    elif historical_travel_time:
        # Scale historical time by current distance vs historical distance
        historical_distance = get_station_to_gate_distance(gate_id)
        if historical_distance > 0:
            time_ratio = distance_km / historical_distance
            eta_seconds = historical_travel_time * time_ratio
            confidence = "MEDIUM"
        else:
            eta_seconds = historical_travel_time
            confidence = "LOW"
    
    # Method 3: Default assumption (50 km/h average)
    else:
        average_speed_kmh = 50
        speed_km_per_second = average_speed_kmh / 3600
        eta_seconds = distance_km / speed_km_per_second
        confidence = "LOW"
    
    return {
        "eta_seconds": max(0, eta_seconds),
        "distance_km": distance_km,
        "confidence": confidence,
        "method": method_used
    }
```

### Train Approaching Detection

```python
def is_train_approaching(train_current_pos, train_prev_pos, gate_pos, threshold_km=5.0):
    """
    Determine if train is moving toward the gate.
    
    Conditions:
    1. Train distance to gate < threshold
    2. Distance is decreasing (current < previous)
    3. Train direction aligns with gate direction
    """
    
    current_distance = haversine_distance(train_current_pos, gate_pos)
    previous_distance = haversine_distance(train_prev_pos, gate_pos)
    
    # Check if train is within threshold and getting closer
    is_close = current_distance < threshold_km
    is_getting_closer = current_distance < previous_distance
    
    # Calculate direction vector from train to gate
    gate_direction = calculate_bearing(train_current_pos, gate_pos)
    train_direction_deg = parse_direction(train.direction)
    
    # Check if direction is within ±45 degrees of gate
    direction_diff = abs(gate_direction - train_direction_deg)
    direction_diff = min(direction_diff, 360 - direction_diff)
    is_aligned = direction_diff < 45
    
    return is_close and is_getting_closer and is_aligned
```

## 6. Geospatial Calculations

### Haversine Formula

```python
import math

def haversine_distance(lat1, lon1, lat2, lon2, radius_km=6371):
    """
    Calculate great-circle distance between two points.
    
    Args:
        lat1, lon1: First point (degrees)
        lat2, lon2: Second point (degrees)
        radius_km: Earth radius in kilometers (default: 6371)
    
    Returns:
        distance: Distance in kilometers
    """
    lat1_rad = math.radians(lat1)
    lon1_rad = math.radians(lon1)
    lat2_rad = math.radians(lat2)
    lon2_rad = math.radians(lon2)
    
    dlat = lat2_rad - lat1_rad
    dlon = lon2_rad - lon1_rad
    
    a = math.sin(dlat/2)**2 + math.cos(lat1_rad) * math.cos(lat2_rad) * math.sin(dlon/2)**2
    c = 2 * math.asin(math.sqrt(a))
    
    return radius_km * c
```

### Bearing Calculation

```python
def calculate_bearing(start_lat, start_lon, end_lat, end_lon):
    """
    Calculate bearing from start to end point (0-360 degrees).
    0° = North, 90° = East, 180° = South, 270° = West
    """
    start_lat_rad = math.radians(start_lat)
    start_lon_rad = math.radians(start_lon)
    end_lat_rad = math.radians(end_lat)
    end_lon_rad = math.radians(end_lon)
    
    dlon = end_lon_rad - start_lon_rad
    
    y = math.sin(dlon) * math.cos(end_lat_rad)
    x = math.cos(start_lat_rad) * math.sin(end_lat_rad) - \
        math.sin(start_lat_rad) * math.cos(end_lat_rad) * math.cos(dlon)
    
    bearing_rad = math.atan2(y, x)
    bearing_deg = (math.degrees(bearing_rad) + 360) % 360
    
    return bearing_deg
```

## 7. Train Data Provider Interface

### Abstract Provider

```python
from abc import ABC, abstractmethod
from typing import List
from datetime import datetime

class TrainDataProvider(ABC):
    """
    Abstract base class for train data providers.
    Implementations can fetch from mock data, real APIs, etc.
    """
    
    @abstractmethod
    async def get_train_positions(self, railway_line: str) -> List[TrainPosition]:
        """Fetch current positions of all trains on a railway line."""
        pass
    
    @abstractmethod
    async def get_train_by_number(self, train_number: str) -> TrainPosition:
        """Fetch position of a specific train."""
        pass
    
    @abstractmethod
    async def get_trains_in_region(self, lat: float, lon: float, radius_km: float) -> List[TrainPosition]:
        """Fetch trains within a geographic region."""
        pass
    
    @property
    @abstractmethod
    def is_healthy(self) -> bool:
        """Check if data provider is functioning correctly."""
        pass
    
    @property
    @abstractmethod
    def last_update_time(self) -> datetime:
        """Return timestamp of last successful data fetch."""
        pass
```

### Mock Provider Implementation

The mock provider generates realistic simulated train data for development and testing.

## 8. Real-time Communication (WebSocket)

### Message Format

```json
{
  "type": "train_approaching",
  "gate_id": "uuid",
  "train_number": "12345",
  "eta_seconds": 120,
  "distance_km": 2.5,
  "speed_kmph": 60,
  "timestamp": "2026-09-11T10:30:00Z",
  "message": "Train 12345 approaching railway gate. ETA: 2 minutes."
}
```

### Connection Manager

- Manages active WebSocket connections per gate
- Broadcasts updates to all connected clients
- Handles connection/disconnection events
- Automatic reconnection logic on frontend

## 9. Error Handling Strategy

### Backend Errors
1. **Database Errors** → Log, return 500 with generic message
2. **API Timeout** → Retry with exponential backoff
3. **Invalid Input** → Return 400 with validation error
4. **Not Found** → Return 404
5. **Rate Limited** → Return 429, inform client to retry

### Frontend Errors
1. **GPS Unavailable** → Show fallback input for manual location
2. **Network Offline** → Show offline indicator, cache last data
3. **WebSocket Disconnected** → Auto-reconnect, show status
4. **API Error** → Display user-friendly error message

## 10. Security Considerations

### API Security
- CORS configuration to allow only frontend origin
- Rate limiting to prevent abuse
- Input validation and sanitization
- SQL injection prevention via ORM

### Data Privacy
- User GPS location NOT persisted
- Train data treated as public information
- No personal information collected
- HTTPS/WSS for encrypted communication

### Environment & Secrets
- Database credentials in environment variables
- API keys never exposed to frontend
- `.env` file in `.gitignore`
- `.env.example` provided as template

## 11. Deployment Architecture

### Development
- Docker Compose stack
- PostgreSQL + PostGIS in Docker
- Backend and Frontend in separate containers
- Hot reload enabled for development

### Production
- Backend: Deployed as containerized service (Docker/Kubernetes)
- Frontend: Static files served via CDN or nginx
- Database: Managed PostgreSQL service (AWS RDS, Azure, etc.)
- Load balancing and auto-scaling enabled

## 12. Monitoring & Logging

### Metrics
- API response times
- WebSocket connection count
- Train data freshness
- Prediction accuracy
- Error rates

### Logs
- Application logs to stdout (container-friendly)
- Database query logs (development only)
- WebSocket event logs
- Train prediction logs for analysis

## 13. Limitations & Future Improvements

### Current Limitations
1. Straight-line distance calculation (doesn't follow railway tracks)
2. No real railway track geometry data
3. Mock train data for development
4. Single geographic region support

### Future Enhancements
1. Integrate real railway track data (OpenRailwayMap API)
2. Support multiple geographic regions
3. Machine learning-based prediction refinement
4. Historical accuracy analytics
5. Mobile app with native notifications
6. Integration with official railway APIs
7. Railway barrier status integration
8. Multi-language support

---

**Last Updated**: September 2026
