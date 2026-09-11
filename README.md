# Railway Gate Train Prediction & Alert System

A real-time full-stack application that detects user location, identifies nearby railway gates, tracks approaching trains, and predicts train arrival times with warning notifications.

## 📋 Project Overview

This system:
- ✅ Detects user's GPS location
- ✅ Identifies nearest railway level crossing/gate
- ✅ Displays live train positions and movements
- ✅ Predicts train arrival times at the railway gate
- ✅ Sends real-time notifications for approaching trains
- ✅ Calculates user waiting time estimates
- ✅ Distinguishes trains moving toward vs. away from the gate
- ✅ Handles multiple approaching trains simultaneously
- ✅ Works with mock train data (ready for real API integration)

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────┐
│           Frontend (React.js + Leaflet)         │
│  - GPS Location Detection                       │
│  - Map Display (User, Gate, Trains)            │
│  - Real-time Updates (WebSocket)               │
│  - Notifications & Alerts                      │
└────────────────────┬────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
   ┌─────────────┐         ┌──────────────┐
   │  WebSocket  │         │  REST API    │
   │  /ws/gate   │         │  /api/*      │
   └─────────────┘         └──────────────┘
        │                         │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │   Backend (FastAPI)     │
        │  - Gate Management      │
        │  - Station Management   │
        │  - Train Tracking       │
        │  - ETA Prediction       │
        │  - Real-time Updates    │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │  Services & Engines     │
        │  - Train Data Service   │
        │  - Prediction Engine    │
        │  - Geospatial Utils     │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │  PostgreSQL + PostGIS   │
        │  - Railway Gates        │
        │  - Stations             │
        │  - Train Positions      │
        │  - Historical Data      │
        └────────────────────────┘
```

## 🚀 Quick Start

### Prerequisites
- Python 3.9+
- Node.js 16+
- PostgreSQL 12+ with PostGIS extension
- Docker & Docker Compose (optional)

### Setup with Docker (Recommended)

```bash
# Clone repository
git clone https://github.com/HemanthSSG/railway-gate-alert-system.git
cd railway-gate-alert-system

# Copy environment file
cp .env.example .env

# Start services
docker-compose up -d

# Initialize database
docker-compose exec backend python -m app.scripts.init_db

# Frontend available at http://localhost:3000
# Backend available at http://localhost:8000
```

## ⚠️ Important Safety Disclaimer

**This application is an INFORMATIONAL PREDICTION SYSTEM, NOT an official railway safety/signaling system.**

- Predictions are estimates based on available data
- Always follow official railway signals and barriers
- Never cross a railway gate when it is closed or when a train is approaching
- In case of doubt, wait for the railway barrier to open fully

## 📚 Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) - Detailed system design
- [backend/README.md](./backend/README.md) - Backend setup & API docs
- [frontend/README.md](./frontend/README.md) - Frontend setup & UI guide
- [data/README.md](./data/README.md) - Data structure & sample datasets

## 📝 License

MIT License
