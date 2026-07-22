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
SQL
       ↓
Tableau
```

The Entity Dictionary defines:

* logical entities,
* entity purpose,
* grain,
* primary keys,
* foreign keys,
* important attributes,
* relationships,
* business constraints,
* derived values,
* and modeling assumptions.

This document is a **logical data model specification**.

It is not yet a physical database schema.

SQL data types, indexes, partitioning, database constraints, and implementation-specific details will be defined later.

---

# 2. Modeling Principles

The data model follows the following principles:

1. Each entity must have a clearly defined business meaning.
2. Each entity must have an explicit grain.
3. Each transactional entity represents a specific business event or state.
4. Master data entities define relatively stable business concepts.
5. Fact entities capture measurable operational or financial activity.
6. Derived KPIs should not be duplicated as independent transactional facts unless required for performance.
7. Time-dependent business rules must be represented with effective dates where necessary.
8. Historical changes must not overwrite previous business states when historical analysis is required.
9. The model must support both operational analysis and BI reporting.
10. The model must support synthetic data generation with deterministic business logic.
11. The model must distinguish physical capacity from contractual NFZ capacity.
12. The model must distinguish process status from final business outcome.
13. Scenario configuration must be separated from actual operational events.
14. The model must preserve traceability from demand to service delivery and financial outcome.

---

# 3. Entity Classification

Entities are classified into the following groups:

## 3.1 Master Data

* PATIENT
* DOCTOR
* SPECIALTY
* CLINIC
* ROOM
* SERVICE_TYPE
* DATE_DIMENSION

## 3.2 Organizational and Availability Structures

* CLINIC_SPECIALTY
* DOCTOR_SPECIALTY

## 3.3 Demand and Referral

* DEMAND_EVENT
* PATIENT_VISIT
* REFERRAL
* PATIENT_DECISION

## 3.4 Scheduling and Waiting

* WAITING_LIST_ENTRY
* APPOINTMENT
* VISIT

## 3.5 Service Delivery

* SERVICE_DELIVERY
* SERVICE_PRICING

## 3.6 Capacity and Operations

* DOCTOR_SCHEDULE
* ROOM_SCHEDULE
* CAPACITY_FACT
* RESOURCE_REALLOCATION

## 3.7 NFZ and Contracting

* NFZ_CONTRACT
* NFZ_CONTRACT_AMENDMENT

## 3.8 Revenue and Finance

* REVENUE_TRANSACTION
* COMPENSATION_RULE
* DOCTOR_COMPENSATION_TRANSACTION
* OPERATING_COST

## 3.9 Simulation and Scenarios

* SCENARIO
* SCENARIO_RESOURCE_CHANGE
* PATIENT_BEHAVIOR_RULE

---

# 4. Master Data Entities

## 4.1 PATIENT

### Purpose

Represents an individual patient participating in the simulated healthcare system.

### Grain

One record per simulated patient.

### Primary Key

```text
patient_id
```

### Attributes

| Field             | Description                                                     |
| ----------------- | --------------------------------------------------------------- |
| patient_id        | Unique patient identifier                                       |
| date_of_birth     | Patient date of birth                                           |
| gender            | Simulated patient gender                                        |
| registration_date | Date patient entered the organization                           |
| patient_segment   | Optional analytical patient segment                             |
| active_flag       | Indicates whether patient is active in the simulated population |

### Relationships

```text
PATIENT
   ├── PATIENT_VISIT
   ├── REFERRAL
   ├── WAITING_LIST_ENTRY
   ├── APPOINTMENT
   ├── VISIT
   └── PATIENT_DECISION
```

---

## 4.2 DOCTOR

### Purpose

Represents a doctor providing healthcare services within the organization.

### Grain

One record per doctor.

### Primary Key

```text
doctor_id
```

### Foreign Keys

```text
specialty_id
```

### Attributes

| Field                 | Description                       |
| --------------------- | --------------------------------- |
| doctor_id             | Unique doctor identifier          |
| specialty_id          | Primary specialty                 |
| clinic_id             | Primary assigned clinic           |
| employment_type       | Employment classification         |
| compensation_model    | PERCENTAGE, HOURLY_RATE, or MIXED |
| employment_start_date | Start date                        |
| employment_end_date   | Optional end date                 |
| active_flag           | Current active status             |

### Modeling Note

For MVP, each doctor has one primary specialty.

If future requirements introduce multi-specialty doctors, the model may use `DOCTOR_SPECIALTY`.

---

## 4.3 SPECIALTY

### Purpose

Represents the complete catalog of medical specialties available in the simulated healthcare market.

### Grain

One record per specialty.

### Primary Key

```text
specialty_id
```

### Attributes

| Field              | Description                                       |
| ------------------ | ------------------------------------------------- |
| specialty_id       | Unique specialty identifier                       |
| specialty_name     | Specialty name                                    |
| specialty_category | Optional specialty grouping                       |
| active_flag        | Indicates whether specialty exists in the catalog |

### Modeling Note

Internal availability is **not stored directly on SPECIALTY**.

A specialty may exist in the broader catalog while being unavailable internally.

Internal availability is determined through:

```text
SPECIALTY
    ↓
CLINIC_SPECIALTY
    ↓
CLINIC
```

This allows availability to vary by clinic and over time.

---

## 4.4 CLINIC

### Purpose

Represents an operational healthcare clinic or organizational unit.

### Grain

One record per clinic.

### Primary Key

```text
clinic_id
```

### Attributes

| Field        | Description                |
| ------------ | -------------------------- |
| clinic_id    | Unique clinic identifier   |
| clinic_name  | Clinic name                |
| clinic_type  | POZ, PEDIATRIC, SPECIALIST |
| opening_date | Clinic opening date        |
| closing_date | Optional closing date      |
| active_flag  | Current operational status |

### Relationships

```text
CLINIC
   ├── CLINIC_SPECIALTY
   ├── DOCTOR
   ├── ROOM
   ├── DOCTOR_SCHEDULE
   ├── ROOM_SCHEDULE
   ├── DEMAND_EVENT
   ├── REFERRAL
   ├── CAPACITY_FACT
   ├── NFZ_CONTRACT
   ├── REVENUE_TRANSACTION
   └── OPERATING_COST
```

---

## 4.5 ROOM

### Purpose

Represents a physical room used to deliver healthcare services.

### Grain

One record per physical room.

### Primary Key

```text
room_id
```

### Foreign Keys

```text
clinic_id
```

### Attributes

| Field         | Description                |
| ------------- | -------------------------- |
| room_id       | Unique room identifier     |
| clinic_id     | Owning clinic              |
| room_type     | Room classification        |
| capacity_type | Type of services supported |
| active_flag   | Current operational status |

### Business Rules

A room cannot host two overlapping appointments.

---

## 4.6 SERVICE_TYPE

### Purpose

Defines the types of healthcare services delivered by the organization.

### Grain

One record per service type.

### Primary Key

```text
service_type_id
```

### Attributes

| Field                    | Description                  |
| ------------------------ | ---------------------------- |
| service_type_id          | Unique service identifier    |
| service_type_name        | Service name                 |
| service_category         | Service category             |
| visit_type               | FIRST_VISIT or FOLLOW_UP     |
| default_duration_minutes | Default appointment duration |
| active_flag              | Current status               |

### Business Rules

The model must distinguish at minimum:

```text
FIRST_VISIT
FOLLOW_UP
```

These may have different:

* durations,
* prices,
* NFZ limits,
* compensation rules,
* capacity characteristics.

---

## 4.7 DATE_DIMENSION

### Purpose

Provides a common calendar dimension for analytical reporting.

### Grain

One record per calendar date.

### Primary Key

```text
date_key
```

### Attributes

| Field         | Description                    |
| ------------- | ------------------------------ |
| date_key      | Date identifier                |
| calendar_date | Actual calendar date           |
| day_of_week   | Day number/name                |
| week_number   | Calendar week                  |
| month         | Month                          |
| quarter       | Quarter                        |
| year          | Year                           |
| is_weekend    | Weekend flag                   |
| season        | Optional season classification |

---

# 5. Organizational Availability Entities

## 5.1 CLINIC_SPECIALTY

### Purpose

Defines which specialties are provided internally by which clinics and during which periods.

### Grain

One record per clinic-specialty availability period.

### Primary Key

```text
clinic_specialty_id
```

### Foreign Keys

```text
clinic_id
specialty_id
```

### Attributes

| Field               | Description                    |
| ------------------- | ------------------------------ |
| clinic_specialty_id | Unique relationship identifier |
| clinic_id           | Clinic                         |
| specialty_id        | Specialty                      |
| effective_from      | Availability start             |
| effective_to        | Availability end               |
| is_active           | Current availability           |

### Business Rules

A specialty is considered internally available when an active record exists for the relevant date.

```text
SPECIALTY
    ↓
CLINIC_SPECIALTY
    ↓
CLINIC
```

This entity replaces the need for a static:

```text
SPECIALTY.available_internally
```

attribute.

---

## 5.2 DOCTOR_SPECIALTY

### Purpose

Optional bridge entity supporting doctors with multiple specialties.

### Grain

One record per doctor-specialty relationship.

### Primary Key

```text
doctor_specialty_id
```

### Foreign Keys

```text
doctor_id
specialty_id
```

### Attributes

| Field               | Description            |
| ------------------- | ---------------------- |
| doctor_specialty_id | Unique identifier      |
| doctor_id           | Doctor                 |
| specialty_id        | Specialty              |
| is_primary          | Primary specialty flag |
| effective_from      | Relationship start     |
| effective_to        | Relationship end       |

### MVP Note

For the initial MVP, `DOCTOR.specialty_id` may remain the primary specialty reference.

This entity exists to support future extension.

---

# 6. Demand and Referral Entities

## 6.1 DEMAND_EVENT

### Purpose

Represents a unit of simulated healthcare demand.

A demand event represents a patient's need or request for healthcare service that may ultimately be:

* fulfilled internally,
* placed into a waiting list,
* redirected externally,
* or abandoned.

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

| Field           | Description                                     |
| --------------- | ----------------------------------------------- |
| demand_event_id | Unique demand event                             |
| patient_id      | Patient generating demand                       |
| clinic_id       | Relevant clinic                                 |
| specialty_id    | Required specialty                              |
| service_type_id | Required service                                |
| demand_source   | POZ, PEDIATRIC, REFERRAL, FOLLOW_UP, DIAGNOSTIC |
| demand_date     | Date demand occurred                            |
| scenario_id     | Simulation scenario                             |

### Business Rule

Demand does not automatically equal completed service.

Possible outcomes include:

```text
Demand
   ├── Internal Service
   ├── Waiting
   ├── External
   └── Drop
```

---

## 6.2 PATIENT_VISIT

### Purpose

Represents an initial healthcare visit that may generate a specialist referral.

### Grain

One record per patient visit.

### Primary Key

```text
patient_visit_id
```

### Foreign Keys

```text
patient_id
doctor_id
clinic_id
```

### Attributes

| Field                  | Description                              |
| ---------------------- | ---------------------------------------- |
| patient_visit_id       | Unique visit identifier                  |
| patient_id             | Patient                                  |
| doctor_id              | Doctor                                   |
| clinic_id              | Clinic                                   |
| visit_date             | Date                                     |
| visit_type             | POZ or PEDIATRIC                         |
| referral_required_flag | Whether specialist referral is generated |

---

## 6.3 REFERRAL

### Purpose

Represents a referral from an originating provider to a target specialty.

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
target_clinic_id
demand_event_id
```

### Attributes

| Field               | Description                                       |
| ------------------- | ------------------------------------------------- |
| referral_id         | Unique referral                                   |
| patient_id          | Referred patient                                  |
| referring_doctor_id | Referring doctor                                  |
| referring_clinic_id | Originating clinic                                |
| target_specialty_id | Requested specialty                               |
| target_clinic_id    | Target internal clinic, if applicable             |
| demand_event_id     | Related demand                                    |
| referral_date       | Referral creation date                            |
| referral_status     | CREATED, WAITING, SCHEDULED, COMPLETED, CANCELLED |
| final_outcome       | INTERNAL, EXTERNAL, NOT_COMPLETED                 |
| outcome_date        | Date final outcome was determined                 |

### Business Rules

`referral_status` represents process lifecycle.

`final_outcome` represents the final business result.

These concepts must not be conflated.

Example:

```text
referral_status = COMPLETED
final_outcome = INTERNAL
```

or:

```text
referral_status = CANCELLED
final_outcome = NOT_COMPLETED
```

The final outcome categories are:

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

These support:

```text
Internal Capture Rate
External Leakage Rate
Non-Completion Rate
```

---

## 6.4 PATIENT_DECISION

### Purpose

Records the patient's behavioral decision in response to waiting time.

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
behavior_rule_id
```

### Attributes

| Field                            | Description                                |
| -------------------------------- | ------------------------------------------ |
| patient_decision_id              | Unique decision                            |
| referral_id                      | Related referral                           |
| patient_id                       | Patient                                    |
| decision_date                    | Decision date                              |
| waiting_time_at_decision_days    | Waiting time when decision occurred        |
| decision_type                    | WAIT, EXTERNAL, DROP                       |
| behavior_rule_id                 | Applied behavior rule                      |
| decision_probability_at_decision | Optional probability used at decision time |

### Business Rules

For an internally available specialty:

```text
WAIT
EXTERNAL
DROP
```

are valid decisions.

For an unavailable specialty:

```text
EXTERNAL
```

is the expected outcome.

The probability used for the decision should be traceable to `PATIENT_BEHAVIOR_RULE`.

---

# 7. Waiting and Scheduling Entities

## 7.1 WAITING_LIST_ENTRY

### Purpose

Represents a patient's waiting episode for a service.

### Grain

One record per waiting episode.

### Primary Key

```text
waiting_list_entry_id
```

### Foreign Keys

```text
patient_id
referral_id
clinic_id
specialty_id
service_type_id
```

### Attributes

| Field                 | Description                                   |
| --------------------- | --------------------------------------------- |
| waiting_list_entry_id | Unique waiting episode                        |
| patient_id            | Patient                                       |
| referral_id           | Referral                                      |
| clinic_id             | Clinic                                        |
| specialty_id          | Specialty                                     |
| service_type_id       | Service                                       |
| queue_entry_datetime  | Queue entry                                   |
| queue_exit_datetime   | Queue exit                                    |
| queue_status          | WAITING, SERVED, EXTERNAL, DROPPED, CANCELLED |
| waiting_days          | Derived waiting duration                      |

### Business Rules

The waiting list is dynamic.

Conceptually:

```text
New Demand
+
Existing Queue
-
Served Patients
-
External Leakage
-
Dropped Patients
=
New Queue
```

Daily or weekly queue simulation is supported.

---

## 7.2 APPOINTMENT

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
referral_id
service_type_id
```

### Attributes

| Field                    | Description                              |
| ------------------------ | ---------------------------------------- |
| appointment_id           | Unique appointment                       |
| patient_id               | Patient                                  |
| doctor_id                | Doctor                                   |
| clinic_id                | Clinic                                   |
| room_id                  | Room                                     |
| referral_id              | Referral, if applicable                  |
| service_type_id          | Service                                  |
| scheduled_start_datetime | Start                                    |
| scheduled_end_datetime   | End                                      |
| appointment_status       | SCHEDULED, COMPLETED, CANCELLED, NO_SHOW |

### Business Rules

The model must prevent:

* overlapping appointments for the same doctor,
* overlapping appointments in the same room.

---

## 7.3 VISIT

### Purpose

Represents an actual completed or attempted healthcare encounter.

### Grain

One record per patient encounter.

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
```

### Attributes

| Field          | Description                   |
| -------------- | ----------------------------- |
| visit_id       | Unique visit                  |
| appointment_id | Related appointment           |
| patient_id     | Patient                       |
| doctor_id      | Doctor                        |
| clinic_id      | Clinic                        |
| visit_date     | Actual date                   |
| visit_status   | COMPLETED, CANCELLED, NO_SHOW |

---

# 8. Service Delivery Entities

## 8.1 SERVICE_DELIVERY

### Purpose

Represents a healthcare service actually delivered.

### Grain

One record per delivered service.

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
```

### Attributes

| Field               | Description              |
| ------------------- | ------------------------ |
| service_delivery_id | Unique delivered service |
| visit_id            | Related visit            |
| patient_id          | Patient                  |
| doctor_id           | Doctor                   |
| clinic_id           | Clinic                   |
| specialty_id        | Specialty                |
| service_type_id     | Service                  |
| service_date        | Delivery date            |
| payer_type          | NFZ or COMMERCIAL        |
| delivery_status     | DELIVERED, VOIDED        |

---

## 8.2 SERVICE_PRICING

### Purpose

Defines the price or economic value of a service for a specific payer and effective period.

### Grain

One record per service type, payer type, and effective pricing period.

### Primary Key

```text
service_pricing_id
```

### Foreign Keys

```text
service_type_id
```

### Attributes

| Field              | Description           |
| ------------------ | --------------------- |
| service_pricing_id | Unique pricing record |
| service_type_id    | Service               |
| payer_type         | NFZ or COMMERCIAL     |
| price_amount       | Service price/value   |
| effective_from     | Pricing start         |
| effective_to       | Pricing end           |
| active_flag        | Current status        |

### Business Rules

Pricing may differ by:

* service type,
* payer type,
* time period.

Example:

```text
FIRST_VISIT
    NFZ        → 250 PLN
    COMMERCIAL → 350 PLN

FOLLOW_UP
    NFZ        → 150 PLN
    COMMERCIAL → 220 PLN
```

The applicable price is determined by service date and payer type.

---

# 9. Capacity and Operational Entities

## 9.1 DOCTOR_SCHEDULE

### Purpose

Defines doctor availability and working capacity.

### Grain

One record per doctor per schedule period.

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

| Field              | Description     |
| ------------------ | --------------- |
| doctor_schedule_id | Unique schedule |
| doctor_id          | Doctor          |
| clinic_id          | Clinic          |
| date_key           | Date            |
| available_hours    | Available hours |
| working_flag       | Working status  |

---

## 9.2 ROOM_SCHEDULE

### Purpose

Defines room availability.

### Grain

One record per room per schedule period.

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

| Field            | Description          |
| ---------------- | -------------------- |
| room_schedule_id | Unique schedule      |
| room_id          | Room                 |
| clinic_id        | Clinic               |
| date_key         | Date                 |
| available_hours  | Available room hours |
| available_flag   | Availability         |

---

## 9.3 CAPACITY_FACT

### Purpose

Stores calculated operational and contractual capacity for a service line.

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
nfz_contract_id
scenario_id
```

### Attributes

| Field                     | Description                             |
| ------------------------- | --------------------------------------- |
| capacity_fact_id          | Unique capacity record                  |
| date_key                  | Capacity date                           |
| clinic_id                 | Clinic                                  |
| specialty_id              | Specialty                               |
| service_type_id           | Service                                 |
| nfz_contract_id           | Applicable NFZ contract                 |
| scenario_id               | Scenario                                |
| operational_capacity      | Physical capacity                       |
| nfz_funded_capacity       | Contractual funded capacity             |
| effective_capacity        | Minimum of operational and NFZ capacity |
| actual_delivered_services | Actual delivered services               |

### Business Rules

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

Operational capacity is constrained by:

* doctors,
* doctor hours,
* doctor schedules,
* rooms,
* room availability,
* appointment duration.

NFZ funded capacity is a contractual constraint.

The model must distinguish:

```text
Operational Capacity
Funded Capacity
Effective Capacity
Actual Delivered Services
```

---

# 10. NFZ Contracting Entities

## 10.1 NFZ_CONTRACT

### Purpose

Represents a simulated NFZ contract for a specific service line and contractual period.

### Grain

One record per clinic-specialty-service contract period.

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

| Field                      | Description                |
| -------------------------- | -------------------------- |
| nfz_contract_id            | Unique contract            |
| clinic_id                  | Clinic                     |
| specialty_id               | Specialty                  |
| service_type_id            | Service                    |
| contract_start_date        | Contract start             |
| contract_end_date          | Contract end               |
| original_contracted_volume | Original contracted volume |
| volume_period              | MONTH, QUARTER, YEAR       |
| contract_status            | ACTIVE, EXPIRED, CANCELLED |
| active_flag                | Current status             |

### Business Rules

The contract defines the maximum funded volume for the applicable period.

The model should support:

```text
Contracted Volume
Delivered Volume
Remaining Contract Capacity
Contract Utilization
Overperformance
Underperformance
```

The contract should be analyzable by:

* specialty,
* clinic,
* service type,
* quarter,
* year.

---

## 10.2 NFZ_CONTRACT_AMENDMENT

### Purpose

Represents a change to an existing NFZ contract.

### Grain

One record per contract amendment.

### Primary Key

```text
nfz_contract_amendment_id
```

### Foreign Keys

```text
nfz_contract_id
```

### Attributes

| Field                      | Description            |
| -------------------------- | ---------------------- |
| nfz_contract_amendment_id  | Unique amendment       |
| nfz_contract_id            | Contract               |
| amendment_date             | Amendment date         |
| effective_from             | Effective date         |
| previous_contracted_volume | Previous volume        |
| new_contracted_volume      | New volume             |
| change_amount              | Difference             |
| amendment_reason           | Reason                 |
| scenario_id                | Scenario, if simulated |

### Business Rules

An amendment may:

* increase funded capacity,
* decrease funded capacity,
* change service volume,
* change effective capacity.

Historical contract states must remain reconstructable.

---

# 11. Revenue and Financial Entities

## 11.1 REVENUE_TRANSACTION

### Purpose

Represents economic revenue generated by a delivered healthcare service.

### Grain

One record per revenue-generating service delivery.

### Primary Key

```text
revenue_transaction_id
```

### Foreign Keys

```text
service_delivery_id
service_pricing_id
patient_id
doctor_id
clinic_id
```

### Attributes

| Field                  | Description                |
| ---------------------- | -------------------------- |
| revenue_transaction_id | Unique revenue transaction |
| service_delivery_id    | Delivered service          |
| service_pricing_id     | Applied pricing            |
| patient_id             | Patient                    |
| doctor_id              | Doctor                     |
| clinic_id              | Clinic                     |
| transaction_date       | Date                       |
| gross_service_revenue  | Gross service revenue      |
| payer_type             | NFZ or COMMERCIAL          |

### Business Rules

The model separates:

```text
Total Service Revenue
Doctor Compensation
Clinic Revenue
Operating Costs
Net Clinic Result
```

Clinic revenue is derived from:

```text
Clinic Revenue
=
Gross Service Revenue
-
Doctor Compensation
```

unless a future business rule explicitly introduces additional revenue allocation components.

---

## 11.2 COMPENSATION_RULE

### Purpose

Defines the compensation logic applicable to a doctor.

### Grain

One record per doctor compensation rule and effective period.

### Primary Key

```text
compensation_rule_id
```

### Foreign Keys

```text
doctor_id
```

### Attributes

| Field                | Description                    |
| -------------------- | ------------------------------ |
| compensation_rule_id | Unique rule                    |
| doctor_id            | Doctor                         |
| compensation_model   | PERCENTAGE, HOURLY_RATE, MIXED |
| percentage_rate      | Revenue percentage             |
| hourly_rate          | Hourly compensation            |
| fixed_amount         | Optional fixed component       |
| effective_from       | Rule start                     |
| effective_to         | Rule end                       |

### Business Rules

```text
PERCENTAGE
→ percentage_rate

HOURLY_RATE
→ hourly_rate

MIXED
→ hourly_rate + percentage_rate
```

Only attributes relevant to the selected model should be populated.

---

## 11.3 DOCTOR_COMPENSATION_TRANSACTION

### Purpose

Represents compensation generated for a doctor.

### Grain

One record per compensation transaction.

### Primary Key

```text
doctor_compensation_transaction_id
```

### Foreign Keys

```text
doctor_id
service_delivery_id
compensation_rule_id
```

### Attributes

| Field                              | Description         |
| ---------------------------------- | ------------------- |
| doctor_compensation_transaction_id | Unique transaction  |
| doctor_id                          | Doctor              |
| service_delivery_id                | Service             |
| compensation_rule_id               | Applied rule        |
| transaction_date                   | Date                |
| compensation_amount                | Doctor compensation |

---

## 11.4 OPERATING_COST

### Purpose

Represents operating costs incurred by the organization.

### Grain

One record per cost transaction or cost allocation period.

### Primary Key

```text
operating_cost_id
```

### Foreign Keys

```text
clinic_id
date_key
scenario_id
```

### Attributes

| Field             | Description                                     |
| ----------------- | ----------------------------------------------- |
| operating_cost_id | Unique cost                                     |
| clinic_id         | Clinic                                          |
| date_key          | Cost date                                       |
| cost_category     | RENT, UTILITIES, INSURANCE, DEPRECIATION, OTHER |
| cost_period_start | Cost period start                               |
| cost_period_end   | Cost period end                                 |
| cost_amount       | Cost amount                                     |
| scenario_id       | Scenario                                        |

### Business Rules

Costs may be modeled as:

* daily,
* monthly,
* quarterly,
* annual,
* or one-time transactions.

The period must be explicit when a cost represents an allocation over time.

---

# 12. Resource Reallocation

## 12.1 RESOURCE_REALLOCATION

### Purpose

Represents an operational decision to reallocate resources between clinics.

### Grain

One record per resource reallocation event.

### Primary Key

```text
resource_reallocation_id
```

### Foreign Keys

```text
source_clinic_id
target_clinic_id
doctor_id
room_id
scenario_id
```

### Attributes

| Field                    | Description                                      |
| ------------------------ | ------------------------------------------------ |
| resource_reallocation_id | Unique event                                     |
| reallocation_date        | Decision date                                    |
| source_clinic_id         | Origin clinic                                    |
| target_clinic_id         | Destination clinic                               |
| resource_type            | DOCTOR, DOCTOR_HOURS, ROOM, OPERATIONAL_CAPACITY |
| doctor_id                | Doctor, when applicable                          |
| room_id                  | Room, when applicable                            |
| source_quantity          | Quantity removed from source                     |
| target_quantity          | Quantity added to target                         |
| reason                   | Operational rationale                            |
| scenario_id              | Scenario                                         |

### Business Rules

Resource reallocation is an **operational event**.

It must affect future capacity.

The model should preserve:

```text
Current State
```

and:

```text
Post-Reallocation State
```

This enables before-and-after analysis.

For `DOCTOR_HOURS`, quantities represent hours.

For `ROOM`, quantities represent room units.

For `OPERATIONAL_CAPACITY`, quantities represent capacity units.

---

# 13. Scenario Entities

## 13.1 SCENARIO

### Purpose

Represents a simulation scenario used for What-if analysis.

### Grain

One record per scenario.

### Primary Key

```text
scenario_id
```

### Foreign Keys

```text
parent_scenario_id
```

### Attributes

| Field              | Description                    |
| ------------------ | ------------------------------ |
| scenario_id        | Unique scenario                |
| scenario_name      | Scenario name                  |
| scenario_type      | BASELINE or WHAT_IF            |
| parent_scenario_id | Parent scenario, if applicable |
| is_baseline        | Baseline flag                  |
| description        | Scenario description           |
| effective_from     | Scenario start                 |
| effective_to       | Scenario end                   |
| created_date       | Creation date                  |

### Business Rules

The baseline scenario represents the reference operating model.

What-if scenarios may inherit from the baseline or another scenario.

Example:

```text
BASELINE
   ├── NEW_CLINIC
   ├── ADD_DOCTORS
   └── INCREASE_NFZ_CONTRACT
```

---

## 13.2 SCENARIO_RESOURCE_CHANGE

### Purpose

Represents a simulated change in resources under a scenario.

### Grain

One record per scenario resource change.

### Primary Key

```text
scenario_resource_change_id
```

### Foreign Keys

```text
scenario_id
clinic_id
doctor_id
room_id
```

### Attributes

| Field                       | Description                          |
| --------------------------- | ------------------------------------ |
| scenario_resource_change_id | Unique change                        |
| scenario_id                 | Scenario                             |
| clinic_id                   | Affected clinic                      |
| resource_type               | DOCTOR, DOCTOR_HOURS, ROOM, CAPACITY |
| doctor_id                   | Doctor, if applicable                |
| room_id                     | Room, if applicable                  |
| quantity_change             | Change amount                        |
| effective_from              | Effective date                       |
| effective_to                | Optional end date                    |

### Business Rules

`SCENARIO_RESOURCE_CHANGE` represents a **simulation configuration or scenario assumption**.

It is different from:

```text
RESOURCE_REALLOCATION
```

which represents an operational event in the simulated organization.

---

# 14. Patient Behavior

## 14.1 PATIENT_BEHAVIOR_RULE

### Purpose

Defines configurable probabilistic patient behavior in response to waiting time.

### Grain

One record per waiting-time interval and behavior rule version.

### Primary Key

```text
behavior_rule_id
```

### Attributes

| Field                 | Description             |
| --------------------- | ----------------------- |
| behavior_rule_id      | Unique rule             |
| waiting_time_min_days | Minimum waiting time    |
| waiting_time_max_days | Maximum waiting time    |
| probability_wait      | Probability of WAIT     |
| probability_external  | Probability of EXTERNAL |
| probability_drop      | Probability of DROP     |
| effective_from        | Rule start              |
| effective_to          | Rule end                |

### Business Rules

The probabilities must satisfy:

```text
probability_wait
+
probability_external
+
probability_drop
=
1
```

Example configurable assumptions:

```text
0–7 days       → 5% leakage
8–30 days      → 10%
31–60 days     → 20%
61–90 days     → 35%
91–180 days    → 55%
180+ days      → 75%
```

These are synthetic simulation assumptions and not real-world clinical statistics.

The parameters must be configurable without changing generator logic.

---

# 15. Core Entity Relationships

The primary operational flow is:

```text
PATIENT
    ↓
PATIENT_VISIT
    ↓
DEMAND_EVENT
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
    ↓
REVENUE_TRANSACTION
```

The capacity flow is:

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
OPERATIONAL CAPACITY
    ↓
NFZ_CONTRACT
    ↓
NFZ FUNDED CAPACITY
    ↓
EFFECTIVE CAPACITY
```

The availability flow is:

```text
SPECIALTY
    ↓
CLINIC_SPECIALTY
    ↓
CLINIC
```

The financial flow is:

```text
SERVICE_DELIVERY
       ↓
SERVICE_PRICING
       ↓
REVENUE_TRANSACTION
       ↓
GROSS SERVICE REVENUE
       ↓
COMPENSATION_RULE
       ↓
DOCTOR_COMPENSATION_TRANSACTION
       ↓
CLINIC REVENUE
       ↓
OPERATING_COST
       ↓
NET CLINIC RESULT
```

The scenario flow is:

```text
BASELINE SCENARIO
       ↓
SCENARIO
       ↓
SCENARIO_RESOURCE_CHANGE
       ↓
CAPACITY / DEMAND / FINANCIAL IMPACT
```

The operational reallocation flow is:

```text
RESOURCE_REALLOCATION
       ↓
POST-REALLOCATION RESOURCE STATE
       ↓
FUTURE CAPACITY
       ↓
WAITING TIME
       ↓
PATIENT BEHAVIOR
       ↓
SERVICE DELIVERY
```

---

# 16. Key Derived Metrics

The following values should generally be derived rather than stored as independent transactional entities.

## 16.1 Internal Capture Rate

```text
Internal Capture Rate
=
Internal Referrals
/
Total Referrals
```

## 16.2 External Leakage Rate

```text
External Leakage Rate
=
External Referrals
/
Total Referrals
```

## 16.3 Referral Non-Completion Rate

```text
Non-Completion Rate
=
Not Completed Referrals
/
Total Referrals
```

## 16.4 Effective Capacity

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

## 16.5 Contract Utilization

```text
Contract Utilization
=
Delivered Funded Services
/
Contracted Services
```

## 16.6 Remaining Contract Capacity

```text
Remaining Contract Capacity
=
Contracted Volume
-
Delivered Funded Services
```

## 16.7 Clinic Revenue

```text
Clinic Revenue
=
Gross Service Revenue
-
Doctor Compensation
```

## 16.8 Net Clinic Result

```text
Net Clinic Result
=
Clinic Revenue
-
Operating Costs
```

---

# 17. Critical Business Constraints

The following constraints must be respected by the data generator and later database implementation.

## 17.1 Appointment Conflicts

A doctor cannot have overlapping appointments.

A room cannot have overlapping appointments.

---

## 17.2 Referral Outcome

Each completed referral pathway must ultimately resolve to one of:

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

---

## 17.3 Specialty Availability

Internal specialty availability must be determined using the applicable `CLINIC_SPECIALTY` record for the relevant date.

---

## 17.4 Capacity

```text
Effective Capacity
≤ Operational Capacity
```

and:

```text
Effective Capacity
≤ NFZ Funded Capacity
```

---

## 17.5 Contract Volume

Delivered funded services should be traceable to the applicable NFZ contract and contractual period.

---

## 17.6 Pricing

Each revenue-generating service must use the pricing rule applicable to:

```text
Service Type
+
Payer Type
+
Service Date
```

---

## 17.7 Compensation

The compensation transaction must reference the compensation rule applicable to the doctor and service date.

---

## 17.8 Patient Behavior

Behavior probabilities must sum to:

```text
1.0
```

for every active behavior rule.

---

## 17.9 Scenario Separation

Scenario configuration must not overwrite baseline data.

Baseline and What-if scenarios must remain analytically distinguishable.

---

# 18. Entity Dependency Overview

```text
SPECIALTY
    │
    └── CLINIC_SPECIALTY ── CLINIC
                              │
                              ├── DOCTOR
                              │     │
                              │     ├── DOCTOR_SCHEDULE
                              │     └── COMPENSATION_RULE
                              │
                              ├── ROOM
                              │     └── ROOM_SCHEDULE
                              │
                              ├── NFZ_CONTRACT
                              │     └── NFZ_CONTRACT_AMENDMENT
                              │
                              └── CAPACITY_FACT

PATIENT
    │
    ├── PATIENT_VISIT
    │       └── REFERRAL
    │              ├── PATIENT_DECISION
    │              └── WAITING_LIST_ENTRY
    │                       └── APPOINTMENT
    │                              └── VISIT
    │                                   └── SERVICE_DELIVERY
    │                                          ├── SERVICE_PRICING
    │                                          └── REVENUE_TRANSACTION
    │
    └── DEMAND_EVENT


SERVICE_DELIVERY
    │
    ├── REVENUE_TRANSACTION
    │
    └── DOCTOR_COMPENSATION_TRANSACTION
               │
               └── COMPENSATION_RULE


SCENARIO
    │
    ├── SCENARIO_RESOURCE_CHANGE
    ├── RESOURCE_REALLOCATION
    ├── CAPACITY_FACT
    ├── DEMAND_EVENT
    ├── NFZ_CONTRACT_AMENDMENT
    └── OPERATING_COST
```

---

# 19. V1.2 Change Summary

Version `1.2` introduces the following key modeling changes.

### Added

```text
CLINIC_SPECIALTY
SERVICE_PRICING
REFERRAL.final_outcome
NFZ_CONTRACT.volume_period
NFZ_CONTRACT ↔ CAPACITY_FACT relationship
SCENARIO.parent_scenario_id
RESOURCE_REALLOCATION.source_quantity
RESOURCE_REALLOCATION.target_quantity
OPERATING_COST.cost_period_start
OPERATING_COST.cost_period_end
```

### Changed

```text
SPECIALTY.available_internally
```

was removed from the conceptual model.

Internal availability is now represented by:

```text
CLINIC_SPECIALTY
```

Referral lifecycle and referral outcome are explicitly separated:

```text
referral_status
        +
final_outcome
```

Scenario changes and operational resource reallocations are explicitly separated:

```text
SCENARIO_RESOURCE_CHANGE
        ≠
RESOURCE_REALLOCATION
```

Pricing is now explicitly modeled:

```text
SERVICE_TYPE
    ↓
SERVICE_PRICING
    ↓
SERVICE_DELIVERY
    ↓
REVENUE_TRANSACTION
```

NFZ contractual capacity is explicitly connected to the capacity fact:

```text
NFZ_CONTRACT
    ↓
CAPACITY_FACT
```

---

# 20. Version Status

**Version:** 1.2

**Status:** Logical model — ready for Process Flow design

**Next document:**

```text
05_Process_Flows.md
```

The next stage should define the chronological and causal processes connecting the entities described in this document.

The recommended process flows are:

1. Patient Entry and Demand Generation
2. Primary Care Visit and Referral Generation
3. Referral Routing and Specialty Availability
4. Waiting List and Capacity Management
5. Patient Leakage and Drop-off
6. Appointment Scheduling
7. Service Delivery
8. NFZ Contract Utilization
9. Revenue and Doctor Compensation
10. Resource Reallocation
11. NFZ Contract Amendment
12. Scenario / What-if Simulation

The Process Flow document should be validated against this Entity Dictionary before the ERD is created.
