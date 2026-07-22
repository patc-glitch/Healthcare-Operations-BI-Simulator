# 07 — Generator Architecture

**Version:** 1.0
**Status:** Draft — Ready for Implementation Planning
**Last Updated:** 2026-07-22

---

# 1. Purpose

This document defines the architecture of the Python-based data generation and simulation engine for the **Healthcare Operations BI Simulator**.

The generator translates the logical model defined in:

* `01_Project_Overview.md`
* `02_Business_Model.md`
* `03_Business_Rules.md`
* `04_Entity_Dictionary.md`
* `05_Process_Flows.md`
* `06_ERD.md`

into a reproducible synthetic healthcare operations dataset.

The generator is responsible for simulating:

* healthcare demand,
* patient behavior,
* referrals,
* internal and external care pathways,
* waiting lists,
* doctor schedules,
* room availability,
* operational capacity,
* NFZ-funded capacity,
* effective capacity,
* appointment scheduling,
* service delivery,
* revenue,
* doctor compensation,
* operating costs,
* resource reallocations,
* management scenarios.

The generator must produce data that is:

```text
Logically consistent
+
Referentially valid
+
Temporally consistent
+
Business-rule compliant
+
Reproducible
+
Analytically useful
```

---

# 2. Architecture Principle

The generator is not a simple random data generator.

It is a:

> **Rule-driven, time-aware, dependency-controlled healthcare operations simulation engine.**

The generation process follows:

```text
Configuration
      ↓
Reference Data
      ↓
Organization
      ↓
Workforce
      ↓
Resources
      ↓
Schedules
      ↓
Contracts
      ↓
Capacity
      ↓
Demand
      ↓
Referrals
      ↓
Patient Decisions
      ↓
Waiting Lists
      ↓
Appointments
      ↓
Service Delivery
      ↓
Financial Transactions
      ↓
Scenario Changes
      ↓
KPIs
      ↓
Validation
      ↓
Export
```

The generator must never create downstream transactional records before their required parent entities exist.

---

# 3. Design Goals

The generator must satisfy the following goals.

## 3.1 Reproducibility

The same configuration and random seed must produce the same dataset.

```text
Seed
+
Configuration
+
Simulation Period
=
Deterministic Output
```

Example:

```python
seed = 42
```

Running the generator twice with the same seed must produce identical results.

---

## 3.2 Referential Integrity

Every foreign key must reference an existing valid entity.

Example:

```text
APPOINTMENT.doctor_id
        ↓
DOCTOR.doctor_id
```

Invalid references are prohibited.

---

## 3.3 Temporal Integrity

Events must respect chronological order.

For example:

```text
Demand Date
    ≤
Referral Date
    ≤
Waiting List Entry Date
    ≤
Appointment Date
    ≤
Service Delivery Date
```

A service cannot occur before its appointment.

An appointment cannot occur before the referral that created it.

---

## 3.4 Business Rule Compliance

The generator must implement the rules defined in `03_Business_Rules.md`.

Randomness must operate inside business constraints.

The generator should not generate:

```text
Random data
+
Post-hoc corrections
```

Instead:

```text
Business Constraints
        ↓
Valid State Space
        ↓
Controlled Randomness
        ↓
Generated Data
```

---

## 3.5 Analytical Realism

The dataset should exhibit realistic operational patterns.

Examples:

```text
Demand peaks
Seasonality
Different specialty demand
Doctor availability differences
Room bottlenecks
Waiting list growth
Referral leakage
NFZ utilization differences
Revenue variability
Cost variability
```

The objective is not statistical perfection.

The objective is:

> A realistic dataset that supports meaningful BI analysis.

---

# 4. Simulation Architecture

The generator is organized into the following layers.

```text
┌───────────────────────────────┐
│ Configuration Layer            │
├───────────────────────────────┤
│ Reference Data Layer           │
├───────────────────────────────┤
│ Master Data Layer              │
├───────────────────────────────┤
│ Resource & Capacity Layer      │
├───────────────────────────────┤
│ Demand & Patient Layer         │
├───────────────────────────────┤
│ Operational Simulation Layer   │
├───────────────────────────────┤
│ Financial Layer                │
├───────────────────────────────┤
│ Scenario Layer                 │
├───────────────────────────────┤
│ KPI & Analytical Layer         │
├───────────────────────────────┤
│ Validation Layer               │
├───────────────────────────────┤
│ Export Layer                   │
└───────────────────────────────┘
```

---

# 5. Generator Execution Pipeline

The complete execution pipeline is:

```text
STEP 01
Load Configuration
        ↓
STEP 02
Initialize Random Seed
        ↓
STEP 03
Generate Reference Data
        ↓
STEP 04
Generate Clinics
        ↓
STEP 05
Generate Specialties
        ↓
STEP 06
Generate Clinic-Specialty Relationships
        ↓
STEP 07
Generate Doctors
        ↓
STEP 08
Generate Doctor-Specialty Relationships
        ↓
STEP 09
Generate Rooms
        ↓
STEP 10
Generate Service Types
        ↓
STEP 11
Generate Doctor Schedules
        ↓
STEP 12
Generate NFZ Contracts
        ↓
STEP 13
Generate NFZ Contract Lines
        ↓
STEP 14
Generate Demand Events
        ↓
STEP 15
Generate Referrals
        ↓
STEP 16
Simulate Patient Decisions
        ↓
STEP 17
Generate Waiting List Entries
        ↓
STEP 18
Calculate Operational Capacity
        ↓
STEP 19
Calculate NFZ Funded Capacity
        ↓
STEP 20
Calculate Effective Capacity
        ↓
STEP 21
Schedule Appointments
        ↓
STEP 22
Generate Service Delivery
        ↓
STEP 23
Generate Revenue
        ↓
STEP 24
Generate Doctor Compensation
        ↓
STEP 25
Generate Operating Costs
        ↓
STEP 26
Apply Resource Reallocations
        ↓
STEP 27
Run Scenario Simulations
        ↓
STEP 28
Calculate KPIs
        ↓
STEP 29
Run Validation
        ↓
STEP 30
Export Dataset
```

The actual implementation may optimize this order where necessary, but dependencies must remain logically valid.

---

# 6. Configuration Layer

All major simulation assumptions must be configurable.

Configuration should include:

```text
simulation_start_date
simulation_end_date
random_seed
number_of_patients
number_of_doctors
number_of_clinics
number_of_rooms
demand_volume
specialty_distribution
service_type_distribution
referral_probability
internal_capture_probability
external_leakage_probability
patient_drop_probability
doctor_schedule_rules
room_availability_rules
service_duration_rules
revenue_rules
compensation_rules
operating_cost_rules
NFZ_contract_rules
scenario_rules
```

Configuration must not be hardcoded throughout the generator.

---

# 7. Random Seed Management

The generator uses a centralized random seed.

Example:

```python
RANDOM_SEED = 42
```

The generator should use a controlled random number generator.

Recommended pattern:

```python
rng = numpy.random.default_rng(RANDOM_SEED)
```

Randomness should be derived from the centralized generator.

The code should avoid uncontrolled use of:

```python
random.random()
```

throughout unrelated modules.

---

# 8. Master Data Generation

Master data is generated before transactional data.

The generation order is:

```text
CLINIC
    ↓
SPECIALTY
    ↓
CLINIC_SPECIALTY
    ↓
DOCTOR
    ↓
DOCTOR_SPECIALTY
    ↓
ROOM
    ↓
SERVICE_TYPE
```

---

# 9. Clinic Generation

Each clinic receives:

```text
clinic_id
clinic_name
clinic_type
location
status
```

Clinic IDs must be unique.

Example:

```text
CL001
CL002
CL003
```

The generator must support multiple clinics.

---

# 10. Specialty Generation

Specialties are generated from configuration.

Example:

```text
Cardiology
Dermatology
Pediatrics
Orthopedics
Neurology
```

Each specialty receives:

```text
specialty_id
specialty_name
status
```

---

# 11. Clinic-Specialty Assignment

The generator creates valid `CLINIC_SPECIALTY` relationships.

Rules:

```text
A clinic may offer multiple specialties.
A specialty may be offered by multiple clinics.
```

The assignment should allow realistic variation.

Example:

```text
Clinic A
    Cardiology
    Dermatology
    Pediatrics

Clinic B
    Cardiology
    Orthopedics
```

The assignment should support referral capture analysis.

---

# 12. Doctor Generation

Each doctor receives:

```text
doctor_id
doctor_name
employment_type
compensation_model
status
```

Doctors are assigned to clinics.

The generator should support:

```text
FULL_TIME
PART_TIME
CONTRACTOR
```

where applicable.

---

# 13. Doctor Specialty Assignment

Doctors are assigned to one or more specialties.

Rules:

```text
DOCTOR
    ↓
DOCTOR_SPECIALTY
    ↓
SPECIALTY
```

A doctor may only deliver services for an assigned specialty.

---

# 14. Room Generation

Rooms are generated per clinic.

Each room receives:

```text
room_id
clinic_id
room_name
room_type
status
```

Example:

```text
CL001-R01
CL001-R02
CL001-R03
```

Room types may include:

```text
CONSULTATION
DIAGNOSTIC
PROCEDURE
OTHER
```

Room type eligibility may constrain service types.

---

# 15. Service Type Configuration

Each service type must define:

```text
service_type_id
service_type_name
standard_duration
revenue_rule
compensation_rule
NFZ_eligibility
status
```

Example:

```text
FIRST_VISIT
FOLLOW_UP
DIAGNOSTIC
```

Service duration is a key input to capacity calculation.

---

# 16. Doctor Schedule Generation

`DOCTOR_SCHEDULE` represents planned operational availability.

Each schedule record contains:

```text
doctor_schedule_id
doctor_id
clinic_id
schedule_date
start_time
end_time
status
```

The generator must ensure:

```text
start_time < end_time
```

Schedules must not overlap for the same doctor.

---

# 17. Schedule-Based Capacity

Doctor capacity is derived from schedule time.

Conceptually:

```text
Available Minutes
=
End Time
-
Start Time
```

Then:

```text
Potential Slots
=
Available Minutes
/
Service Duration
```

However, actual operational capacity must also consider:

```text
Doctor Qualification
Room Availability
Existing Appointments
Service Type
Breaks
Operational Constraints
```

Therefore:

```text
Potential Capacity
        ↓
Doctor Constraint
        +
Room Constraint
        +
Service Eligibility
        ↓
Operational Capacity
```

---

# 18. NFZ Contract Generation

The generator creates:

```text
NFZ_CONTRACT
```

for applicable clinics and periods.

Each contract contains:

```text
nfz_contract_id
clinic_id
contract_number
contract_start_date
contract_end_date
status
```

---

# 19. NFZ Contract Line Generation

Each contract contains one or more:

```text
NFZ_CONTRACT_LINE
```

The grain is:

```text
Clinic
+
Specialty
+
Service Type
+
Contract Period
```

Each line contains:

```text
nfz_contract_line_id
nfz_contract_id
specialty_id
service_type_id
contracted_volume
effective_from
effective_to
```

---

# 20. Contract Amendments

Contract amendments are represented as effective-dated changes to contract lines.

Example:

```text
Original Contract Line
01.01.2026 – 30.06.2026
Volume = 600
```

Amendment:

```text
Effective Date = 01.07.2026
Volume = 800
```

The generator must preserve historical values.

The historical record must not be overwritten.

The generator should create a new effective-dated version.

---

# 21. Demand Generation

Demand is generated from:

```text
PATIENT
+
SPECIALTY
+
CLINIC
+
TIME
+
DEMAND SOURCE
```

Each demand event contains:

```text
demand_event_id
patient_id
clinic_id
specialty_id
demand_source
demand_date
care_pathway
status
```

Demand volume should vary by:

```text
Specialty
Month
Clinic
Seasonality
Patient Segment
```

---

# 22. Demand Seasonality

The generator should support seasonal variation.

Conceptually:

```text
Base Demand
×
Seasonality Factor
=
Adjusted Demand
```

Example:

```text
January
1.10

February
1.00

July
0.80

September
1.15
```

Exact factors are configuration parameters.

---

# 23. Referral Generation

A demand event may generate a referral.

Conceptually:

```text
DEMAND_EVENT
    ↓
Referral Probability
    ↓
REFERRAL
```

The referral contains:

```text
patient_id
demand_event_id
referring_doctor_id
referring_clinic_id
target_clinic_id
target_specialty_id
referral_date
```

---

# 24. Internal Specialty Availability

Before patient routing, the generator checks:

```text
CLINIC_SPECIALTY
```

If an active internal specialty exists:

```text
Internal Availability = TRUE
```

Otherwise:

```text
Internal Availability = FALSE
```

This affects patient pathway simulation.

---

# 25. Patient Decision Model

Each referral receives a simulated patient decision.

Possible values:

```text
WAIT
EXTERNAL
DROP
```

The decision is probabilistic but constrained by business rules.

Example:

```text
WAIT Probability
+
EXTERNAL Probability
+
DROP Probability
=
1.0
```

Probabilities may vary by:

```text
Specialty
Expected Waiting Time
Clinic
Patient Segment
Demand Pressure
```

---

# 26. Referral Outcome

Patient decision and referral outcome are separate.

```text
PATIENT_DECISION
        ≠
REFERRAL_OUTCOME
```

Example:

```text
WAIT
    ↓
WAITING_LIST_ENTRY
    ↓
APPOINTMENT
    ↓
SERVICE_DELIVERY
    ↓
INTERNAL
```

Example:

```text
EXTERNAL
    ↓
EXTERNAL
```

Example:

```text
DROP
    ↓
NOT_COMPLETED
```

---

# 27. Waiting List Generation

Patients choosing:

```text
WAIT
```

enter:

```text
WAITING_LIST_ENTRY
```

The record contains:

```text
created_at
scheduled_at
completed_at
removed_at
status
removal_reason
```

The waiting list must be processed chronologically.

The queue should behave as a time-dependent operational system.

---

# 28. Waiting List Queue Logic

At each simulation date:

```text
1. Identify active waiting patients.
2. Calculate available capacity.
3. Rank eligible patients.
4. Allocate available slots.
5. Create appointments.
6. Update waiting list state.
7. Continue unresolved patients.
```

The queue may use:

```text
FIFO
```

as the baseline rule.

Optional priority logic may be added later.

---

# 29. Waiting Time Calculation

For completed appointments:

```text
Waiting Time
=
Appointment Date
-
Waiting List Entry Date
```

For unresolved patients:

```text
Current Waiting Time
=
Simulation Date
-
Waiting List Entry Date
```

The generator should preserve both:

```text
Actual Waiting Time
```

and:

```text
Current Waiting Time
```

where analytically required.

---

# 30. Operational Capacity Engine

Operational capacity is derived.

The primary sources are:

```text
DOCTOR
DOCTOR_SPECIALTY
DOCTOR_SCHEDULE
ROOM
SERVICE_TYPE
APPOINTMENT
```

The calculation flow is:

```text
DOCTOR_SCHEDULE
        ↓
Available Doctor Time
        ↓
Doctor Specialty Eligibility
        ↓
SERVICE_TYPE Duration
        ↓
Potential Service Slots
        ↓
ROOM Availability
        ↓
Existing Appointment Occupancy
        ↓
Operational Capacity
```

Operational capacity must not be treated as an arbitrary random number.

---

# 31. Doctor Capacity

For a given:

```text
Doctor
+
Clinic
+
Date
+
Specialty
+
Service Type
```

the theoretical number of slots is:

```text
Available Minutes
/
Service Duration
```

The actual number is constrained by:

```text
Room Availability
```

and:

```text
Existing Appointments
```

---

# 32. Room Capacity

Room capacity is calculated from:

```text
Room Availability
+
Room Type
+
Service Eligibility
+
Appointment Occupancy
```

A room cannot host overlapping appointments.

Therefore:

```text
Room Capacity
=
Available Room Slots
```

---

# 33. Operational Capacity Bottleneck

The system must identify the bottleneck.

Conceptually:

```text
Doctor Capacity
        vs
Room Capacity
```

The effective operational capacity is constrained by the bottleneck.

Example:

```text
Doctor Slots = 100
Room Slots = 70

Operational Capacity = 70
```

This allows the BI layer to identify:

```text
Doctor Bottleneck
```

vs.

```text
Room Bottleneck
```

---

# 34. NFZ Funded Capacity

NFZ funded capacity is based on:

```text
NFZ_CONTRACT_LINE
```

and funded service delivery.

Conceptually:

```text
Contracted Volume
-
Delivered NFZ Services
=
Remaining NFZ Capacity
```

Only services classified as NFZ-funded should reduce the remaining funded capacity.

---

# 35. Effective Capacity

The core rule is:

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

The calculation grain is:

```text
Clinic
+
Specialty
+
Service Type
+
Time Period
```

---

# 36. Capacity Allocation

Available effective capacity is allocated to eligible demand.

The process is:

```text
Effective Capacity
        ↓
Eligible Waiting List
        ↓
Queue Ordering
        ↓
Appointment Slot
        ↓
Appointment
```

Capacity must not be allocated beyond:

```text
Operational Capacity
```

or:

```text
NFZ Funded Capacity
```

where NFZ constraints apply.

---

# 37. Appointment Scheduling

An appointment can be created only when:

```text
Doctor Available
AND
Doctor Qualified
AND
Room Available
AND
Service Eligible
AND
Capacity Available
```

The appointment must satisfy:

```text
No Doctor Overlap
No Room Overlap
Within Doctor Schedule
Valid Specialty
Valid Service Type
```

---

# 38. Appointment Allocation Algorithm

Baseline algorithm:

```text
FOR each simulation date:

    Identify available doctors

    Identify available rooms

    Identify eligible services

    Identify active waiting patients

    Sort waiting patients by:
        waiting_list_entry_date

    FOR each eligible patient:

        Find earliest valid doctor slot

        Find compatible room

        Check operational capacity

        Check NFZ funded capacity

        IF all constraints pass:

            Create appointment

            Reserve doctor time

            Reserve room time

            Update waiting list

        ELSE:

            Keep patient in waiting list
```

---

# 39. Service Delivery

A completed appointment may generate:

```text
SERVICE_DELIVERY
```

The delivery record contains:

```text
appointment_id
patient_id
clinic_id
doctor_id
specialty_id
service_type_id
delivery_date
delivery_status
payer_type
```

Only completed services generate financial transactions.

---

# 40. Revenue Generation

Revenue is generated from completed service delivery.

Conceptually:

```text
SERVICE_DELIVERY
        ↓
SERVICE_TYPE
        ↓
PAYER_TYPE
        ↓
Revenue Rule
        ↓
REVENUE_TRANSACTION
```

Revenue may differ by:

```text
Service Type
Payer Type
Contract
Clinic
```

---

# 41. Doctor Compensation

Doctor compensation is generated from completed service delivery.

Possible models:

```text
PERCENTAGE
HOURLY
MIXED
```

Conceptually:

```text
SERVICE_DELIVERY
        ↓
Doctor
        ↓
Compensation Model
        ↓
DOCTOR_COMPENSATION_TRANSACTION
```

---

# 42. Operating Cost Generation

Operating costs are generated at clinic level.

Cost categories may include:

```text
RENT
UTILITIES
STAFF
SERVICE_VOLUME
ROOM_USAGE
OTHER
```

Cost behavior:

```text
FIXED
VARIABLE
SEMI_VARIABLE
```

The generator should model:

```text
Fixed Costs
+
Variable Costs
+
Semi-Variable Costs
```

---

# 43. Financial Result

The base financial model is:

```text
Service Revenue
-
Doctor Compensation
-
Operating Costs
=
Net Clinic Result
```

At analytical level:

```text
REVENUE_TRANSACTION
        -
DOCTOR_COMPENSATION_TRANSACTION
        -
OPERATING_COST_TRANSACTION
        =
NET CLINIC RESULT
```

---

# 44. Resource Reallocation

Resource reallocations represent management decisions.

Examples:

```text
Doctor moved between clinics
Doctor hours reallocated
Room reassigned
Operational capacity changed
```

The flow is:

```text
RESOURCE_REALLOCATION
        ↓
Effective Date
        ↓
Future Resource State
        ↓
DOCTOR_SCHEDULE / ROOM Availability
        ↓
Operational Capacity
```

Historical data must remain unchanged.

---

# 45. Resource Reallocation Rule

A reallocation effective on:

```text
2026-07-01
```

must not change capacity calculated for:

```text
2026-06-30
```

The generator must preserve historical state.

---

# 46. Scenario Simulation

Scenario analysis uses:

```text
SCENARIO
SCENARIO_CHANGE
```

A scenario is applied to future state.

The baseline remains unchanged.

Flow:

```text
BASELINE
    ↓
SCENARIO
    ↓
SCENARIO_CHANGE
    ↓
FUTURE STATE
    ↓
RECALCULATE CAPACITY
    ↓
RECALCULATE WAITING TIME
    ↓
RECALCULATE SERVICE DELIVERY
    ↓
RECALCULATE FINANCIAL RESULTS
```

---

# 47. Baseline vs Scenario

The generator must support comparison between:

```text
BASELINE
```

and:

```text
SCENARIO
```

Key comparison metrics:

```text
Waiting Time
Waiting List Size
Operational Capacity
Effective Capacity
NFZ Utilization
Internal Capture Rate
Revenue
Doctor Compensation
Operating Costs
Net Clinic Result
```

---

# 48. Internal Capture Rate

Internal capture rate is derived from referral outcomes.

Conceptually:

```text
Internal Capture Rate
=
Internal Referrals
/
Eligible Referrals
```

The numerator should include:

```text
REFERRAL_OUTCOME = INTERNAL
```

The denominator should be explicitly defined in the analytical layer.

Recommended baseline:

```text
Eligible Referrals
=
Internal + External + Not Completed
```

where applicable.

---

# 49. Referral Leakage

Referral leakage is derived from external outcomes.

Conceptually:

```text
Referral Leakage Rate
=
External Referrals
/
Eligible Referrals
```

The generator should preserve:

```text
External
```

as a distinct outcome.

---

# 50. KPI Generation

KPIs are calculated after transactional simulation.

The KPI layer should include:

## Demand

```text
Total Demand
Demand by Specialty
Demand by Clinic
Demand Trend
```

## Capacity

```text
Operational Capacity
NFZ Funded Capacity
Effective Capacity
Capacity Utilization
```

## Waiting

```text
Average Waiting Time
Median Waiting Time
Maximum Waiting Time
Waiting List Size
```

## Referral

```text
Internal Capture Rate
Referral Leakage Rate
External Referral Volume
```

## Operations

```text
Doctor Utilization
Room Utilization
Service Volume
```

## Finance

```text
Revenue
Doctor Compensation
Operating Costs
Net Clinic Result
```

## NFZ

```text
Contracted Volume
Delivered Volume
Contract Utilization
Remaining Capacity
```

---

# 51. Validation Layer

Validation is mandatory.

The generator must not export data before validation succeeds.

Validation is divided into:

```text
1. Structural Validation
2. Referential Validation
3. Temporal Validation
4. Business Rule Validation
5. Capacity Validation
6. Financial Validation
```

---

# 52. Structural Validation

Check:

```text
Primary Keys are unique
Required fields are populated
Data types are valid
Expected columns exist
```

---

# 53. Referential Validation

Check all foreign keys.

Examples:

```text
APPOINTMENT.doctor_id
    → DOCTOR.doctor_id
```

```text
SERVICE_DELIVERY.appointment_id
    → APPOINTMENT.appointment_id
```

```text
NFZ_CONTRACT_LINE.nfz_contract_id
    → NFZ_CONTRACT.nfz_contract_id
```

No orphan records are allowed.

---

# 54. Temporal Validation

Check:

```text
Demand Date
≤ Referral Date
≤ Waiting List Entry Date
≤ Appointment Date
≤ Service Delivery Date
```

Also check:

```text
Contract Start
≤ Contract End
```

and:

```text
Schedule Start
< Schedule End
```

---

# 55. Appointment Validation

The following must never occur:

```text
Doctor Overlap
Room Overlap
Appointment Outside Schedule
Unqualified Doctor
Invalid Specialty
```

---

# 56. Capacity Validation

Check:

```text
Effective Capacity
≤ Operational Capacity
```

and:

```text
Effective Capacity
≤ NFZ Funded Capacity
```

where NFZ constraints apply.

Also check:

```text
Delivered NFZ Services
≤ Contracted Volume
```

unless overperformance is intentionally simulated.

If overperformance is allowed, it must be explicitly flagged.

---

# 57. Financial Validation

Check:

```text
Revenue >= 0
Compensation >= 0
Operating Costs >= 0
```

and:

```text
Net Clinic Result
=
Revenue
-
Compensation
-
Operating Costs
```

within expected rounding tolerance.

---

# 58. Data Quality Thresholds

The generator should calculate:

```text
Null Rate
Duplicate Rate
Orphan Rate
Invalid Date Rate
Invalid FK Rate
```

Target:

```text
Orphan Rate = 0
Invalid FK Rate = 0
Duplicate PK Rate = 0
```

Other thresholds may be configurable.

---

# 59. Generator Module Structure

Recommended Python project structure:

```text
src/
│
├── config/
│   ├── settings.py
│   ├── distributions.py
│   └── rules.py
│
├── generators/
│   ├── clinic_generator.py
│   ├── specialty_generator.py
│   ├── doctor_generator.py
│   ├── room_generator.py
│   ├── patient_generator.py
│   ├── demand_generator.py
│   ├── referral_generator.py
│   └── contract_generator.py
│
├── simulation/
│   ├── schedule_engine.py
│   ├── capacity_engine.py
│   ├── waiting_list_engine.py
│   ├── appointment_engine.py
│   ├── service_engine.py
│   └── scenario_engine.py
│
├── finance/
│   ├── revenue_engine.py
│   ├── compensation_engine.py
│   └── cost_engine.py
│
├── analytics/
│   └── kpi_engine.py
│
├── validation/
│   ├── structural_checks.py
│   ├── referential_checks.py
│   ├── temporal_checks.py
│   ├── business_rule_checks.py
│   └── financial_checks.py
│
└── export/
    ├── csv_exporter.py
    └── database_loader.py
```

The exact structure may evolve during implementation.

---

# 60. Generator Dependency Graph

The dependency graph is:

```text
CLINIC
    ↓
SPECIALTY
    ↓
CLINIC_SPECIALTY
    ↓
DOCTOR
    ↓
DOCTOR_SPECIALTY
    ↓
ROOM
    ↓
SERVICE_TYPE
    ↓
DOCTOR_SCHEDULE
    ↓
NFZ_CONTRACT
    ↓
NFZ_CONTRACT_LINE
    ↓
PATIENT
    ↓
DEMAND_EVENT
    ↓
REFERRAL
    ↓
PATIENT_DECISION
    ↓
WAITING_LIST_ENTRY
    ↓
CAPACITY ENGINE
    ↓
APPOINTMENT
    ↓
SERVICE_DELIVERY
    ↓
REVENUE
    ↓
COMPENSATION
    ↓
OPERATING COST
    ↓
KPI
    ↓
VALIDATION
```

---

# 61. Core Simulation Loop

The main simulation engine should operate by date.

Conceptually:

```python
for simulation_date in simulation_calendar:

    apply_effective_schedule_changes()

    apply_resource_reallocations()

    apply_contract_amendments()

    generate_daily_demand()

    create_referrals()

    simulate_patient_decisions()

    update_waiting_list()

    calculate_daily_capacity()

    schedule_appointments()

    complete_due_appointments()

    generate_service_delivery()

    generate_financial_transactions()
```

After the simulation period:

```python
calculate_kpis()

run_validation()

export_data()
```

---

# 62. Event Ordering

Within a simulation day, the recommended order is:

```text
1. Apply effective changes
2. Generate demand
3. Generate referrals
4. Process patient decisions
5. Update waiting list
6. Calculate capacity
7. Allocate appointments
8. Complete scheduled services
9. Generate financial transactions
10. Update operational state
```

This order may be adjusted depending on the exact simulation granularity.

---

# 63. Deterministic IDs

IDs should be generated deterministically where possible.

Examples:

```text
CL001
SP001
DOC001
ROOM001
PAT000001
DEM000001
REF000001
WLE000001
APT000001
SRV000001
REV000001
```

IDs must be:

```text
Unique
Stable
Non-null
Referentially consistent
```

---

# 64. Output Dataset

The generator should produce the following core datasets:

```text
clinics
specialties
clinic_specialties
doctors
doctor_specialties
doctor_schedules
rooms
patients
service_types
demand_events
referrals
waiting_list_entries
appointments
service_deliveries
revenue_transactions
doctor_compensation_transactions
operating_cost_transactions
nfz_contracts
nfz_contract_lines
resource_reallocations
scenarios
scenario_changes
```

Derived KPI datasets may be generated separately.

---

# 65. Export Strategy

The first implementation should support:

```text
CSV
```

as the primary output.

Optional future targets:

```text
PostgreSQL
Parquet
```

Recommended structure:

```text
data/
├── raw/
├── processed/
└── output/
```

The exact folder structure should be confirmed before implementation.

---

# 66. Generator Run Metadata

Each generator execution should produce metadata.

Example:

```text
run_id
random_seed
simulation_start_date
simulation_end_date
generator_version
timestamp
```

This enables reproducibility and auditability.

Example:

```text
RUN_ID = RUN_2026_07_22_001
SEED = 42
GENERATOR_VERSION = 0.1.0
```

---

# 67. Error Handling

The generator should fail fast on critical integrity violations.

Examples:

```text
Duplicate Primary Key
Missing Foreign Key
Invalid Schedule
Doctor Double Booking
Room Double Booking
Negative Capacity
Invalid Financial Result
```

The generator should produce readable error messages.

Example:

```text
ValidationError:
Doctor DOC023 has overlapping appointments:
APT004521
APT004522
Date: 2026-05-14
```

---

# 68. Logging

The generator should log:

```text
Run Start
Configuration
Seed
Generation Steps
Record Counts
Validation Results
Warnings
Errors
Run Completion
```

Example:

```text
[INFO] Generator started
[INFO] Seed: 42
[INFO] Clinics generated: 11
[INFO] Doctors generated: 87
[INFO] Patients generated: 25,000
[INFO] Demand events generated: 31,250
[INFO] Appointments generated: 21,400
[INFO] Validation passed
[INFO] Export completed
```

---

# 69. Performance Considerations

The initial implementation should prioritize:

```text
Correctness
Reproducibility
Readability
```

over premature optimization.

However, the architecture should avoid unnecessary row-by-row operations where vectorized Pandas operations are appropriate.

Potential optimization areas:

```text
Demand generation
Daily capacity calculations
Waiting list processing
Appointment allocation
KPI aggregation
```

---

# 70. Recommended Implementation Sequence

Implementation should proceed in phases.

## Phase 1 — Foundation

```text
Configuration
Random Seed
Logging
ID Generation
```

## Phase 2 — Master Data

```text
Clinics
Specialties
Doctors
Rooms
Service Types
```

## Phase 3 — Relationships

```text
Clinic Specialties
Doctor Specialties
```

## Phase 4 — Operational Resources

```text
Doctor Schedules
NFZ Contracts
NFZ Contract Lines
```

## Phase 5 — Demand

```text
Patients
Demand Events
Referrals
Patient Decisions
```

## Phase 6 — Operations

```text
Waiting List
Capacity Engine
Appointment Engine
Service Delivery
```

## Phase 7 — Finance

```text
Revenue
Compensation
Operating Costs
```

## Phase 8 — Management

```text
Resource Reallocation
Scenario Simulation
```

## Phase 9 — Analytics

```text
KPI Engine
```

## Phase 10 — Validation

```text
Structural
Referential
Temporal
Business Rules
Financial
```

## Phase 11 — Export

```text
CSV
Database
```

---

# 71. Implementation Readiness

The architecture is considered ready for implementation when:

```text
✓ Business Rules are defined
✓ Entities are defined
✓ Process Flows are defined
✓ ERD is defined
✓ Generation order is defined
✓ Capacity logic is defined
✓ NFZ logic is defined
✓ Waiting list logic is defined
✓ Appointment constraints are defined
✓ Financial lineage is defined
✓ Validation strategy is defined
```

Current status:

```text
READY FOR PYTHON GENERATOR IMPLEMENTATION
```

---

# 72. Open Implementation Decisions

The following decisions should be finalized during implementation planning.

## 72.1 Simulation Granularity

Recommended:

```text
Daily simulation
```

with appointment-level timestamps.

---

## 72.2 Waiting List Priority

Recommended baseline:

```text
FIFO
```

with optional priority extension.

---

## 72.3 NFZ Overperformance

Decision required:

```text
Allow overperformance
```

or:

```text
Hard cap at contracted volume
```

Recommended:

```text
Allow overperformance as a flagged business condition.
```

This allows analysis of:

```text
Contract Overperformance
```

without hiding operational reality.

---

## 72.4 Resource Reallocation

Recommended:

```text
Event-based
Effective-dated
Future-state only
```

Historical data must remain immutable.

---

## 72.5 Contract Amendments

Recommended:

```text
Effective-dated NFZ_CONTRACT_LINE versions
```

rather than a separate amendment entity.

---

# 73. Final Architecture

The final architecture is:

```text
CONFIGURATION
      ↓
MASTER DATA
      ↓
RESOURCE STATE
      ↓
SCHEDULES
      ↓
CONTRACTS
      ↓
DEMAND
      ↓
PATIENT BEHAVIOR
      ↓
WAITING LIST
      ↓
CAPACITY ENGINE
      ↓
APPOINTMENT ENGINE
      ↓
SERVICE DELIVERY
      ↓
FINANCIAL ENGINE
      ↓
SCENARIO ENGINE
      ↓
KPI ENGINE
      ↓
VALIDATION ENGINE
      ↓
EXPORT
```

The generator must preserve the principle:

```text
Business Rules
        ↓
Simulation Logic
        ↓
Generated Transactions
        ↓
Analytical Metrics
```

The generator must never bypass the business model by generating final KPIs independently of the underlying operational transactions.

---

# 74. Next Step

The next implementation stage is:

```text
08 — Python Generator Foundation
```

The first coding milestone should not generate the full healthcare simulation.

It should implement:

```text
1. Project structure
2. Configuration
3. Random seed management
4. Logging
5. Deterministic ID generation
6. Base generator interfaces
7. Data validation framework
8. Minimal master data generation
```

After the foundation passes validation, the project should incrementally implement:

```text
Master Data
        ↓
Resources
        ↓
Capacity
        ↓
Demand
        ↓
Operations
        ↓
Finance
        ↓
Scenarios
```

Each stage should produce a valid, testable dataset before the next stage is added.

---

# 75. Document Status

**Version:** 1.0

**Status:** Draft — Ready for Implementation Planning

**Dependencies:**

```text
01_Project_Overview.md
02_Business_Model.md
03_Business_Rules.md
04_Entity_Dictionary.md
05_Process_Flows.md
06_ERD.md
```

**Architectural Principle:**

```text
Generate the system,
not just the data.
```

The Python generator is the executable implementation of the business rules, process flows, and logical data model defined in the preceding documentation.
