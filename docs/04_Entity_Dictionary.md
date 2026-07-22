# 04 — Entity Dictionary

## 1. Purpose

This document defines the logical data entities required by the **Healthcare Operations BI Simulator**.

The Entity Dictionary translates the business concepts and rules defined in:

* `01_Project_Overview.md`
* `02_Business_Model.md`
* `03_Business_Rules.md`

into a structured logical data model.

It is the bridge between the business domain and the technical implementation:

```text
Project Overview
       ↓
Business Model
       ↓
Business Rules
       ↓
Entity Dictionary
       ↓
Process Flows
       ↓
ERD
       ↓
Generator Architecture
       ↓
Python Data Generator
       ↓
SQL Data Model
       ↓
Tableau Analytics
```

The Entity Dictionary defines:

* entities,
* entity purpose,
* grain,
* primary keys,
* foreign keys,
* important attributes,
* relationships,
* business meaning,
* derived metrics,
* and implementation notes.

The document is designed to ensure that the Python generator, SQL model, and Tableau analytical layer are based on the same logical data model.

---

# 2. Modeling Principles

The data model follows several core principles.

## 2.1 Separate Master Data, Events, State, Rules, and Derived Metrics

The model distinguishes between:

```text
MASTER DATA
    ↓
Entities that define the organization

EVENTS
    ↓
Things that happen

STATE
    ↓
Current or historical operational conditions

RULES / PARAMETERS
    ↓
Configuration used by the simulator

DERIVED METRICS
    ↓
Values calculated from source data
```

Examples:

```text
PATIENT
DOCTOR
CLINIC
SPECIALTY
ROOM
```

are master data.

```text
REFERRAL
APPOINTMENT
VISIT
SERVICE_DELIVERY
PATIENT_DECISION
```

are operational events.

```text
WAITING_LIST_ENTRY
```

represents an operational state over time.

```text
PATIENT_BEHAVIOR_RULE
COMPENSATION_RULE
```

represent configurable business logic.

```text
WAITING_TIME
EFFECTIVE_CAPACITY
CONTRACT_UTILIZATION
INTERNAL_CAPTURE_RATE
EXTERNAL_LEAKAGE_RATE
```

are derived metrics and should generally not be stored as independent source entities.

---

## 2.2 Grain Must Be Explicit

Every fact or event entity must have a clearly defined grain.

Examples:

```text
One REFERRAL
= one patient referral event

One APPOINTMENT
= one scheduled patient appointment

One VISIT
= one actual patient attendance event

One SERVICE_DELIVERY
= one delivered healthcare service

One WAITING_LIST_ENTRY
= one patient episode in a waiting queue

One CAPACITY_FACT
= one capacity record for a given date / clinic / specialty / service type
```

This prevents double counting and supports reliable SQL and BI analysis.

---

## 2.3 Derived Metrics Are Not Independent Entities

The following should be calculated from source data where possible:

```text
Waiting Time
Effective Capacity
Capacity Utilization
Contract Utilization
Remaining Contract Capacity
Overperformance
Underutilization
Internal Capture Rate
External Leakage Rate
Non-Completion Rate
Clinic Revenue
Net Clinic Result
```

These metrics may be implemented as SQL views, analytical tables, or Tableau calculations depending on performance requirements.

---

## 2.4 Temporal Modeling

The simulator is time-dependent.

Operational events and states must support:

* daily analysis,
* weekly analysis,
* monthly analysis,
* quarterly analysis,
* annual analysis.

The model therefore uses a centralized:

```text
DATE_DIMENSION
```

for consistent analytical aggregation.

---

# 3. Entity Classification

| Entity                          | Type            | Grain                                             |
| ------------------------------- | --------------- | ------------------------------------------------- |
| DATE_DIMENSION                  | Dimension       | One calendar date                                 |
| PATIENT                         | Master          | One patient                                       |
| CLINIC                          | Master          | One operational clinic                            |
| SPECIALTY                       | Master          | One medical specialty                             |
| DOCTOR                          | Master          | One doctor                                        |
| DOCTOR_CLINIC_ASSIGNMENT        | Relationship    | One doctor-clinic assignment                      |
| ROOM                            | Master          | One physical room                                 |
| SERVICE_TYPE                    | Master          | One service definition                            |
| DOCTOR_SCHEDULE                 | Operational     | One doctor schedule record                        |
| ROOM_SCHEDULE                   | Operational     | One room availability record                      |
| PATIENT_VISIT                   | Event           | One primary-care or pediatric visit               |
| DEMAND_EVENT                    | Event           | One demand event                                  |
| REFERRAL                        | Event           | One referral                                      |
| PATIENT_DECISION                | Event           | One patient decision                              |
| WAITING_LIST_ENTRY              | State/Event     | One waiting episode                               |
| APPOINTMENT                     | Event           | One scheduled appointment                         |
| VISIT                           | Event           | One actual attendance event                       |
| SERVICE_DELIVERY                | Event/Fact      | One delivered healthcare service                  |
| NFZ_CONTRACT                    | Master/Contract | One contract service line                         |
| NFZ_CONTRACT_AMENDMENT          | Event           | One contract amendment                            |
| CAPACITY_FACT                   | Fact            | One date/clinic/specialty/service capacity record |
| RESOURCE_REALLOCATION           | Event           | One resource reallocation event                   |
| REVENUE_TRANSACTION             | Fact            | One revenue event                                 |
| COMPENSATION_RULE               | Rule            | One compensation configuration                    |
| DOCTOR_COMPENSATION_TRANSACTION | Fact            | One compensation transaction                      |
| COST_CATEGORY                   | Dimension       | One cost category                                 |
| OPERATING_COST                  | Fact            | One operating cost transaction                    |
| SCENARIO                        | Dimension       | One simulation scenario                           |
| SCENARIO_PARAMETER              | Rule            | One scenario parameter                            |
| SCENARIO_RESOURCE_CHANGE        | Event           | One scenario resource change                      |

---

# 4. Core Master Data Entities

## 4.1 DATE_DIMENSION

### Purpose

Central calendar dimension used for consistent time-based analysis.

### Grain

One record per calendar date.

### Primary Key

```text
date_key
```

### Attributes

```text
date_key
date
day
day_of_week
week_number
month
month_name
quarter
year
is_weekend
```

### Derived / Analytical Use

Supports:

* daily reporting,
* weekly reporting,
* monthly reporting,
* quarterly NFZ settlement,
* annual analysis.

---

## 4.2 PATIENT

### Purpose

Represents an individual synthetic patient in the simulated healthcare organization.

### Grain

One record per patient.

### Primary Key

```text
patient_id
```

### Attributes

```text
patient_id
date_of_birth
gender
registration_date
patient_status
```

### Relationships

```text
PATIENT
    ↓
PATIENT_VISIT
    ↓
REFERRAL
    ↓
PATIENT_DECISION
    ↓
WAITING_LIST_ENTRY
    ↓
APPOINTMENT
    ↓
VISIT
    ↓
SERVICE_DELIVERY
```

### Notes

Patient-level records should use synthetic identifiers only.

No real patient-identifiable information is required.

---

## 4.3 CLINIC

### Purpose

Represents an operational healthcare clinic or organizational unit.

### Grain

One record per clinic.

### Primary Key

```text
clinic_id
```

### Attributes

```text
clinic_id
clinic_name
clinic_type
specialty_id
opening_date
closing_date
is_active
```

### Relationships

A clinic may have:

* doctors,
* rooms,
* schedules,
* service types,
* demand,
* referrals,
* capacity,
* waiting lists,
* NFZ contracts,
* revenue,
* operating costs.

---

## 4.4 SPECIALTY

### Purpose

Represents the master catalog of medical specialties.

### Grain

One record per specialty.

### Primary Key

```text
specialty_id
```

### Attributes

```text
specialty_id
specialty_name
available_internally
is_active
```

### Business Rule

The model should contain approximately 20–25 specialties.

The organization initially provides approximately 11 specialties internally.

Therefore:

```text
available_internally = TRUE
```

identifies specialties currently offered by the organization.

Specialties with:

```text
available_internally = FALSE
```

remain in the catalog and may generate external referral leakage.

---

## 4.5 DOCTOR

### Purpose

Represents a healthcare professional providing services.

### Grain

One record per doctor.

### Primary Key

```text
doctor_id
```

### Attributes

```text
doctor_id
first_name
last_name
specialty_id
employment_type
compensation_model
is_active
```

### Notes

Doctor capacity is not stored as a static attribute.

Capacity is derived from:

* doctor availability,
* schedules,
* working hours,
* service duration,
* room availability.

---

## 4.6 DOCTOR_CLINIC_ASSIGNMENT

### Purpose

Represents the relationship between doctors and clinics.

### Grain

One record per doctor-clinic assignment over a defined effective period.

### Primary Key

```text
doctor_clinic_assignment_id
```

### Foreign Keys

```text
doctor_id
clinic_id
```

### Attributes

```text
effective_from
effective_to
is_primary_assignment
```

### Notes

This entity allows doctors to work in more than one clinic or to be reallocated between clinics over time.

---

## 4.7 ROOM

### Purpose

Represents a physical room used for healthcare services.

### Grain

One record per room.

### Primary Key

```text
room_id
```

### Foreign Keys

```text
clinic_id
```

### Attributes

```text
room_name
room_type
is_active
```

### Business Rule

A room cannot host overlapping appointments.

---

## 4.8 SERVICE_TYPE

### Purpose

Defines the type of healthcare service delivered.

### Grain

One record per service type.

### Primary Key

```text
service_type_id
```

### Attributes

```text
service_type_name
visit_category
default_duration_minutes
is_nfz_eligible
is_commercial
is_active
```

### Supported Visit Categories

```text
FIRST_VISIT
FOLLOW_UP
PRIMARY_CARE
PEDIATRIC
DIAGNOSTIC
```

### Notes

Service type determines operational and financial characteristics.

Different service types may have different:

* appointment durations,
* revenue,
* NFZ limits,
* compensation rules.

---

# 5. Scheduling and Availability Entities

## 5.1 DOCTOR_SCHEDULE

### Purpose

Represents the planned availability of a doctor.

### Grain

One doctor schedule record for a defined date/time interval.

### Primary Key

```text
doctor_schedule_id
```

### Foreign Keys

```text
doctor_id
clinic_id
date_key
```

### Attributes

```text
schedule_start_datetime
schedule_end_datetime
available_minutes
schedule_status
```

### Notes

Doctor schedules are inputs to operational capacity.

---

## 5.2 ROOM_SCHEDULE

### Purpose

Represents the planned availability of a room.

### Grain

One room availability record for a defined date/time interval.

### Primary Key

```text
room_schedule_id
```

### Foreign Keys

```text
room_id
clinic_id
date_key
```

### Attributes

```text
schedule_start_datetime
schedule_end_datetime
available_minutes
schedule_status
```

### Notes

Room availability constrains operational capacity.

---

# 6. Demand and Referral Entities

## 6.1 DEMAND_EVENT

### Purpose

Represents a single simulated patient demand event.

### Grain

One record per demand event.

### Primary Key

```text
demand_event_id
```

### Foreign Keys

```text
patient_id
clinic_id
specialty_id
service_type_id
date_key
```

### Attributes

```text
demand_datetime
demand_source
demand_status
```

### Demand Sources

Examples:

```text
DIRECT_PRIMARY_CARE
PEDIATRIC
SPECIALIST_REFERRAL
FOLLOW_UP
DIAGNOSTIC
```

### Important Rule

Demand does not automatically equal completed visits.

A demand event may result in:

```text
INTERNAL_SERVICE
WAITING
EXTERNAL
NOT_COMPLETED
```

### Grain Decision

The base simulation model uses patient-level demand events.

Daily, weekly, monthly, quarterly, and annual demand should be aggregated analytically.

---

## 6.2 PATIENT_VISIT

### Purpose

Represents primary-care or pediatric encounters that may generate specialist referrals.

### Grain

One record per primary-care or pediatric visit.

### Primary Key

```text
patient_visit_id
```

### Foreign Keys

```text
patient_id
doctor_id
clinic_id
service_type_id
date_key
```

### Attributes

```text
visit_datetime
visit_type
referral_required
```

### Visit Types

```text
PRIMARY_CARE
PEDIATRIC
```

---

## 6.3 REFERRAL

### Purpose

Represents a referral from an originating healthcare encounter to a target specialty.

### Grain

One record per referral.

### Primary Key

```text
referral_id
```

### Foreign Keys

```text
patient_id
referring_doctor_id
referring_clinic_id
target_specialty_id
source_patient_visit_id
date_key
```

### Attributes

```text
referral_datetime
referral_status
destination_type
```

### Referral Status

```text
CREATED
WAITING
SCHEDULED
COMPLETED
CANCELLED
```

### Destination Type

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

### Important Modeling Rule

`referral_status` and `destination_type` represent different concepts.

They must not be merged into a single attribute.

Example:

```text
referral_status = WAITING
destination_type = INTERNAL
```

means that the patient is currently waiting for internal care.

---

# 7. Patient Behavior and Waiting List Entities

## 7.1 PATIENT_DECISION

### Purpose

Represents a patient's decision in response to the referral pathway and waiting time.

### Grain

One record per patient decision event.

### Primary Key

```text
patient_decision_id
```

### Foreign Keys

```text
referral_id
patient_id
date_key
```

### Attributes

```text
decision_datetime
decision_type
waiting_time_at_decision_days
decision_probability
```

### Decision Types

```text
WAIT
EXTERNAL
DROP
```

### Business Rule

For an internally available specialty, patient behavior is influenced primarily by waiting time.

For an unavailable specialty:

```text
Referral
    ↓
No Internal Clinic
    ↓
EXTERNAL
```

---

## 7.2 WAITING_LIST_ENTRY

### Purpose

Represents one patient's episode in an internal waiting queue.

### Grain

One record per patient referral waiting episode.

### Primary Key

```text
waiting_list_entry_id
```

### Foreign Keys

```text
referral_id
patient_id
clinic_id
specialty_id
service_type_id
date_entered_key
date_exited_key
```

### Attributes

```text
queue_entry_datetime
queue_exit_datetime
queue_status
exit_reason
```

### Queue Status

```text
WAITING
SCHEDULED
COMPLETED
EXTERNAL
DROPPED
```

### Exit Reasons

Examples:

```text
APPOINTMENT_SCHEDULED
SERVICE_COMPLETED
EXTERNAL_PROVIDER
PATIENT_DROPPED
REFERRAL_CANCELLED
```

### Important Rule

The waiting list is dynamic.

Queue size should be derived from active waiting list entries for a given point in time.

---

## 7.3 WAITING TIME

### Modeling Decision

`WAITING_TIME` is not modeled as an independent source entity.

It is a derived metric.

For completed internal care:

```text
Waiting Time
=
Appointment Start Datetime
-
Queue Entry Datetime
```

For an active waiting episode:

```text
Current Waiting Time
=
Current Date
-
Queue Entry Datetime
```

Waiting time may influence patient decision probability.

---

# 8. Appointment and Service Delivery Entities

## 8.1 APPOINTMENT

### Purpose

Represents a scheduled appointment.

### Grain

One record per scheduled appointment.

### Primary Key

```text
appointment_id
```

### Foreign Keys

```text
patient_id
doctor_id
clinic_id
room_id
specialty_id
service_type_id
referral_id
date_key
```

### Attributes

```text
scheduled_start_datetime
scheduled_end_datetime
appointment_status
booking_datetime
```

### Appointment Status

```text
SCHEDULED
COMPLETED
CANCELLED
NO_SHOW
```

### Business Rules

The model must prevent:

```text
Doctor overlap
Room overlap
```

The same doctor cannot have overlapping appointments.

The same room cannot host overlapping appointments.

---

## 8.2 VISIT

### Purpose

Represents the actual patient attendance associated with an appointment.

### Grain

One record per actual patient attendance event.

### Primary Key

```text
visit_id
```

### Foreign Keys

```text
appointment_id
patient_id
doctor_id
clinic_id
room_id
service_type_id
date_key
```

### Attributes

```text
actual_start_datetime
actual_end_datetime
attendance_status
```

### Attendance Status

```text
ATTENDED
NO_SHOW
CANCELLED
```

### Modeling Rule

An appointment is a planned event.

A visit is an actual attendance event.

They must remain separate.

---

## 8.3 SERVICE_DELIVERY

### Purpose

Represents the healthcare service actually delivered and used for operational and financial analysis.

### Grain

One record per delivered healthcare service.

### Primary Key

```text
service_delivery_id
```

### Foreign Keys

```text
visit_id
patient_id
doctor_id
clinic_id
specialty_id
service_type_id
date_key
nfz_contract_id
```

### Attributes

```text
delivery_datetime
payer_type
delivery_status
```

### Payer Types

```text
NFZ
COMMERCIAL
```

### Delivery Status

```text
DELIVERED
```

### Important Rule

`SERVICE_DELIVERY` represents actual output.

It must not be confused with:

```text
DEMAND
OPERATIONAL_CAPACITY
NFZ_FUNDED_CAPACITY
EFFECTIVE_CAPACITY
```

These are separate concepts.

---

# 9. Capacity Entities

## 9.1 CAPACITY_FACT

### Purpose

Stores calculated operational and funded capacity for analytical use.

### Grain

One record per:

```text
Date
Clinic
Specialty
Service Type
```

### Primary Key

```text
capacity_fact_id
```

### Foreign Keys

```text
date_key
clinic_id
specialty_id
service_type_id
```

### Attributes

```text
operational_capacity
nfz_funded_capacity
effective_capacity
actual_delivered_services
```

### Capacity Definitions

```text
Operational Capacity
=
Physical service capacity based on available resources

NFZ Funded Capacity
=
Contractually funded service capacity

Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

### Important Rule

`actual_delivered_services` is sourced from `SERVICE_DELIVERY`.

It should not be independently generated as an unrelated capacity measure.

---

## 9.2 Operational Capacity Calculation

Operational capacity depends on:

```text
DOCTOR
+
DOCTOR_SCHEDULE
+
ROOM
+
ROOM_SCHEDULE
+
SERVICE_TYPE
```

Conceptually:

```text
Doctor Available Minutes
        ÷
Service Duration
        ↓
Doctor Capacity

Room Available Minutes
        ÷
Service Duration
        ↓
Room Capacity

MIN(
    Doctor Capacity,
    Room Capacity
)
        ↓
Operational Capacity
```

This ensures that operational capacity cannot exceed the physical ability of the organization to deliver services.

---

# 10. NFZ Contract Entities

## 10.1 NFZ_CONTRACT

### Purpose

Represents a simulated NFZ contract for a specific service line.

### Grain

One contract service line for a defined period.

### Primary Key

```text
nfz_contract_id
```

### Foreign Keys

```text
clinic_id
specialty_id
service_type_id
```

### Attributes

```text
contract_start_date
contract_end_date
contracted_volume
contract_status
```

### Contract Status

```text
ACTIVE
EXPIRED
CANCELLED
```

### Analytical Metrics

The following are derived:

```text
Delivered Funded Services
Remaining Contract Capacity
Contract Utilization
Overperformance
Underutilization
```

---

## 10.2 NFZ_CONTRACT_AMENDMENT

### Purpose

Represents a change to an existing NFZ contract.

### Grain

One contract amendment event.

### Primary Key

```text
amendment_id
```

### Foreign Keys

```text
nfz_contract_id
effective_date_key
```

### Attributes

```text
amendment_type
previous_contracted_volume
new_contracted_volume
change_amount
```

### Amendment Types

```text
INCREASE
DECREASE
SERVICE_VOLUME_CHANGE
```

### Business Impact

A contract amendment may affect:

```text
NFZ Funded Capacity
        ↓
Effective Capacity
        ↓
Waiting Time
        ↓
Patient Leakage
        ↓
Delivered Services
        ↓
Revenue
```

---

# 11. Resource Reallocation

## 11.1 RESOURCE_REALLOCATION

### Purpose

Represents a decision to move resources between clinics.

### Grain

One resource reallocation event.

### Primary Key

```text
resource_reallocation_id
```

### Foreign Keys

```text
source_clinic_id
target_clinic_id
doctor_id
source_room_id
target_room_id
effective_date_key
```

### Attributes

```text
resource_type
quantity
trigger_reason
```

### Resource Types

```text
DOCTOR
DOCTOR_HOURS
ROOM
OPERATIONAL_CAPACITY
```

### Trigger Reasons

Examples:

```text
LOW_UTILIZATION
HIGH_DEMAND
LONG_WAITING_TIME
HIGH_LEAKAGE
STRATEGIC_PRIORITY
```

### Modeling Rule

The model must preserve:

```text
Current State
        ↓
Reallocation Event
        ↓
Post-Reallocation State
```

This allows before-and-after analysis.

---

# 12. Revenue and Financial Entities

## 12.1 REVENUE_TRANSACTION

### Purpose

Represents economic revenue generated by a delivered healthcare service.

### Grain

One revenue transaction per delivered service.

### Primary Key

```text
revenue_transaction_id
```

### Foreign Keys

```text
service_delivery_id
patient_id
doctor_id
clinic_id
service_type_id
date_key
```

### Attributes

```text
payer_type
gross_service_revenue
```

### Payer Types

```text
NFZ
COMMERCIAL
```

---

## 12.2 COMPENSATION_RULE

### Purpose

Defines how doctor compensation is calculated.

### Grain

One compensation rule for a doctor or applicable service configuration over an effective period.

### Primary Key

```text
compensation_rule_id
```

### Foreign Keys

```text
doctor_id
service_type_id
```

### Attributes

```text
compensation_model
percentage_rate
hourly_rate
effective_from
effective_to
```

### Compensation Models

```text
PERCENTAGE
HOURLY_RATE
MIXED
```

### Notes

This entity stores the business rule.

It does not represent an actual payment.

---

## 12.3 DOCTOR_COMPENSATION_TRANSACTION

### Purpose

Represents the actual compensation cost generated from delivered services.

### Grain

One compensation transaction.

### Primary Key

```text
compensation_transaction_id
```

### Foreign Keys

```text
service_delivery_id
doctor_id
compensation_rule_id
date_key
```

### Attributes

```text
compensation_amount
```

### Modeling Rule

The distinction is:

```text
COMPENSATION_RULE
        ↓
Business Logic

DOCTOR_COMPENSATION_TRANSACTION
        ↓
Actual Financial Cost
```

---

## 12.4 COST_CATEGORY

### Purpose

Defines operating cost categories.

### Grain

One record per cost category.

### Primary Key

```text
cost_category_id
```

### Attributes

```text
cost_category_name
cost_behavior
```

### Cost Behaviors

```text
FIXED
VARIABLE
SEMI_VARIABLE
```

### Example Categories

```text
NURSING
REGISTRATION
ADMINISTRATION
CLEANING
UTILITIES
RENT
IT
MEDICAL_SYSTEMS
INSURANCE
EQUIPMENT
DEPRECIATION
OTHER
```

---

## 12.5 OPERATING_COST

### Purpose

Represents operating costs incurred by the organization.

### Grain

One cost transaction for a defined clinic and accounting period.

### Primary Key

```text
operating_cost_id
```

### Foreign Keys

```text
clinic_id
cost_category_id
date_key
```

### Attributes

```text
cost_amount
cost_description
```

---

# 13. Scenario Modeling Entities

## 13.1 SCENARIO

### Purpose

Represents a simulation scenario.

### Grain

One record per scenario.

### Primary Key

```text
scenario_id
```

### Attributes

```text
scenario_name
scenario_type
description
is_baseline
```

### Scenario Types

Examples:

```text
BASELINE
NEW_CLINIC
NEW_SPECIALTY
NEW_DOCTORS
NEW_ROOMS
NEW_EQUIPMENT
NEW_NFZ_CONTRACT
RESOURCE_REALLOCATION
```

### Modeling Rule

The baseline scenario must remain immutable.

Scenario changes must not overwrite baseline source data.

---

## 13.2 SCENARIO_PARAMETER

### Purpose

Stores configurable parameters used by the simulation.

### Grain

One parameter value for one scenario.

### Primary Key

```text
scenario_parameter_id
```

### Foreign Keys

```text
scenario_id
```

### Attributes

```text
parameter_name
parameter_value
parameter_type
effective_from
effective_to
```

### Examples

```text
WAITING_TIME_LEAKAGE_RATE
SEASONALITY_FACTOR
DEMAND_GROWTH_RATE
NO_SHOW_RATE
```

---

## 13.3 SCENARIO_RESOURCE_CHANGE

### Purpose

Represents a simulated resource change introduced by a scenario.

### Grain

One resource change within one scenario.

### Primary Key

```text
scenario_resource_change_id
```

### Foreign Keys

```text
scenario_id
clinic_id
specialty_id
doctor_id
room_id
service_type_id
```

### Attributes

```text
resource_type
change_type
quantity
effective_date
```

### Resource Types

```text
DOCTOR
DOCTOR_HOURS
ROOM
EQUIPMENT
NFZ_CAPACITY
OPERATIONAL_CAPACITY
```

### Change Types

```text
ADD
REMOVE
REALLOCATE
INCREASE
DECREASE
```

---

# 14. Business Rules and Parameter Entities

## 14.1 PATIENT_BEHAVIOR_RULE

### Purpose

Stores configurable patient leakage assumptions.

### Grain

One waiting-time interval and associated behavioral probability.

### Primary Key

```text
patient_behavior_rule_id
```

### Foreign Keys

```text
scenario_id
```

### Attributes

```text
min_waiting_days
max_waiting_days
external_probability
drop_probability
wait_probability
effective_from
effective_to
```

### Example Configuration

```text
0–7 days       → 5% leakage
8–30 days      → 10%
31–60 days     → 20%
61–90 days     → 35%
91–180 days    → 55%
180+ days      → 75%
```

### Important Rule

These values are simulation assumptions.

They are configurable parameters and are not intended to represent real-world clinical statistics.

---

# 15. Derived Metrics

The following metrics should generally be calculated from source entities.

## 15.1 Waiting Time

```text
Appointment Start
-
Queue Entry
```

---

## 15.2 Effective Capacity

```text
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

---

## 15.3 Capacity Utilization

```text
Delivered Services
/
Effective Capacity
```

---

## 15.4 Operational Utilization

```text
Delivered Services
/
Operational Capacity
```

---

## 15.5 NFZ Contract Utilization

```text
Delivered Funded Services
/
Contracted Services
```

---

## 15.6 Remaining Contract Capacity

```text
Contracted Volume
-
Delivered Funded Services
```

---

## 15.7 Internal Capture Rate

```text
Internal Referrals
/
Total Referrals
```

---

## 15.8 External Leakage Rate

```text
External Referrals
/
Total Referrals
```

---

## 15.9 Referral Non-Completion Rate

```text
Not Completed Referrals
/
Total Referrals
```

---

## 15.10 Clinic Revenue

Conceptually:

```text
Total Service Revenue
-
Doctor Compensation
```

The exact financial implementation may be extended to include other direct revenue allocations.

---

## 15.11 Net Clinic Result

```text
Clinic Revenue
-
Operating Costs
```

---

# 16. Core Entity Relationships

The primary operational flow is:

```text
PATIENT
    ↓
PATIENT_VISIT
    ↓
REFERRAL
    ↓
SPECIALTY
    ↓
    ├── Available
    │       ↓
    │   CAPACITY
    │       ↓
    │   WAITING_LIST_ENTRY
    │       ↓
    │   PATIENT_DECISION
    │       ├── WAIT
    │       ├── EXTERNAL
    │       └── DROP
    │
    └── Not Available
            ↓
        EXTERNAL
```

Internal service flow:

```text
WAITING_LIST_ENTRY
        ↓
APPOINTMENT
        ↓
VISIT
        ↓
SERVICE_DELIVERY
        ↓
REVENUE_TRANSACTION
        ↓
DOCTOR_COMPENSATION_TRANSACTION
        ↓
CLINIC FINANCIAL RESULT
```

Capacity flow:

```text
DOCTOR
    +
DOCTOR_SCHEDULE
    +
ROOM
    +
ROOM_SCHEDULE
    +
SERVICE_TYPE
        ↓
OPERATIONAL_CAPACITY
        +
NFZ_CONTRACT
        ↓
NFZ_FUNDED_CAPACITY
        ↓
EFFECTIVE_CAPACITY
        ↓
WAITING_TIME
        ↓
PATIENT_DECISION
```

Strategic flow:

```text
SCENARIO
    ↓
SCENARIO_PARAMETER
    +
SCENARIO_RESOURCE_CHANGE
    ↓
Changed Resources / Capacity
    ↓
Demand Fulfillment
    ↓
Waiting Time
    ↓
Leakage
    ↓
Service Delivery
    ↓
Revenue
    ↓
Costs
    ↓
Net Result
```

---

# 17. Source-of-Truth Rules

The following rules define which entity is the source of truth for major concepts.

| Concept                     | Source of Truth                 |
| --------------------------- | ------------------------------- |
| Patient                     | PATIENT                         |
| Clinic                      | CLINIC                          |
| Specialty                   | SPECIALTY                       |
| Doctor                      | DOCTOR                          |
| Doctor-Clinic Relationship  | DOCTOR_CLINIC_ASSIGNMENT        |
| Room                        | ROOM                            |
| Service Definition          | SERVICE_TYPE                    |
| Doctor Availability         | DOCTOR_SCHEDULE                 |
| Room Availability           | ROOM_SCHEDULE                   |
| Demand                      | DEMAND_EVENT                    |
| Primary/Pediatric Encounter | PATIENT_VISIT                   |
| Referral                    | REFERRAL                        |
| Patient Decision            | PATIENT_DECISION                |
| Waiting Episode             | WAITING_LIST_ENTRY              |
| Scheduled Care              | APPOINTMENT                     |
| Actual Attendance           | VISIT                           |
| Delivered Service           | SERVICE_DELIVERY                |
| Physical Capacity           | CAPACITY_FACT                   |
| NFZ Contract                | NFZ_CONTRACT                    |
| Contract Change             | NFZ_CONTRACT_AMENDMENT          |
| Resource Movement           | RESOURCE_REALLOCATION           |
| Service Revenue             | REVENUE_TRANSACTION             |
| Compensation Logic          | COMPENSATION_RULE               |
| Compensation Cost           | DOCTOR_COMPENSATION_TRANSACTION |
| Cost Classification         | COST_CATEGORY                   |
| Operating Cost              | OPERATING_COST                  |
| Scenario                    | SCENARIO                        |
| Scenario Parameters         | SCENARIO_PARAMETER              |
| Scenario Resource Changes   | SCENARIO_RESOURCE_CHANGE        |
| Patient Leakage Assumptions | PATIENT_BEHAVIOR_RULE           |
| Calendar                    | DATE_DIMENSION                  |

---

# 18. Important Modeling Boundaries

The following distinctions must be preserved.

## 18.1 Demand vs Service Delivery

```text
DEMAND_EVENT
≠
SERVICE_DELIVERY
```

Demand represents required or requested care.

Service delivery represents care actually provided.

---

## 18.2 Appointment vs Visit

```text
APPOINTMENT
≠
VISIT
```

Appointment is scheduled.

Visit is actual attendance.

---

## 18.3 Visit vs Service Delivery

```text
VISIT
≠
SERVICE_DELIVERY
```

A visit represents attendance.

Service delivery represents the actual healthcare service used for operational and financial accounting.

---

## 18.4 Referral Status vs Referral Destination

```text
referral_status
≠
destination_type
```

These represent different dimensions of the referral lifecycle.

---

## 18.5 Operational Capacity vs NFZ Capacity

```text
Operational Capacity
≠
NFZ Funded Capacity
```

Operational capacity is physically constrained.

NFZ capacity is contractually constrained.

---

## 18.6 Effective Capacity

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

Effective capacity is derived.

---

## 18.7 Business Rules vs Transactions

```text
COMPENSATION_RULE
≠
DOCTOR_COMPENSATION_TRANSACTION
```

The rule defines how compensation is calculated.

The transaction represents the resulting financial cost.

---

## 18.8 Baseline vs Scenario

```text
BASELINE DATA
must remain immutable.

SCENARIO DATA
represents simulated changes.
```

Scenario analysis must not overwrite the baseline model.

---

# 19. Recommended Implementation Layers

The logical model can be implemented in the following layers.

```text
DIMENSIONS
    ↓
PATIENT
CLINIC
SPECIALTY
DOCTOR
ROOM
SERVICE_TYPE
DATE_DIMENSION
COST_CATEGORY

OPERATIONAL EVENTS
    ↓
PATIENT_VISIT
DEMAND_EVENT
REFERRAL
PATIENT_DECISION
APPOINTMENT
VISIT
SERVICE_DELIVERY

OPERATIONAL STATE
    ↓
WAITING_LIST_ENTRY
DOCTOR_SCHEDULE
ROOM_SCHEDULE

CAPACITY & CONTRACT
    ↓
CAPACITY_FACT
NFZ_CONTRACT
NFZ_CONTRACT_AMENDMENT

FINANCE
    ↓
REVENUE_TRANSACTION
COMPENSATION_RULE
DOCTOR_COMPENSATION_TRANSACTION
OPERATING_COST

STRATEGY & SIMULATION
    ↓
SCENARIO
SCENARIO_PARAMETER
SCENARIO_RESOURCE_CHANGE
PATIENT_BEHAVIOR_RULE
RESOURCE_REALLOCATION
```

---

# 20. Entity Dictionary v1.1 Scope

Version 1.1 resolves the main modeling gaps identified during the audit of the Business Rules.

The model now explicitly distinguishes:

* master data,
* operational events,
* operational state,
* business rules,
* contracts,
* capacity,
* financial transactions,
* scenario configuration,
* and derived metrics.

The most important design decisions are:

```text
1. Demand is modeled as patient-level events.

2. Waiting list is modeled as a patient referral episode.

3. Waiting time is a derived metric.

4. Referral status and destination are separate attributes.

5. Appointment, Visit, and Service Delivery are separate concepts.

6. Operational Capacity and NFZ Funded Capacity are separate concepts.

7. Effective Capacity is derived.

8. NFZ Contract and Contract Amendment are separate entities.

9. Compensation Rules and Compensation Transactions are separate entities.

10. Baseline data is immutable relative to scenarios.

11. Date Dimension supports all temporal analysis.

12. Capacity Fact is aggregated at:
    Date × Clinic × Specialty × Service Type.

13. Delivered services are sourced from SERVICE_DELIVERY.

14. Financial transactions are traceable back to delivered services.

15. Patient behavior assumptions are configurable.
```

This version is intended to serve as the logical foundation for:

```text
05_Process_Flows.md
        ↓
06_ERD.md
        ↓
07_Generator_Architecture.md
```

The next design step should validate the relationships and cardinalities between these entities before implementation begins.
