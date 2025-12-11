# System Architecture and Data Flow Document

**Document Title:** IoT Sensor Monitoring Platform - Architecture & Detailed Flow

**Date:** December 2025 (Based on code logs)

**Backend Framework:** NestJS (TypeScript)

**Frontend Framework:** Next.js (React)

---


![Dayliff IOT Telemetry Platform Architecture](https://raw.githubusercontent.com/jobmathenge/test-markdown/main/Dayliff%20IOT%20Telemetry%20plaform.png)

## Alarm System Testing Matrix (Table Summary)

This table consolidates the MQTT connection parameters and the specific test payloads required to trigger or clear each defined alert, ensuring complete coverage in one concise format.

| Section | Metric (Topic) | Full Topic Name | Broker/Test Detail | Trigger Payload | Clear Payload | Expected Result |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Broker** | URL/Auth | N/A | **URL:** `mqtts://5e86...hivemq.cloud:8883` **User:** `hivemq.webclient...` **Pass:** `w8QcE2GF!S$5h3xf#%Hg` | N/A | N/A | Connection success required for all tests. |
| **Test Case 1** | Temperature | `client1/temperature` | **Trigger Cond:** $> 40.0^\circ\text{C}$ **Clear Cond:** $\le 35.0^\circ\text{C}$ | `"42.5"` | `"34.9"` | Alert: **Active** $\rightarrow$ Alert: **Cleared** |
| **Test Case 2** | Flowrate | `client1/flowrate` | **Trigger Cond:** $> 12.0 \text{ L/s}$ **Clear Cond:** $\le 10.0 \text{ L/s}$ | `"15.0"` | `"9.9"` | Alert: **Active** $\rightarrow$ Alert: **Cleared** |
| **Test Case 3** | Power | `client1/power` | **Trigger Cond:** $= 0.0 \text{ kW}$ **Clear Cond:** $> 0.0 \text{ kW}$ | `"0.0"` | `"0.1"` | Alert: **Active** (Outage) $\rightarrow$ Alert: **Cleared** |
| **Test Case 4** | Humidity | `client1/humidity` | **Trigger Cond:** $\ge 80.0\%$ **Clear Cond:** $< 70.0\%$ | `"85.0"` | `"69.9"` | Alert: **Active** $\rightarrow$ Alert: **Cleared** |
| **Test Case 5** | Current | `client1/current` | **Trigger Cond:** $\ge 10.0 \text{ A}$ **Clear Cond:** $< 9.0 \text{ A}$ | `"12.0"` | `"8.9"` | Alert: **Active** $\rightarrow$ Alert: **Cleared** |



## 1. High-Level System Architecture

The application is structured around a centralized NestJS API, which is responsible for data ingestion, real-time communication, security, and serving both the application logic and historical data.



| Layer | Key Components | Function |
| :--- | :--- | :--- |
| **Data Source** | MQTT Broker, Sensors | Publishes real-time sensor readings to the specific topics below. |
| **Backend (API)** | `MqttService`, `DataService`, `AlertsService`, `AuthService` | Ingests data, runs alert logic, saves to DB, handles authentication, and provides REST/WS APIs. |
| **Database** | PostgreSQL / TypeORM | Stores all `SensorReadingEntity` data and `AlertEntity` history. |
| **Frontend (Client)** | Next.js Components (`DataChart`, `AlertBell`) | Handles user login, displays dashboard tiles, renders charts, and monitors real-time alerts. |

---

## 2. 🔑 User Authentication and Role Management

The security model is built on JWTs and NestJS Guards (`JwtAuthGuard` and `RolesGuard`).

### 2.1. Test User Accounts and Roles

The authentication service is comparing credentials against mock user data where the test password `"testpass"`.

| Account | Role | Required JWT Claim | password
| :--- | :--- | :--- | :--- |
| `admin@test.com` | **Admin** | `role: 'Admin'` |`testpass` |
| `user@test.com` | **User** | `role: 'User'` | `testpass` |

### 2.2. Authentication Flow

| Step | Component | Detail |
| :--- | :--- | :--- |
| **1. Login Request** | `login-form.tsx` $\rightarrow$ `POST /auth/login` | Sends `email` and  `password`. |
| **2. Validation** | `AuthService.validateUser` | Compares password against user data's `password` field. |
| **3. JWT Generation** | `AuthService.login` | Creates a JWT with payload: `{ sub: userId, email: user.email, **role: user.role** }`. |
| **4. Authorization Check** | `api-fetcher.ts` / `RolesGuard` | Requests include `Authorization: Bearer <JWT>`. `RolesGuard` ensures `user.role` matches the route's required role (e.g., `@Roles(Role.Admin)`). |

---

## 3. 🌡️ Data Ingestion and Alerting Flow (Detailed)

This flow details the processing of a single sensor reading, including persistence and alert monitoring.

### 3.1. Subscribed MQTT Topics (Full List)

| Topic Name (Full) | UI Display Name | Unit |
| :--- | :--- | :--- |
| `client1/temperature` | Temperature | °C |
| `client1/humidity` | Humidity | % |
| `client1/flowrate` | Water Flowrate | L/s |
| `client1/power` | Power Consumption | kW |
| `client1/cumulative` | Cumulative Energy | kWh |
| `client1/current` | Current Draw | A |

### 3.2. Alert Rules and Critical Ranges (`AlertsService` Logic)

The system checks for critical conditions using simple thresholds with hysteresis to prevent rapid on/off state changes.

| Metric | Trigger Condition | **Clearing Condition (Hysteresis)** | Initial Alert Message |
| :--- | :--- | :--- | :--- |
| **Temperature** | Value **> $40.0^\circ\text{C}$** | Value **$\le 35.0^\circ\text{C}$** | `Temperature CRITICAL HIGH...` |
| **Flowrate** | Value **$> 12.0 \text{ L/s}$** | Value **$\le 10.0 \text{ L/s}$** | `Flowrate CRITICAL HIGH...` |
| **Power** | Value **$= 0.0 \text{ kW}$** | Value **$> 0.0 \text{ kW}$** | `Power CRITICAL OUTAGE (NO POWER AT SITE)` |
| **Humidity** | Value **$\ge 80.0\%$** | Value **$< 70.0\%$** | *(Placeholder: High Humidity)* |
| **Current** | Value **$\ge 10.0 \text{ A}$** | Value **$< 9.0 \text{ A}$** | *(Placeholder: High Current)* |

### 3.3. Data Ingestion Sequence

| Step | Component(s) | Action |
| :--- | :--- | :--- |
| **1. Receive** | `MqttService` | Receives raw payload from the MQTT Broker. |
| **2. Validate** | `DataService` | Parses value and calls `alertsService.checkAndCreateAlert(reading)`. |
| **3. Alert Check** | `AlertsService` | **A.** Runs the rules from 3.2. **B.** Saves a new 'Active' `AlertEntity` if triggered. **C.** Clears a previous 'Active' alert if clearing condition is met. |
| **4. Persistence** | `DataService` | Saves `SensorReadingEntity` to PostgreSQL. |
| **5. Broadcast** | `DataGateway` | Emits the new reading via WebSocket (topic `'new_reading'`). |

---

## 4. 📊 Client Data Retrieval and Reporting Flow

This details the data retrieval pattern for the dashboard UI elements.

| Feature | Protocol | Endpoint / Key Action | Purpose |
| :--- | :--- | :--- | :--- |
| **Status Tiles** | REST | `GET /data/latest` | Initial load to show the last known value for all 6 topics. |
| **Data Chart** | REST | `GET /data/history?startTime=...` | Retrieves large historical data ranges for all topics upon user date selection. |
| **Live Updates** | WebSocket | Subscription to `'new_reading'` | Real-time updates for tiles and charts (live stream). |
| **Active Alert Count** | REST | `GET /alerts/count` | Used by `AlertBell.tsx` to display the red badge and trigger the ring animation. |
| **Acknowledge Alert** | REST | `PATCH /alerts/acknowledge/:id` | Changes the alert's status from 'Active' to 'Acknowledged' in the database. |
| **Export PDF** | REST | `GET /report/pdf` | Triggers a complex service call to fetch sensor data and alert summaries, generate a PDF using `pdfmake`, and stream the binary file. |




