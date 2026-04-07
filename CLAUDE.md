# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

CASSAVA_IOT is a full-stack Smart Farming IoT platform for managing cassava crop fields with real-time sensor monitoring and automated irrigation control. It consists of a React frontend and a Spring Boot backend connected to MongoDB.

## Build & Run Commands

### Frontend (`CassavaFE/`)
```bash
cd CassavaFE
npm install           # install dependencies
npm run dev           # dev server at http://localhost:5173
npm run build         # production build to dist/
npm run lint          # ESLint
npm run preview       # preview production build
```

### Backend (`cassavaBE/`)
```bash
cd cassavaBE
mvn clean install     # build + download dependencies
mvn spring-boot:run   # run at http://localhost:8081
mvn test              # run tests
```

## Architecture

```
React SPA (5173) ──Axios──▶ Spring Boot API (8081) ──▶ MongoDB (remote)
                                    │
                          ┌─────────┼─────────┐
                       MQTT       NASA API   Firebase
                    (HiveMQ)    (weather)   (storage)
```

### Backend (`cassavaBE/src/main/java/com/example/demo/`)

- **controller/**: REST endpoints — `FieldMongoController` (`/mongo/field`), `UserController` (`/api/auth`), `SensorValueController` (`/api/sensor-values`)
- **service/**: Business logic — `FieldMongoService`, `MqttWeatherService` (MQTT subscriber + NASA fallback), `SensorValueService`
- **entity/**: MongoDB document models — `Field`, `User`, `FieldSensor`, `SensorValue`
- **repositories/**: Spring Data MongoDB repos
- **Jwt/**: `JwtUtils` (token gen/validation), `JwtAuthFilter` (request filter)
- **config/**: `SecurityConfig` (CORS + auth rules), `WebConfig`
- **firebase/**: Firebase Realtime Database integration

Auth flow: JWT with 24h expiry. Roles are ADMIN and USER. Public endpoints: `/api/auth/**` only. Token injected via `JwtAuthFilter` in the Spring Security filter chain.

### Frontend (`CassavaFE/src/`)

- **pages/**: Login, Register, FieldList, FieldDetail (4 tabs: disease, irrigation, yield, history), WeatherDashboard, WeatherDetail, UserList
- **services/**: API clients (`api.js` has Axios interceptor for JWT, `fieldService.js`, `authService.js`, `weatherService.js`)
- **contexts/**: `AuthContext` for global auth state
- **routes/**: Route config with `PrivateRoute` protection
- **components/**: `MainLayout` (sidebar+header), `FieldModal`
- **utils/**: `exportExcel.js` (Excel export), `formatters.js`

UI framework: Ant Design. Routing: React Router DOM.

### IoT Data Flow

MQTT broker (HiveMQ `broker.hivemq.com:1883`) → topic `/sensor/weatherStation` → `MqttWeatherService` validates and persists JSON sensor readings (`t`, `h`, `rai`, `rad`, `w`) to `sensor_value` collection. Falls back to NASA Power API if MQTT data is invalid, with a 10-minute cooldown between NASA calls.

### Key External Dependencies

- **MongoDB**: Remote instance at `112.137.129.218:27017`, database `iot_agriculture`
- **Firebase**: Service account key expected at `serviceAccountKey.json` (path configured in `FirebaseInitializer`)
- **MQTT**: HiveMQ public broker
- **APIs**: NASA Power (no key), OpenWeather (key in `MqttWeatherService`)

## Conventions

- Backend packaging is WAR (servlet-based with embedded Tomcat)
- Java 17 required
- CORS is configured to allow only `http://localhost:5173`
- Frontend API base URL: `http://localhost:8081/api` (configured in `CassavaFE/src/services/api.js`)
