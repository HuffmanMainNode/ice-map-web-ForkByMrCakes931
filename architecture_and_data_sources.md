# Architecture and Data Sources

## Identified Real-Time Data Sources
1. ACLU National ICE Detention Tracking GitHub repository
2. AILA ICE Detention Resources
3. Freedom for Immigrants map data

## Proposed Relational Database Schema
### 1. `Facilities` Table (Static Location Data)
- `id` (Primary Key, UUID)
- `name` (String)
- `address` (String)
- `latitude` (Float)
- `longitude` (Float)
- `facility_type` (String)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

### 2. `PopulationLogs` Table (Time-Series Population Counts)
- `id` (Primary Key, UUID)
- `facility_id` (Foreign Key -> Facilities.id)
- `population_count` (Integer)
- `report_date` (Date)
- `source` (String, e.g., 'TRAC', 'FOIA')
- `created_at` (Timestamp)

### 3. `SpotReports` Table (Civilian-Sourced Events)
- `id` (Primary Key, UUID)
- `latitude` (Float)
- `longitude` (Float)
- `location_description` (String)
- `report_time` (Timestamp)
- `event_type` (String, e.g., 'Checkpoint', 'Raid', 'Transport')
- `description` (Text)
- `verification_status` (String, e.g., 'Unverified', 'Verified')
- `created_at` (Timestamp)

### 4. `Deployments` Table (Active ICE Operations)
- `id` (Primary Key, UUID)
- `operation_name` (String)
- `target_area` (String)
- `start_date` (Timestamp)
- `end_date` (Timestamp, nullable)
- `status` (String, e.g., 'Active', 'Concluded')
- `source` (String)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)
