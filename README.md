# Fleet AI Planner Pro - System Architecture Documentation

**Version:** 1.0.0  
**Last Updated:** December 2025  
**Status:** Production Prototype

---

## 1. Title & Overview

### System Name

**Fleet AI Planner Pro** (Internal codename: FCN)

### Purpose

A file-based fleet management and capital planning system designed for utility companies (Electric, Gas, Water Operations) to manage vehicle fleets, forecast replacement needs, and optimize capital expenditure.

### Target Users

- **Fleet Managers**: Day-to-day vehicle inventory management
- **Capital Planning Teams**: Multi-year budget forecasting and scenario planning
- **Finance Directors**: Budget oversight and cost analysis
- **Operations Managers**: Fleet utilization and lifecycle tracking
- **Sustainability Officers**: EV transition planning and compliance

### Problems Solved

1. **Manual Fleet Tracking**: Replaces spreadsheet-based fleet management with centralized web interface
2. **Capital Planning Complexity**: Automates multi-year forecasting with configurable parameters
3. **Data Accessibility**: Maintains Excel-based storage for easy audit and non-technical user access
4. **Compliance Tracking**: Automated out-of-lifecycle vehicle monitoring
5. **Cost Optimization**: TCO calculations and scenario comparison for replacement strategies

---

## 2. Architectural Summary

### Architecture Style

**Client-Server with File-Based Persistence**

This is a traditional web application architecture with a critical distinction: **data persistence uses Excel files instead of a relational database**. This design choice prioritizes:

- Auditability (Excel files can be opened and reviewed directly)
- Accessibility (non-technical users can inspect data)
- Portability (files can be easily backed up and shared)
- Simplicity (no database server management)

### Key Design Principles

1. **File-Based Data Layer**
   - Excel files (.xlsx) serve as the primary data store
   - JSON files provide performance caching for analytics
   - Pandas library handles all Excel I/O operations

2. **Deterministic Logic**
   - All calculations use rule-based algorithms
   - No machine learning or AI models (despite "AI" in the name)
   - Forecasting uses linear projections and configurable parameters

3. **Stateless API**
   - Flask REST API with JWT authentication
   - No server-side session storage
   - Each request is independent

4. **Client-Side State Management**
   - TanStack Query handles server state caching
   - React Context for authentication state
   - No global state management library (Redux, Zustand, etc.)

### Why This Architecture Was Chosen

**Excel-Based Storage:**

- Utility companies already use Excel extensively
- Easy data migration from existing spreadsheets
- Non-technical stakeholders can audit data directly
- No database licensing or infrastructure costs
- Simplified backup and version control

**Flask Backend:**

- Python's Pandas library excels at Excel manipulation
- Lightweight and easy to deploy
- Minimal infrastructure requirements
- Fast development iteration

**React Frontend:**

- Modern, responsive user interface
- Rich ecosystem for charts and data visualization
- TypeScript provides type safety for complex data structures

**Tradeoffs Accepted:**

- Limited concurrent write operations (Excel file locking)
- No ACID transaction guarantees
- Scalability ceiling (file I/O bottleneck)
- Manual data integrity enforcement

---

## 3. High-Level System Flow

### Request-Response Cycle

```
┌─────────┐      HTTPS/JSON      ┌─────────┐      Pandas      ┌──────────┐
│ Browser │ ◄──────────────────► │  Flask  │ ◄──────────────► │  Excel   │
│ (React) │                      │   API   │                  │  Files   │
└─────────┘                      └─────────┘                  └──────────┘
     │                                │                             │
     │                                │                             │
     ▼                                ▼                             ▼
TanStack Query                  Route Blueprints              .xlsx files
(Client Cache)                  Service Layer                 (Data Store)
```

### Data Flow Explanation

1. **User Interaction**: User interacts with React UI (clicks, form submissions, navigation)
2. **API Request**: Frontend makes HTTP request via Axios to Flask backend
3. **Authentication**: JWT token validated on protected routes
4. **Route Handling**: Flask Blueprint routes request to appropriate handler
5. **Service Invocation**: Route handler delegates to service layer
6. **Excel I/O**: Service uses Pandas to read/write Excel files
7. **Data Processing**: Business logic applied (calculations, aggregations, filtering)
8. **Response Formation**: Data serialized to JSON
9. **Client Update**: TanStack Query caches response and updates React components
10. **UI Render**: React re-renders with new data

### Caching Strategy

- **Client-Side**: TanStack Query caches API responses (configurable TTL)
- **Server-Side**: JSON files cache expensive analytics calculations
- **Cache Invalidation**: Manual invalidation on data mutations

---

## 4. Layer-by-Layer Breakdown

### 4.1 User Interaction Layer

**Browser Requirements:**

- Modern browsers (Chrome 90+, Firefox 88+, Edge 90+, Safari 14+)
- JavaScript enabled
- Minimum 1280x720 resolution recommended

**User Roles:**

- **Admin**: Full system access, user management, configuration
- **Manager**: Fleet management, planning, reporting
- **Analyst**: Read-only access, custom reports, pivot tables
- **Viewer**: Dashboard and basic reports only

**Authentication:**

- JWT tokens stored in localStorage
- Token expiration: 24 hours (configurable)
- No automatic refresh (user must re-login)

---

### 4.2 Frontend Layer

**Technology Stack:**

- **React**: 18.3.1 (functional components, hooks)
- **TypeScript**: 5.x (strict mode enabled)
- **Vite**: 5.x (build tool, dev server)
- **TanStack Query**: v4 (server state management)
- **React Router**: v6 (client-side routing)
- **Axios**: HTTP client with interceptors
- **Recharts**: Chart library for data visualization
- **Tailwind CSS**: Utility-first styling
- **Shadcn/UI**: Component library (built on Radix UI)

**Project Structure:**

```
fleet-front/
├── src/
│   ├── components/        # Reusable UI components
│   ├── pages/            # Route-level components
│   ├── services/         # API client functions
│   ├── hooks/            # Custom React hooks
│   ├── context/          # React Context providers
│   ├── types/            # TypeScript type definitions
│   └── utils/            # Helper functions
```

**Key Modules:**

1. **Dashboard** (`/dashboard`)
   - Fleet summary cards (total vehicles, out-of-lifecycle count, etc.)
   - Trend charts (cost over time, fleet composition)
   - Quick actions (add vehicle, run forecast)

2. **Fleet Manager** (`/fleet`)
   - Vehicle inventory table with search/filter
   - Vehicle detail view with edit capabilities
   - Bulk import/export via Excel

3. **Pivot Grid** (`/pivot`)
   - Dynamic pivot table builder
   - Drag-and-drop dimensions and measures
   - Custom aggregations (sum, avg, count, etc.)

4. **Planner** (`/planning`)
   - Multi-year forecast configuration
   - Scenario comparison (baseline vs. alternatives)
   - Budget allocation interface

5. **Admin** (`/admin`)
   - User management (CRUD operations)
   - Cost parameter configuration
   - System logs viewer

**State Management:**

- **AuthContext**: User authentication state, token storage
- **TanStack Query**: All server data (vehicles, forecasts, analytics)
- **Local State**: UI-specific state (modal visibility, form inputs)
- **URL State**: Route parameters, query strings

**API Integration:**

All API calls go through centralized service functions:

```typescript
// Example: services/fleetService.ts
export const getVehicles = () => 
  axios.get('/api/fleet/vehicles');

export const createVehicle = (data) => 
  axios.post('/api/fleet/vehicles', data);
```

Axios interceptors handle:

- JWT token attachment
- Error handling
- Request/response logging (dev mode)

---

### 4.3 Backend Layer (Flask)

**Technology Stack:**

- **Flask**: 2.3.3 (web framework)
- **Flask-CORS**: Cross-origin resource sharing
- **Flask-JWT-Extended**: JWT authentication
- **Pandas**: 2.x (Excel I/O and data manipulation)
- **NumPy**: 1.24+ (numerical computations)
- **OpenPyXL**: Excel file reading/writing
- **python-dotenv**: Environment variable management

**Application Factory Pattern:**

```python
# app.py
app = Flask(__name__)
app.config.from_object(config[config_name])
CORS(app, resources={r"/*": {"origins": "*"}})
```

**Blueprint Registration:**

The application registers 16 route blueprints:

| Blueprint | URL Prefix | Purpose |
|-----------|------------|---------|
| `health_bp` | `/health` | Health check endpoint |
| `auth_bp` | `/api/auth` | User authentication |
| `fleet_bp` | `/api/fleet` | Vehicle CRUD operations |
| `analytics_bp` | `/api/analytics` | Dashboard statistics |
| `equipment_bp` | `/api/equipment` | Equipment lifecycle data |
| `excel_data_bp` | `/api/excel` | Generic Excel operations |
| `cost_parameters_bp` | `/api/cost` | Cost configuration |
| `dynamic_pivot_bp` | `/api/pivot` | Pivot table generation |
| `reports_bp` | `/api/reports` | Report generation |
| `planning_bp` | `/api/planning` | Capital planning |
| `file_management_bp` | `/api/files` | File upload/download |
| `logs_bp` | `/api/logs` | System logs |
| `ool_bp` | `/api/ool` | Out-of-lifecycle tracking |
| `ool_audit_bp` | `/api/ool/audit` | OOL audit logs |
| `fleet_time_bp` | `/api/fleet-time` | Time-series data |
| `radio_cost_bp` | `/api/radio-cost` | Radio equipment costs |

**Request Lifecycle:**

1. **Request Reception**: Flask receives HTTP request
2. **CORS Preflight**: OPTIONS requests handled automatically
3. **JWT Validation**: Protected routes check Authorization header
4. **Route Matching**: Request routed to appropriate Blueprint
5. **Handler Execution**: Route handler function invoked
6. **Service Call**: Handler delegates to service layer
7. **Response Formation**: Data serialized to JSON
8. **Response Return**: HTTP response sent to client

**Error Handling:**

- **400 Bad Request**: Invalid input data
- **401 Unauthorized**: Missing or invalid JWT token
- **404 Not Found**: Resource doesn't exist
- **500 Internal Server Error**: Unhandled exceptions

All errors return JSON:

```json
{
  "error": "Error message",
  "status": 400
}
```

---

### 4.4 Service Layer

The service layer contains all business logic and data access. Services are **not** dependency-injected; they are imported and called directly.

**Service Categories:**

#### Core Data Services

1. **ExcelDataService** (`excel_data_service.py`)
   - **Purpose**: Unified interface for all Excel file operations
   - **Key Methods**:
     - `read_excel(file_path, sheet_name)`: Read Excel sheet into DataFrame
     - `write_excel(df, file_path, sheet_name)`: Write DataFrame to Excel
     - `append_row(file_path, sheet_name, row_data)`: Append single row
     - `update_row(file_path, sheet_name, row_id, updates)`: Update existing row
     - `delete_row(file_path, sheet_name, row_id)`: Delete row
   - **Caching**: Implements in-memory cache for frequently accessed files
   - **Locking**: Uses file-based locking to prevent concurrent writes

2. **FileManagementService** (`file_management_service.py`)
   - **Purpose**: File upload, validation, and storage
   - **Capabilities**:
     - Excel file validation (format, schema)
     - CSV to Excel conversion
     - File backup before overwrite
     - Virus scanning (placeholder, not implemented)

3. **LogsService** (`logs_service.py`)
   - **Purpose**: System event logging
   - **Log Types**:
     - User actions (login, data changes)
     - System events (errors, warnings)
     - Audit trail (who changed what, when)
   - **Storage**: Logs written to Excel file (`logs.xlsx`)

#### Domain Services

1. **EquipmentService** (`equipment_service.py`)
   - **Purpose**: Equipment lifecycle management
   - **Data Source**: `equipment_lifecycle.xlsx`
   - **Operations**:
     - Get lifecycle thresholds by equipment type
     - Calculate depreciation
     - Determine replacement eligibility

2. **CostParametersService** (`cost_parameters_service.py`)
   - **Purpose**: Economic parameter management
   - **Data Source**: `cost_parameters.xlsx`
   - **Parameters**:
     - Inflation rate (annual percentage)
     - Fuel costs (per gallon/kWh)
     - Maintenance rates (per mile/year)
     - Procurement costs (by vehicle type)

3. **FleetTimeService** (`fleet_time_service.py`)
   - **Purpose**: Time-series fleet data
   - **Data Source**: `fleet_time_data.xlsx`
   - **Capabilities**:
     - Utilization tracking over time
     - Downtime analysis
     - Temporal pattern detection

4. **OOLAuditService** (`ool_audit_service.py`)
   - **Purpose**: Out-of-lifecycle compliance tracking
   - **Data Source**: `ool_audit_logs.xlsx`
   - **Functions**:
     - Log vehicles exceeding lifecycle limits
     - Generate compliance reports
     - Alert generation (email placeholders)

#### Computation Services

1. **AnalyticsService** (`analytics_service.py`)
   - **Purpose**: Dashboard statistics and KPI calculations
   - **Key Metrics**:
     - Total fleet count
     - Out-of-lifecycle percentage
     - Average vehicle age
     - Total fleet value
     - Cost per mile/year
   - **Caching**: Results cached in `ANALYTICS_SUMMARY.json`
   - **Cache Invalidation**: Manual invalidation on data changes

2. **ScenarioService** (`scenario_service.py`)
   - **Purpose**: Multi-year forecasting
   - **Algorithm**: Linear projection with configurable growth rates
   - **Inputs**:
     - Current fleet composition
     - Replacement schedule
     - Cost parameters (inflation, fuel, maintenance)
     - Budget constraints
   - **Outputs**:
     - Year-by-year fleet composition
     - Annual costs (acquisition, maintenance, fuel)
     - Budget variance
   - **Note**: This is **deterministic calculation**, not AI/ML

3. **DynamicPivotService** (`dynamic_pivot_service.py`)
    - **Purpose**: Custom pivot table generation
    - **Capabilities**:
      - Multi-dimensional grouping
      - Aggregation functions (sum, avg, count, min, max)
      - Filtering and sorting
    - **Implementation**: Uses Pandas `pivot_table()` function

4. **ReportsService** (`reports_service.py`)
    - **Purpose**: Report generation
    - **Report Types**:
      - Fleet summary (PDF/Excel)
      - Cost analysis (Excel)
      - Lifecycle status (PDF)
      - Out-of-lifecycle (Excel)
    - **Generation**: Uses Pandas for Excel, ReportLab for PDF (future)

**Service Interaction:**

Services call each other directly (no dependency injection):

```python
# Example: AnalyticsService calls ExcelDataService
from services.excel_data_service import ExcelDataService

class AnalyticsService:
    def get_fleet_stats(self):
        excel_service = ExcelDataService()
        df = excel_service.read_excel('vehicle_fleet_master.xlsx', 'Fleet')
        # ... calculations ...
```

---

### 4.5 Processing / Computation Layer

**Forecasting Logic:**

The forecasting engine uses **deterministic, rule-based calculations**:

1. **Baseline Projection**:

   ```
   Future Fleet Size = Current Size × (1 + Growth Rate)^Years
   ```

2. **Replacement Logic**:
   - Vehicles reaching age/mileage threshold are flagged
   - Replacement cost = Procurement Cost × (1 + Inflation)^Years
   - Replacement scheduled based on budget availability

3. **Cost Modeling**:

   ```
   Total Annual Cost = Acquisition + Maintenance + Fuel + Depreciation
   
   Maintenance Cost = Base Rate × Miles × (1 + Age Factor)
   Fuel Cost = Miles / MPG × Fuel Price
   Depreciation = (Purchase Price - Salvage Value) / Useful Life
   ```

4. **EV Transition Logic**:
   - User specifies EV adoption percentage by year
   - System calculates infrastructure costs (chargers)
   - Compares TCO: Traditional vs. EV
   - **No AI/ML**: Simple cost comparison

**Scenario Comparison:**

Users can create multiple scenarios with different parameters:

- Scenario A: 5% annual growth, 10% EV by 2030
- Scenario B: 3% annual growth, 25% EV by 2030

System calculates each scenario independently and presents side-by-side comparison.

**Performance Optimization:**

- **Caching**: Expensive calculations cached in JSON
- **Lazy Loading**: Data loaded on-demand, not upfront
- **Chunking**: Large Excel files processed in chunks (Pandas `chunksize`)

---

### 4.6 Data Layer

**Excel File Structure:**

All data stored in Excel files located in `backend/database/` directory:

| File Name | Purpose | Key Columns | Approximate Size |
|-----------|---------|-------------|------------------|
| `users.xlsx` | User accounts | username, password_hash, role, email | < 1 MB |
| `vehicle_fleet_master.xlsx` | Fleet inventory | vehicle_id, make, model, year, vin, dept, cost, mileage, status | 1-5 MB |
| `equipment_lifecycle.xlsx` | Lifecycle rules | equipment_type, max_age, max_mileage, replacement_cost | < 1 MB |
| `cost_parameters.xlsx` | Economic variables | parameter_name, value, effective_date | < 1 MB |
| `fleet_time_data.xlsx` | Time-series data | vehicle_id, date, mileage, utilization | 1-10 MB |
| `ool_audit_logs.xlsx` | Compliance logs | vehicle_id, ool_date, reason, action_taken | < 1 MB |
| `radio_equipment_cost.xlsx` | Radio costs | equipment_id, cost, install_date | < 1 MB |
| `ev_budget_analysis.xlsx` | EV analysis | scenario_id, year, ev_count, cost | < 1 MB |

**JSON Cache Files:**

| File Name | Purpose | Invalidation |
|-----------|---------|--------------|
| `ANALYTICS_SUMMARY.json` | Dashboard KPIs | On fleet data change |

**Read/Write Behavior:**

**Read Operations:**

- Pandas `read_excel()` loads entire sheet into DataFrame
- In-memory caching reduces repeated file reads
- No row-level locking (entire file read)

**Write Operations:**

- Pandas `to_excel()` writes entire DataFrame to file
- File-based locking prevents concurrent writes
- Backup created before overwrite (`.bak` file)
- **Risk**: Concurrent writes can corrupt file

**Data Integrity:**

- **No ACID Guarantees**: Excel files don't support transactions
- **Manual Validation**: Services validate data before write
- **Backup Strategy**: Daily file backups (external script)
- **Corruption Recovery**: Restore from `.bak` file

**Scalability Limits:**

- **File Size**: Performance degrades beyond 10 MB per file
- **Concurrent Users**: Limited to ~5-10 simultaneous users
- **Write Throughput**: ~1-2 writes/second per file

---

### 4.7 Runtime Environment

**Backend Runtime:**

- **Python**: 3.10+ (CPython interpreter)
- **Process Model**: Single-threaded Flask development server (production: Gunicorn with 4 workers)
- **Memory**: ~200-500 MB per worker process
- **CPU**: Pandas operations are CPU-intensive (NumPy uses BLAS)

**Frontend Runtime:**

- **Node.js**: 18+ (development only)
- **Build Output**: Static HTML/CSS/JS files
- **Deployment**: Served via Nginx or similar web server

**File System Dependency:**

The system **requires** direct file system access:

- Excel files must be on local or network-attached storage
- No cloud object storage (S3, Azure Blob) support
- File paths hardcoded in configuration

**Execution Model:**

- **Synchronous**: All operations are synchronous (blocking I/O)
- **No Background Jobs**: No Celery, RQ, or task queues
- **No WebSockets**: No real-time updates (polling only)

---

## 5. Architecture Diagram Explanation

### Diagram Walkthrough

*Refer to the provided architecture diagram image.*

**Top Layer: User Interaction**

- **User**: End user accessing the system
- **Browser**: Web browser (Chrome, Firefox, Edge, Safari)
- **Connection**: HTTPS to frontend (Vite dev server or Nginx)

**Second Layer: Frontend (React + Vite)**

- **UI Core**: Main application shell (`App.tsx`)
- **Auth Components**: Login, registration, password reset
- **Dashboard Components**: KPI cards, charts, quick actions
- **Scenario Builder**: Forecast configuration interface
- **API Client**: Axios instance with interceptors
- **State Management**: TanStack Query for server state
- **Flow**: User interaction → Component → TanStack Query → API Client → HTTP request

**Third Layer: Backend (Flask API)**

- **Flask App Entry**: Application factory (`app.py`)
- **Controllers & Routes**: 16 Blueprint modules
  - `/api/auth`: Authentication
  - `/api/fleet`: Vehicle operations
  - `/api/analytics`: Dashboard stats
  - `/api/reports`: Report generation
  - `/api/planning`: Forecasting
  - `/api/cost`: Cost parameters
  - `/api/pivot`: Pivot tables
  - `/api/files`: File management
  - `/api/ool`: Out-of-lifecycle tracking
- **Services**: Business logic layer
  - **Auth Service**: JWT validation
  - **Scenario Service**: Forecasting logic
  - **Analytics Service**: KPI calculations
  - **Excel Service**: File I/O operations
- **Flow**: HTTP request → Blueprint → Service → Excel I/O → Response

**Bottom Layer: File System Persistence**

- **Master Data Files**:
  - `users.xlsx`: User accounts
  - `vehicle_fleet_master.xlsx`: Fleet inventory
  - `cost_parameters.xlsx`: Economic parameters
- **Cache Files**:
  - `ANALYTICS_SUMMARY.json`: Dashboard cache
- **Flow**: Service → Pandas → Excel file read/write

**Data Flow:**

1. User clicks "View Dashboard" in browser
2. React component triggers API call via TanStack Query
3. Axios sends GET request to `/api/analytics/summary`
4. Flask routes to `analytics_bp` Blueprint
5. Blueprint calls `AnalyticsService.get_summary()`
6. Service calls `ExcelDataService.read_excel('vehicle_fleet_master.xlsx')`
7. Pandas reads Excel file into DataFrame
8. Service calculates KPIs (count, avg age, etc.)
9. Service caches result in `ANALYTICS_SUMMARY.json`
10. Service returns JSON to Blueprint
11. Blueprint returns HTTP 200 with JSON body
12. TanStack Query caches response
13. React component re-renders with data

---

## 6. Data Flow Example (Step-by-Step)

### Example: User Opens Fleet Dashboard

**Step 1: User Navigation**

- User clicks "Dashboard" in navigation menu
- React Router navigates to `/dashboard` route

**Step 2: Component Mount**

- `Dashboard.tsx` component mounts
- `useQuery` hook triggers data fetch

**Step 3: API Request**

```typescript
// Frontend: services/analyticsService.ts
export const getDashboardStats = () => 
  axios.get('/api/analytics/summary');

// Component: pages/Dashboard.tsx
const { data, isLoading } = useQuery({
  queryKey: ['dashboard-stats'],
  queryFn: getDashboardStats
});
```

**Step 4: HTTP Request**

- Axios sends: `GET http://localhost:5000/api/analytics/summary`
- Headers include: `Authorization: Bearer <JWT_TOKEN>`

**Step 5: Backend Route Handling**

```python
# Backend: routes/analytics_routes.py
@analytics_bp.route('/api/analytics/summary', methods=['GET'])
@jwt_required()
def get_summary():
    service = AnalyticsService()
    stats = service.get_summary()
    return jsonify(stats), 200
```

**Step 6: Service Execution**

```python
# Backend: services/analytics_service.py
class AnalyticsService:
    def get_summary(self):
        # Check cache first
        if os.path.exists('ANALYTICS_SUMMARY.json'):
            with open('ANALYTICS_SUMMARY.json') as f:
                return json.load(f)
        
        # Cache miss: calculate from Excel
        excel_service = ExcelDataService()
        df = excel_service.read_excel('vehicle_fleet_master.xlsx', 'Fleet')
        
        stats = {
            'total_vehicles': len(df),
            'avg_age': df['age'].mean(),
            'ool_count': len(df[df['status'] == 'Out of Lifecycle']),
            'total_value': df['cost'].sum()
        }
        
        # Cache result
        with open('ANALYTICS_SUMMARY.json', 'w') as f:
            json.dump(stats, f)
        
        return stats
```

**Step 7: Excel Read**

```python
# Backend: services/excel_data_service.py
import pandas as pd

class ExcelDataService:
    def read_excel(self, file_path, sheet_name):
        full_path = os.path.join('database', file_path)
        df = pd.read_excel(full_path, sheet_name=sheet_name)
        return df
```

**Step 8: Response Formation**

- Service returns Python dict
- Flask `jsonify()` converts to JSON
- HTTP response: `200 OK` with JSON body

**Step 9: Client Update**

- Axios receives response
- TanStack Query caches data with key `['dashboard-stats']`
- Component state updated

**Step 10: UI Render**

```typescript
// Frontend: pages/Dashboard.tsx
return (
  <div>
    {isLoading ? <Spinner /> : (
      <>
        <StatCard title="Total Vehicles" value={data.total_vehicles} />
        <StatCard title="Average Age" value={data.avg_age} />
        <StatCard title="Out of Lifecycle" value={data.ool_count} />
      </>
    )}
  </div>
);
```

**Total Time:** ~200-500ms (depending on cache hit/miss)

---

## 7. Design Decisions & Tradeoffs

### Decision 1: Excel Instead of Database

**Rationale:**

- Utility companies already use Excel for fleet tracking
- Non-technical users can audit data directly
- Easy migration from existing spreadsheets
- No database server infrastructure required
- Simplified backup (just copy files)

**Tradeoffs:**

- ❌ Limited concurrent write operations
- ❌ No ACID transaction guarantees
- ❌ File corruption risk
- ❌ Scalability ceiling (~10 MB files, ~10 concurrent users)
- ✅ Easy data inspection and audit
- ✅ No database licensing costs
- ✅ Portable (files can be moved between servers)

### Decision 2: Deterministic Logic (No AI/ML)

**Rationale:**

- Forecasting requirements are well-defined (linear projections)
- No training data available for ML models
- Deterministic results are easier to explain to stakeholders
- Simpler to debug and maintain

**Tradeoffs:**

- ❌ No adaptive learning from historical patterns
- ❌ Limited to rule-based scenarios
- ✅ Predictable, explainable results
- ✅ No model training or retraining required
- ✅ Simpler codebase

### Decision 3: TanStack Query (No Redux)

**Rationale:**

- Most state is server-derived (vehicles, forecasts, analytics)
- TanStack Query handles caching, refetching, and synchronization
- Less boilerplate than Redux

**Tradeoffs:**

- ❌ Limited global state management
- ❌ Complex client-side state requires workarounds
- ✅ Automatic cache invalidation
- ✅ Built-in loading and error states
- ✅ Less code to maintain

### Decision 4: Synchronous API (No Background Jobs)

**Rationale:**

- Forecast calculations complete in <5 seconds
- No long-running tasks requiring background processing
- Simpler deployment (no Celery/RQ infrastructure)

**Tradeoffs:**

- ❌ Long requests can timeout
- ❌ No progress tracking for slow operations
- ✅ Simpler architecture
- ✅ Easier to debug

---

## 8. Current Limitations

### Scalability Limits

1. **Concurrent Users**: ~5-10 simultaneous users before performance degrades
2. **File Size**: Excel files >10 MB cause slow reads/writes
3. **Write Throughput**: ~1-2 writes/second per file (file locking bottleneck)
4. **Memory**: Each Pandas DataFrame loads entire file into memory

### Concurrency Limits

1. **No Row-Level Locking**: Entire Excel file locked during write
2. **Race Conditions**: Concurrent writes can corrupt files
3. **No Transaction Support**: Partial writes can leave data inconsistent

### Data Integrity Risks

1. **File Corruption**: Power loss during write can corrupt Excel file
2. **Manual Validation**: No database constraints (foreign keys, unique constraints)
3. **Backup Dependency**: Recovery requires manual file restore

### Functional Limitations

1. **No Real-Time Updates**: UI requires manual refresh or polling
2. **No Audit Trail**: Limited change tracking (only in logs.xlsx)
3. **No Multi-Tenancy**: Single organization only
4. **No API Versioning**: Breaking changes affect all clients

---

## 9. Future Evolution

### If Database Is Introduced

**Migration Path:**

1. **Phase 1**: Dual-write (Excel + PostgreSQL) for validation
2. **Phase 2**: Read from PostgreSQL, write to both
3. **Phase 3**: Deprecate Excel writes, keep as backup
4. **Phase 4**: Full PostgreSQL migration

**Benefits:**

- ACID transactions
- Row-level locking
- Better concurrency
- Referential integrity
- Query optimization

**Changes Required:**

- Replace `ExcelDataService` with `DatabaseService`
- Add ORM (SQLAlchemy)
- Implement database migrations (Alembic)
- Update all service layer calls

### If AI/ML Is Added

**Potential Use Cases:**

1. **Predictive Maintenance**: ML model predicts vehicle failures
2. **Anomaly Detection**: Identify unusual cost patterns
3. **Demand Forecasting**: ML-based fleet size predictions

**Changes Required:**

- Add ML framework (scikit-learn, TensorFlow)
- Implement model training pipeline
- Add model versioning and deployment
- Update forecasting service to use ML predictions

**Note:** Current "AI" branding is marketing; no ML models exist today.

---

## 10. Summary

### System Characteristics

**Fleet AI Planner Pro** is a **file-based fleet management system** with:

- **React + TypeScript** frontend for modern UI
- **Flask + Python** backend for business logic
- **Excel files** for data persistence (not a database)
- **Deterministic algorithms** for forecasting (not AI/ML)
- **JWT authentication** for security
- **TanStack Query** for client-side caching

### Key Strengths

1. **Accessibility**: Excel files can be opened and audited by anyone
2. **Simplicity**: No database server or complex infrastructure
3. **Portability**: Easy to backup, migrate, and deploy
4. **Auditability**: Clear data lineage and change tracking
5. **Cost-Effective**: No database licensing fees

### Key Weaknesses

1. **Scalability**: Limited to ~10 concurrent users, ~10 MB files
2. **Concurrency**: File locking prevents simultaneous writes
3. **Data Integrity**: No ACID guarantees, corruption risk
4. **Real-Time**: No WebSockets or live updates

### Ideal Use Case

This architecture is **ideal for**:

- Small to medium utility companies (50-500 vehicles)
- Organizations with existing Excel-based workflows
- Environments requiring easy data audit
- Deployments with limited IT infrastructure

This architecture is **not suitable for**:

- Large-scale deployments (>1000 vehicles)
- High-concurrency environments (>20 simultaneous users)
- Mission-critical systems requiring ACID guarantees
- Real-time data synchronization requirements

---

**Document Version:** 1.0.0  
**Last Updated:** December 2025  
**Maintained By:** Development Team  
**Review Cycle:** Quarterly
