# Lohia Farm — GeoSense Precision Agriculture Dashboard

> A precision-agriculture monitoring dashboard for the Lohia Farm weather station.
> React + TypeScript frontend, Firebase Realtime Database for live telemetry and history,
> and Python alert workers for email/SMS notifications.
> Built as a full-stack IoT / environmental-monitoring project by Team GeoSense, AIKTC.

## Overview

**Lohia Farm / GeoSense** is a web-based environmental monitoring system for a farm weather station. The project connects sensor telemetry to a browser dashboard, performs basic data cleaning and derived-weather calculations, displays historical trends, exports recorded data as CSV, and provides an alert subscription system.

The current repository is primarily a **React + Firebase application**. There is no Express/Node.js API server in the current implementation. A separate set of Python workers under `email_alerts/` listens to Firebase and sends email/SMS notifications.

The system currently works with these measurements:

- Temperature
- Relative humidity
- Atmospheric pressure
- Ambient light / illuminance
- CO₂ concentration
- PM1.0
- PM2.5
- PM10
- Calculated India AQI
- Sensor heartbeat / online state
- Device uptime and reported sensor count

The repository also contains an experimental data-processing / machine-learning workspace under `src/Median and Data chnages/`.

## Main Features

### Live monitoring

The dashboard subscribes directly to the Firebase Realtime Database at the `weather` path. New records update the UI without requiring a separate REST API layer.

The main dashboard presents:

- Temperature
- Humidity
- Pressure
- Ambient light
- CO₂
- AQI
- PM1.0 / PM2.5 / PM10 breakdown
- Current sensor count
- Last telemetry update
- Reported uptime
- Connection / offline state

### Historical trends

Trend data is read from the same `weather` collection in Firebase.

The trend popup supports:

- 1 hour
- 24 hours
- 7 days
- 30 days

Longer ranges are sampled before rendering to keep the chart manageable. Large gaps in telemetry are represented as gaps instead of connecting unrelated points with a continuous line.

### Data export

The dashboard can export cleaned weather records as CSV for:

- Today
- All available records
- A custom date range

The export contains date/time and the main environmental fields, including temperature, humidity, pressure, light, CO₂, PM1.0, PM2.5, PM10, uptime, and sensor count.

Before export, the frontend runs the same general median-based cleaning strategy used by the live data pipeline.

### Alert subscriptions

Users can subscribe or unsubscribe using:

- Email address
- Optional phone number

Subscriber records are stored in Firebase under `subscribers`.

The Python subscriber worker watches this path and sends confirmation messages. Unsubscribe requests are marked in Firebase first so the worker can send the confirmation before deleting the subscriber record.

### Threshold alerts

The Python alert engine monitors incoming weather records and currently checks these configured thresholds:

| Parameter | Alert threshold |
|---|---:|
| Temperature | `> 40 °C` |
| Humidity | `> 80 %` |
| CO₂ | `> 1200 ppm` |
| India AQI | `> 150` |

Threshold alerts have a one-hour cooldown.

The alert engine also has a sensor-offline watchdog. If the weather station has not produced a new record for more than five minutes, it can send an offline notification.

### Email and SMS notifications

Email notifications are implemented with **Apprise** and Gmail credentials supplied through environment variables.

SMS notifications use **Apprise's Twilio integration**.

The notification workers cover:

- Threshold-breach email alerts
- Threshold-breach SMS alerts
- Full station offline alerts
- Partial sensor-fault alerts
- Subscriber welcome messages
- Subscriber unsubscribe confirmations

## Environmental Calculations

The frontend contains a small weather-physics utility module at `src/lib/weather-physics.ts`.

### Heat index / feels-like temperature

`WeatherPhysics.getFeelsLike()` converts Celsius to Fahrenheit, applies the NWS Rothfusz heat-index regression when appropriate, and converts the result back to Celsius.

### Vapor pressure

`WeatherPhysics.getVaporPressure()` estimates actual vapor pressure from temperature and relative humidity using a Tetens-style saturation vapor pressure equation.

### Absolute humidity

`WeatherPhysics.getAbsoluteHumidity()` estimates absolute humidity in `g/m³` from temperature and relative humidity.

### CO₂ partial pressure

`WeatherPhysics.getCO2PartialPressure()` calculates the partial pressure contribution of CO₂ from total pressure and concentration in ppm.

This helper exists in the codebase but is not currently a primary dashboard metric.

### India AQI

`WeatherPhysics.calculateIndiaAQI()` calculates an AQI value from PM2.5 and PM10 using the breakpoint tables implemented in the project.

The same general AQI calculation is duplicated in the Python alert workers so that alerts can be evaluated without depending on the browser.

## Data Cleaning and Median Replacement

Sensor devices can produce invalid values such as `0`, `NaN`, or known error-code values. The project therefore uses historical medians as replacement values.

The live hook `src/hooks/use-farm-data.ts`:

1. Reads the weather history from Firebase.
2. Collects valid positive values for each supported sensor field.
3. Calculates a median for each field.
4. Selects the latest weather record.
5. Replaces invalid or known-bad values with the corresponding historical median.
6. Calculates AQI from the cleaned PM2.5 and PM10 values.
7. Publishes the cleaned record and system state to the React application.

Known invalid-value checks include:

- Non-positive values
- `NaN`
- Humidity exactly equal to `100`
- Temperature exactly equal to `125`
- CO₂ values at or above `3000 ppm`

The same two-pass cleaning approach is implemented in `src/Median and Data chnages/dataCleaner.js` for downloaded/exported data.

## Architecture

The project has four main layers.

### 1. Browser application

The React application is responsible for rendering the dashboard and interacting directly with Firebase.

```text
Browser
  |
  +-- React / TypeScript
  |     |
  |     +-- Dashboard UI
  |     +-- Live data hook
  |     +-- Historical data hook
  |     +-- Weather calculations
  |     +-- CSV export
  |     +-- Subscriber UI
  |
  +-- Firebase Realtime Database
        |
        +-- weather
        +-- subscribers
```

### 2. Firebase Realtime Database

The browser reads live and historical weather telemetry from `weather`.

Subscriber records are stored in `subscribers`.

The frontend Firebase configuration is loaded from Vite environment variables rather than hard-coded credentials:

```text
VITE_FIREBASE_API_KEY
VITE_FIREBASE_AUTH_DOMAIN
VITE_FIREBASE_DATABASE_URL
VITE_FIREBASE_PROJECT_ID
VITE_FIREBASE_STORAGE_BUCKET
VITE_FIREBASE_MESSAGING_SENDER_ID
VITE_FIREBASE_APP_ID
VITE_FIREBASE_MEASUREMENT_ID
```

### 3. Python alert workers

The `email_alerts/` directory contains long-running Python processes that connect to Firebase using the Firebase Admin SDK.

```text
Firebase Realtime Database
        |
        +--> threshold_alerts.py
        |       +--> email alerts
        |       +--> offline email alerts
        |
        +--> sms_alerts.py
        |       +--> SMS threshold alerts
        |       +--> SMS offline alerts
        |
        +--> subscriber_alerts.py
                +--> welcome messages
                +--> unsubscribe confirmations
```

These workers are separate from the Vite frontend and must be started independently if alert delivery is required.

### 4. Data-processing workspace

`src/Median and Data chnages/` contains:

- Raw CSV data
- Cleaned/preprocessed CSV data
- A Jupyter notebook for exploratory preprocessing
- A JavaScript median checker
- A reusable JavaScript data-cleaning utility

The notebook currently performs missing-value replacement using column means and writes a preprocessed CSV. It is best treated as an experimental analysis workspace rather than a production ML pipeline.

## Repository Structure

```text
lohia-farm-weather-hub-05-main/
├── README.md
├── package.json
├── package-lock.json
├── vite.config.ts
├── vitest.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── tailwind.config.ts
├── postcss.config.js
├── eslint.config.js
├── components.json
├── database.rules.json
├── index.html
│
├── public/
│   ├── robots.txt
│   └── team/
│       ├── Abdullah.png
│       ├── Danish.png
│       ├── Hussain.png
│       ├── Idris.png
│       ├── Irfan.png
│       ├── Mueez.png
│       ├── Rehan.png
│       ├── Zaid.png
│       └── team
│
├── email_alerts/
│   ├── config.py
│   ├── threshold_alerts.py
│   ├── sms_alerts.py
│   └── subscriber_alerts.py
│
└── src/
    ├── App.tsx
    ├── App.css
    ├── index.css
    ├── main.tsx
    │
    ├── assets/
    │   ├── geofav.png
    │   └── geosenseLogo.png
    │
    ├── components/
    │   ├── NavLink.tsx
    │   ├── dashboard/
    │   │   ├── DashboardHeader.tsx
    │   │   ├── DashboardFooter.tsx
    │   │   ├── DataDownloader.tsx
    │   │   ├── HeroSection.tsx
    │   │   ├── LoadingOverlay.tsx
    │   │   ├── MetricCard.tsx
    │   │   ├── SubscribeAlerts.tsx
    │   │   ├── SystemStatus.tsx
    │   │   ├── TrendChart.tsx
    │   │   └── TrendPopup.tsx
    │   │
    │   └── ui/
    │       └── shadcn/Radix-based UI primitives
    │
    ├── hooks/
    │   ├── use-farm-data.ts
    │   ├── use-historical-data.ts
    │   ├── use-mobile.tsx
    │   └── use-toast.ts
    │
    ├── lib/
    │   ├── farmData.ts
    │   ├── firebase.ts
    │   ├── utils.ts
    │   └── weather-physics.ts
    │
    ├── pages/
    │   ├── Index.tsx
    │   └── NotFound.tsx
    │
    ├── test/
    │   ├── example.test.ts
    │   ├── mock-hardware.ts
    │   └── setup.ts
    │
    └── Median and Data chnages/
        ├── Data.csv
        ├── preprocessed_data.csv
        ├── prdtreprocessed_data.csv
        ├── Ml.ipynb
        ├── dataCleaner.js
        └── check-medians.cjs
```

## Frontend Technology Stack

| Layer | Technology |
|---|---|
| UI framework | React 18 |
| Language | TypeScript |
| Build tool | Vite 5 |
| Styling | Tailwind CSS 3 |
| Component primitives | Radix UI / shadcn-style components |
| Icons | Lucide React |
| Charts | Recharts |
| Routing | React Router DOM |
| Async/query utilities | TanStack React Query |
| Database client | Firebase Web SDK |
| Notifications/toasts | Sonner + custom toast components |
| Testing | Vitest + Testing Library/JSDOM |
| Form/schema dependencies | React Hook Form + Zod |

Several UI libraries are present because the repository includes a broad shadcn/Radix component set. Not every installed dependency is necessarily used by the current dashboard page.

## Firebase Data Model

The frontend expects a Realtime Database with a structure broadly similar to:

```text
weather/
  <record-id>/
    timestamp: number
    temperature: number
    humidity: number
    pressure: number
    lux: number
    co2: number
    pm1: number
    pm25: number
    pm10: number
    uptime: string | number
    sensorsOnline: number
    totalSensors: number

subscribers/
  <subscriber-id>/
    email: string
    phone: string
    timestamp: number
    welcome_sent: boolean
    unsubscribe_request: boolean
```

The exact producer-side hardware code is not included in this repository, so the weather-station firmware/data-ingestion implementation is outside the scope of this project.

## Firebase Rules

`database.rules.json` currently indexes:

- `weather.timestamp`
- `subscribers.email`

However, the current rule file contains time-based read/write rules with an expiration timestamp. That timestamp corresponds to **April 5, 2026 UTC**, so these rules are already expired relative to the current repository review date.

Before deploying the application, replace the temporary rules with explicit production access rules. In particular, avoid relying on a globally readable/writable database for a public deployment.


## Running the Python Alert Workers

The repository does not currently contain a `requirements.txt`, `pyproject.toml`, or equivalent Python dependency lock file. The workers therefore need their Python dependencies installed manually or through a dependency file added by the maintainer.

The code requires at least:

```bash
pip install firebase-admin apprise python-dotenv
```

`python-dotenv` is optional in the code because `config.py` falls back to system environment variables when it is unavailable.

Place `firebase_credentials.json` in the project root and configure the environment variables described above.

Because the alert files use package-relative imports such as `from . import config`, run them as Python modules from the project root rather than executing the files as unrelated standalone scripts.

Example:

```bash
python -m email_alerts.threshold_alerts
python -m email_alerts.sms_alerts
python -m email_alerts.subscriber_alerts
```

These processes are long-running listeners. In a deployed environment they should be managed by an appropriate process supervisor, container, VM service, or equivalent runtime.

## Data Processing / Analysis

The data-processing directory contains historical CSV files and a Jupyter notebook.

To inspect the notebook:

```bash
jupyter notebook "src/Median and Data chnages/Ml.ipynb"
```

The notebook currently performs exploratory cleaning such as:

- Inspecting dataframe structure
- Checking missing values
- Filling missing numeric values with column means
- Converting CO₂ to numeric values
- Filling uptime and sensor-count fields
- Exporting a preprocessed CSV

This notebook should not be interpreted as a completed machine-learning model. The repository contains ML-related work and a role for an ML engineer, but the checked-in notebook currently focuses on preprocessing rather than training/deploying a predictive model.

## Mock Hardware Utility

`src/test/mock-hardware.ts` contains a small Node HTTP server intended to simulate hardware API responses.

It exposes example endpoints such as:

```text
GET /api/status
GET /api/weather
```

The current production dashboard does not use these endpoints; it reads directly from Firebase. The mock server is therefore useful as a development/testing utility rather than part of the active data path.

## Languages

The dashboard UI includes translations for:

- English (`en`)
- Hindi (`hi`)
- Marathi (`mr`)

The selected language affects dashboard labels, number formatting, timestamps, and several explanatory strings.

## Theme and Responsive UI

The dashboard supports:

- Dark mode
- Light mode
- Responsive layouts for mobile and desktop
- Scrollable trend charts for longer time ranges
- Modal-based trend inspection
- Mobile-friendly controls

The visual system uses Tailwind CSS with CSS variables for semantic colors and status states.

## Alert and Status Logic

There are several independent layers of status logic in the current implementation.

### Dashboard status bands

The React dashboard assigns `Good`, `Moderate`, or `Poor` status using metric-specific ranges for temperature, humidity, pressure, light, CO₂, and AQI.

These display bands are not identical to the Python alert thresholds. A metric can therefore appear as `Poor` in the dashboard without generating an alert, or cross an alert threshold while using a different visual status boundary.

### Sensor data cleaning

The frontend uses historical medians to repair selected invalid readings before displaying the latest record.

### Firebase connection state

The live hook observes Firebase's `.info/connected` state and exposes that connection state to the dashboard.

### Telemetry heartbeat

The Python alert workers treat a station as fully offline after five minutes without new telemetry.

The React page also contains a client-side heartbeat check. Its current implementation uses a `2000`-second comparison even though the nearby comment describes a roughly five-minute threshold. This is an implementation detail worth reviewing if the UI's offline banner is expected to match the Python watchdog exactly.

## Important Implementation Notes

### The live hook reads the full weather history

`useFarmHub()` currently subscribes to the complete `weather` path and recalculates historical medians from the returned records. This is simple and keeps the median calculation consistent, but it can become expensive as the database grows.

A production-scale implementation should consider one or more of:

- Server-side aggregation
- Periodic median snapshots
- A bounded historical window
- Precomputed statistics
- Time-partitioned data

### Historical queries are indexed by timestamp

The trend popup and CSV exporter use Firebase queries ordered by `timestamp`. The database rules include an index for `weather.timestamp`, which is important for these range queries.

### Firestore is initialized but not currently part of the main data path

`src/lib/firebase.ts` creates both a Realtime Database client and a Firestore client. The checked-in dashboard code primarily uses Realtime Database for weather data and subscribers.

Firestore should therefore be considered available infrastructure rather than an active source for the current dashboard implementation.

### AQI logic is duplicated

The India AQI calculation exists in both TypeScript and Python. If the breakpoint table changes, both implementations need to be updated together or consolidated into a shared service/specification.

### Test coverage is minimal

The current Vitest test suite contains a basic example test and test setup utilities. It does not yet provide comprehensive tests for:

- Firebase data mapping
- Median replacement
- AQI calculations
- Heat-index calculations
- Historical sampling
- CSV export
- Subscription flows
- Alert-worker behavior

For a production deployment, these areas should receive focused unit and integration tests.

## Deployment Considerations

A deployment of the complete system has multiple runtime components:

```text
                    +-----------------------+
                    | Weather Station / IoT |
                    +-----------+-----------+
                                |
                                v
                    +-----------------------+
                    | Firebase RT Database  |
                    |  weather / subscribers|
                    +----+--------------+----+
                         |              |
                browser  |              | Python workers
                         v              v
              +----------------+   +------------------+
              | React + Vite   |   | Email / SMS      |
              | GeoSense UI    |   | Alert Workers    |
              +----------------+   +------------------+
```

Deploying only the frontend does not automatically start the Python alert workers. If notifications are required, those workers need their own runtime and credentials.

## Current Project Status

The repository represents a working dashboard codebase with live Firebase integration, historical charts, CSV export, multilingual UI, alert subscriptions, and separate notification workers.

At the same time, several parts are still development-oriented:

- Python dependencies are not locked in a requirements file.
- The ML notebook is primarily a preprocessing experiment, not a deployed prediction service.
- Automated tests are minimal.
- Firebase rules require review before production deployment.
- Live median calculation reads the complete weather history and may not scale indefinitely.
- Frontend and alert thresholds are maintained separately.
- The mock hardware server is not connected to the production data path.
- Firestore is initialized but not currently used by the main dashboard flow.

## Team
Our Team GeoSense roles for eLSI Wada Hackathon:

| Member | Role |
|---|---|
| Abdullah Ansari | Hardware Systems and Power Optimization Engineer |
| Hussain Attar | Electronics and Robotics Engineer |
| Mueez Hajwani | Data Analyst |
| Irfan Shaikh | Machine Learning Engineer |
| Idris Lokhande | IoT and Backend Developer |
| Zaid Pansare | UI/UX Designer |
| Danish Khan | Data Processing Engineer |
| Rehan Shaikh | Frontend Developer |


## License

A license file is not included in the reviewed repository snapshot. Add an explicit `LICENSE` file before distributing the project if a specific open-source or proprietary license is intended.

## Development Philosophy

This project combines IoT telemetry, environmental calculations, frontend visualization, Firebase data services, data cleaning, and notification automation in one learning-oriented system.

The main design goal is straightforward: collect sensor data, make the data understandable, surface abnormal conditions, and keep the architecture simple enough to inspect and extend.
