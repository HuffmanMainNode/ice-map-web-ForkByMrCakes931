
# Comprehensive Security Audit and Technical Enhancement Plan

This document synthesizes the findings of the initial security audit, the technical roadmap for real-time data integration, and the conceptual logic for automated data ingestion for the `ice-map-web` application.


### Conceptual Security Audit: `ice-map-web`

#### 1. Dependency Vulnerabilities
- **`package.json`**: Needs regular auditing (e.g., `npm audit`) to ensure packages like `react`, `mapbox-gl`, and `deck.gl` do not introduce known CVEs. Outdated mapping libraries can sometimes be exploited if they handle malformed GeoJSON insecurely.

#### 2. Configuration Issues
- **`vercel.json`**: Ensure proper security headers are set (e.g., Content-Security-Policy, X-Frame-Options, Strict-Transport-Security) to mitigate XSS and clickjacking attacks. The current configuration might be lacking these.

#### 3. Source Code Patterns (React Components)
- **Cross-Site Scripting (XSS)**: Review `src/components/` for any use of `dangerouslySetInnerHTML`. If external data (even from TRAC) is rendered without sanitization, it poses an XSS risk. Ensure all dynamic content is properly escaped by React.

#### 4. Secret Management
- **API Keys**: Ensure Mapbox or other service API keys are not hardcoded in the frontend source code (`src/`). They should be injected at build time via environment variables (e.g., `VITE_MAPBOX_TOKEN`) and restricted by domain/referrer in the service provider's dashboard.

#### 5. Data Integrity
- **`facilities.json`**: The current static JSON approach relies on the integrity of the build process. If an attacker compromises the build pipeline (e.g., via a compromised GitHub Action or dependency), they could inject malicious data. Implementing subresource integrity (SRI) or moving to a secured API backend mitigates this.



### Technical Roadmap: Integrating MCP Logic via Vercel Serverless Functions

This roadmap outlines the steps to bridge the `ice-locator-mcp` backend logic with the `ice-map-web` frontend using Vercel's serverless architecture.

#### Phase 1: Repository & Directory Setup
1. **Initialize API Directory**: Inside the `ice-map-web` repository, create an `api/` directory at the project root. Vercel automatically maps files in this directory to Serverless Functions.
2. **Configure Routing**: Update `vercel.json` if necessary to ensure proper routing and CORS headers for the `/api/*` endpoints.

#### Phase 2: Create Serverless Functions (MCP Wrappers)
1. **Setup MCP Client**: Install necessary dependencies (e.g., `axios` or native `fetch`) to communicate with the core MCP service or rewrite the core Python logic into Node.js/Python serverless functions if porting.
2. **Endpoint Creation**: Create specific serverless functions to wrap MCP capabilities:
   - `api/facilities.ts`: Fetch real-time facility data and population statistics.
   - `api/search-detainee.ts`: Endpoint to handle query parameters (name, A-number) and securely pass them to the MCP logic.

#### Phase 3: Security and Environment Variables
1. **Environment Configuration**: Define required API keys and secrets (e.g., `MCP_SERVICE_URL`, `MCP_API_KEY`) in the Vercel Dashboard under Project Settings -> Environment Variables.
2. **Secure Access**: Ensure the serverless functions access these keys via `process.env` (Node) or `os.environ` (Python) without exposing them to the client-side bundle.
3. **Rate Limiting**: Implement basic rate-limiting within the serverless functions to prevent abuse of the backend MCP service.

#### Phase 4: Frontend Integration & State Management
1. **Replace Static Imports**: Remove static JSON imports (`import data from './facilities.json'`) in React components.
2. **Implement Data Fetching Hooks**: Use a library like React Query or SWR to fetch data from the new `/api/facilities` endpoint:
   ```typescript
   const { data, error } = useSWR('/api/facilities', fetcher);
   ```
3. **Handle Loading/Error States**: Update UI components to gracefully handle loading states and potential API errors during real-time data retrieval.



### Conceptual Logic: `update_realtime_data.py`

This script will run on a scheduled basis (e.g., via GitHub Actions or a cron job) to ingest, validate, and update the facility data.

#### 1. Data Ingestion
- **Target Sources**: Connect to the identified real-time sources such as ICE FOIA feeds (CSV/API) or TRAC's latest updates.
- **Authentication**: If required, authenticate using stored environment variables (e.g., API tokens).
- **Data Fetching**: Use libraries like `requests` or `pandas` to fetch and parse the incoming data stream.

#### 2. Data Validation & Transformation
- **Schema Checking**: Validate the incoming data against the expected schema (e.g., ensuring fields like `facility_name`, `latitude`, `longitude`, and `population` are present and correctly typed).
- **Data Cleaning**: Handle missing values, normalize facility names, and convert coordinates to proper float types.
- **Difference Calculation**: Compare the fetched data with the existing dataset (from S3 or PostgreSQL) to identify updates, new facilities, or closed facilities.

#### 3. Update Storage (Phased Approach)
- **Phase 2 (S3/Blob Storage)**: If using JSON storage, update the `facilities.json` object with the new data, increment the `meta.t` timestamp, and upload the new JSON blob to the Vercel Blob or S3 bucket.
- **Phase 3 (PostgreSQL)**: If using a database, execute `UPSERT` operations to update existing facility records and insert new ones. Log the update timestamp in a metadata table.

#### 4. Trigger Frontend Revalidation
- **Webhook Call**: Send a POST request to a Vercel webhook (e.g., `/api/revalidate?secret=...`) to trigger an on-demand revalidation of the frontend cache, ensuring users see the latest data without a full site rebuild.

