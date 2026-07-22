# 04 — Entity Dictionary

**Version:** 1.4
**Status:** Draft — Ready for Process Flow Design
**Last Updated:** 2026-07-22

---

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

* entities,
* attributes,
* primary keys,
* foreign keys,
* logical relationships,
* data grain,
* business meaning,
* important constraints,
* and derived metrics.

This document represents the logical model.

It does not yet define:

* physical database implementation,
* SQL DDL,
* indexing strategy,
* partitioning,
* exact Python class structure,
* or final Tableau data sources.

---

# 2. Modeling Principles

The data model follows several core principles.

## 2.1 Separate master data from transactional data

Master and reference entities describe relatively stable business objects:

* PATIENT
* DOCTOR
* CLINIC
* SPECIALTY
* SERVICE_TYPE
* ROOM

Transactional entities represent events:

* PATIENT_VISIT
* REFERRAL
* APPOINTMENT
* SERVICE_DELIVERY
* REVENUE_TRANSACTION
* DOCTOR_COMPENSATION_TRANSACTION
* OPERATING_COST_TRANSACTION

---

## 2.2 Separate operational events from analytical snapshots

Operational entities describe what happened.

Analytical fact entities summarize or snapshot operational conditions.

Example:

```text
DOCTOR
   ↓
DOCTOR_SCHEDULE
   ↓
ROOM
   ↓
APPOINTMENT
   ↓
SERVICE_DELIVERY
   ↓
CAPACITY_FACT
```

`CAPACITY_FACT` is therefore an analytical snapshot and should not be treated as the primary source of operational capacity.

---

## 2.3 Preserve historical business states

Historical data must remain auditable.

For entities where business rules can change over time, effective dating should be used:

```text
effective_from
effective_to
is_active
```

This applies especially to:

* SERVICE_PRICING
* DOCTOR_COMPENSATION_RULE
* NFZ_CONTRACT
* ROOM
* DOCTOR_SCHEDULE

---

## 2.4 Separate actual events from hypothetical scenarios

Actual operational changes:

```text
RESOURCE_REALLOCATION
```

Scenario assumptions:

```text
SCENARIO_RESOURCE_CHANGE
```

Scenario entities must not overwrite actual operational data.

---

# 3. Entity Overview

The logical model contains the following entity groups.

## 3.1 Core Master Data

* PATIENT
* DOCTOR
* CLINIC
* SPECIALTY
* SERVICE_TYPE
* ROOM

## 3.2 Operational Scheduling

* DOCTOR_SCHEDULE
* PATIENT_VISIT
* REFERRAL
* REFERRAL_OUTCOME
* PATIENT_DECISION
* DEMAND_EVENT
* WAITING_LIST_ENTRY
* APPOINTMENT
* SERVICE_DELIVERY

## 3.3 Capacity and Operations

* CAPACITY_FACT
* NFZ_CONTRACT
* RESOURCE_REALLOCATION

## 3.4 Financial

* SERVICE_PRICING
* REVENUE_TRANSACTION
* DOCTOR_COMPENSATION_RULE
* DOCTOR_COMPENSATION_TRANSACTION
* OPERATING_COST_TRANSACTION

## 3.5 Scenario Modeling

* SCENARIO
* SCENARIO_RESOURCE_CHANGE

---

# 4. Core Master Data

---

## 4.1 PATIENT

### Purpose

Represents a patient registered in the healthcare system.

### Grain

One row per patient.

### Primary Key

```text
patient_id
```

### Attributes

| Attribute              | Type           | Key | Description                          |
| ---------------------- | -------------- | --- | ------------------------------------ |
| patient_id             | INTEGER / UUID | PK  | Unique patient identifier            |
| date_of_birth          | DATE           |     | Patient date of birth                |
| gender                 | VARCHAR        |     | Patient gender                       |
| registration_date      | DATE           |     | Date patient entered the system      |
| residence_region       | VARCHAR        |     | Geographic region                    |
| insurance_type         | VARCHAR        |     | Insurance category                   |
| socioeconomic_segment  | VARCHAR        |     | Synthetic socioeconomic segment      |
| chronic_condition_flag | BOOLEAN        |     | Indicates relevant chronic condition |
| created_at             | TIMESTAMP      |     | Record creation timestamp            |

### Business Rules

* `patient_id` must be unique.
* Patient age should be derived from `date_of_birth`.
* Patient demographic attributes should not be used to identify real individuals.
* All generated patients are synthetic.

---

## 4.2 DOCTOR

### Purpose

Represents a healthcare professional providing services.

### Grain

One row per doctor.

### Primary Key

```text
doctor_id
```

### Attributes

| Attribute             | Type           | Key | Description                      |
| --------------------- | -------------- | --- | -------------------------------- |
| doctor_id             | INTEGER / UUID | PK  | Unique doctor identifier         |
| specialty_id          | INTEGER / UUID | FK  | Primary specialty                |
| doctor_name           | VARCHAR        |     | Synthetic doctor name            |
| employment_type       | VARCHAR        |     | Employment model                 |
| compensation_model    | VARCHAR        |     | PERCENTAGE / HOURLY_RATE / MIXED |
| employment_start_date | DATE           |     | Employment start                 |
| employment_end_date   | DATE           |     | Employment end                   |
| is_active             | BOOLEAN        |     | Current active status            |

### Relationships

```text
DOCTOR.specialty_id
→ SPECIALTY.specialty_id
```

### Business Rules

* A doctor must have a primary specialty.
* A doctor may provide services in one or more clinics.
* Doctor availability is modeled separately through `DOCTOR_SCHEDULE`.
* Compensation rules are modeled separately through `DOCTOR_COMPENSATION_RULE`.

---

## 4.3 CLINIC

### Purpose

Represents a healthcare clinic or operational facility.

### Grain

One row per clinic.

### Primary Key

```text
clinic_id
```

### Attributes

| Attribute    | Type           | Key | Description                |
| ------------ | -------------- | --- | -------------------------- |
| clinic_id    | INTEGER / UUID | PK  | Unique clinic identifier   |
| clinic_name  | VARCHAR        |     | Clinic name                |
| city         | VARCHAR        |     | City                       |
| region       | VARCHAR        |     | Geographic region          |
| clinic_type  | VARCHAR        |     | Clinic category            |
| opening_date | DATE           |     | Operational start          |
| closing_date | DATE           |     | Operational end            |
| is_active    | BOOLEAN        |     | Current operational status |

### Relationships

```text
CLINIC
   ├── CLINIC_SPECIALTY
   ├── ROOM
   ├── PATIENT_VISIT
   ├── APPOINTMENT
   ├── SERVICE_DELIVERY
   ├── REVENUE_TRANSACTION
   └── OPERATING_COST_TRANSACTION
```

---

## 4.4 SPECIALTY

### Purpose

Represents a medical specialty.

### Grain

One row per specialty.

### Primary Key

```text
specialty_id
```

### Attributes

| Attribute          | Type           | Key | Description        |
| ------------------ | -------------- | --- | ------------------ |
| specialty_id       | INTEGER / UUID | PK  | Unique specialty   |
| specialty_name     | VARCHAR        |     | Specialty name     |
| specialty_category | VARCHAR        |     | Specialty category |
| is_active          | BOOLEAN        |     | Active status      |

---

## 4.5 CLINIC_SPECIALTY

### Purpose

Represents the specialties offered by each clinic.

### Grain

One row per clinic-specialty combination.

### Primary Key

```text
clinic_specialty_id
```

### Foreign Keys

```text
clinic_id
→ CLINIC.clinic_id

specialty_id
→ SPECIALTY.specialty_id
```

### Attributes

| Attribute           | Type           | Key | Description          |
| ------------------- | -------------- | --- | -------------------- |
| clinic_specialty_id | INTEGER / UUID | PK  | Unique relationship  |
| clinic_id           | INTEGER / UUID | FK  | Clinic               |
| specialty_id        | INTEGER / UUID | FK  | Specialty            |
| effective_from      | DATE           |     | Start date           |
| effective_to        | DATE           |     | End date             |
| is_active           | BOOLEAN        |     | Current availability |

### Business Rules

A clinic may offer multiple specialties.

This model is intentionally more flexible than assuming one specialty per clinic.

---

## 4.6 SERVICE_TYPE

### Purpose

Represents a billable healthcare service.

### Grain

One row per service type.

### Primary Key

```text
service_type_id
```

### Attributes

| Attribute                 | Type           | Key | Description                   |
| ------------------------- | -------------- | --- | ----------------------------- |
| service_type_id           | INTEGER / UUID | PK  | Unique service                |
| service_name              | VARCHAR        |     | Service name                  |
| specialty_id              | INTEGER / UUID | FK  | Associated specialty          |
| standard_duration_minutes | INTEGER        |     | Standard appointment duration |
| service_category          | VARCHAR        |     | Service category              |
| is_nfz_eligible           | BOOLEAN        |     | Indicates NFZ eligibility     |
| is_active                 | BOOLEAN        |     | Active status                 |

### Business Rules

Service duration is used when calculating:

* appointment duration,
* room utilization,
* doctor capacity,
* operational capacity.

---

# 5. Operational Infrastructure

---

## 5.1 ROOM

### Purpose

Represents a physical room where appointments and services can be delivered.

### Grain

One row per physical room.

### Primary Key

```text
room_id
```

### Foreign Keys

```text
clinic_id
→ CLINIC.clinic_id
```

### Attributes

| Attribute      | Type           | Key | Description                           |
| -------------- | -------------- | --- | ------------------------------------- |
| room_id        | INTEGER / UUID | PK  | Unique room                           |
| clinic_id      | INTEGER / UUID | FK  | Owning clinic                         |
| room_code      | VARCHAR        |     | Internal room identifier              |
| room_type      | VARCHAR        |     | Consultation / Procedure / Diagnostic |
| capacity_units | INTEGER        |     | Number of simultaneous service units  |
| effective_from | DATE           |     | Operational start                     |
| effective_to   | DATE           |     | Operational end                       |
| is_active      | BOOLEAN        |     | Current operational status            |

### Business Rules

* A room belongs to one clinic.
* A room cannot host overlapping appointments when `capacity_units = 1`.
* Room availability contributes to operational capacity.
* A room may be unavailable outside its effective operating period.
* Room availability must be considered when calculating effective capacity.

### Key Constraint

For a standard room:

```text
No overlapping APPOINTMENT records
for the same room_id.
```

---

## 5.2 DOCTOR_SCHEDULE

### Purpose

Represents doctor availability and scheduled working capacity.

### Grain

One row per doctor, clinic, specialty and schedule date.

### Primary Key

```text
doctor_schedule_id
```

### Foreign Keys

```text
doctor_id
→ DOCTOR.doctor_id

clinic_id
→ CLINIC.clinic_id

specialty_id
→ SPECIALTY.specialty_id
```

### Attributes

| Attribute          | Type           | Key | Description                     |
| ------------------ | -------------- | --- | ------------------------------- |
| doctor_schedule_id | INTEGER / UUID | PK  | Unique schedule record          |
| doctor_id          | INTEGER / UUID | FK  | Doctor                          |
| clinic_id          | INTEGER / UUID | FK  | Clinic                          |
| specialty_id       | INTEGER / UUID | FK  | Specialty                       |
| schedule_date      | DATE           |     | Date of availability            |
| start_time         | TIME           |     | Schedule start                  |
| end_time           | TIME           |     | Schedule end                    |
| available_hours    | DECIMAL        |     | Total available hours           |
| booked_hours       | DECIMAL        |     | Hours allocated to appointments |
| leave_flag         | BOOLEAN        |     | Doctor unavailable              |
| is_available       | BOOLEAN        |     | Availability status             |

### Business Rules

Doctor capacity is derived from schedule availability.

Conceptually:

```text
Available Doctor Hours
=
Scheduled Hours
-
Leave / Unavailability
```

Doctor schedules are one of the source inputs for:

```text
Operational Capacity
```

---

# 6. Patient Journey and Referral

---

## 6.1 PATIENT_VISIT

### Purpose

Represents an interaction between a patient and a healthcare provider.

### Grain

One row per patient visit.

### Primary Key

```text
visit_id
```

### Foreign Keys

```text
patient_id
→ PATIENT.patient_id

doctor_id
→ DOCTOR.doctor_id

clinic_id
→ CLINIC.clinic_id
```

### Attributes

| Attribute  | Type           | Key | Description   |
| ---------- | -------------- | --- | ------------- |
| visit_id   | INTEGER / UUID | PK  | Unique visit  |
| patient_id | INTEGER / UUID | FK  | Patient       |
| doctor_id  | INTEGER / UUID | FK  | Doctor        |
| clinic_id  | INTEGER / UUID | FK  | Clinic        |
| visit_date | DATE           |     | Visit date    |
| visit_type | VARCHAR        |     | Visit type    |
| outcome    | VARCHAR        |     | Visit outcome |

---

## 6.2 REFERRAL

### Purpose

Represents a referral generated from a patient interaction.

### Grain

One row per referral.

### Primary Key

```text
referral_id
```

### Foreign Keys

```text
patient_id
→ PATIENT.patient_id

referring_doctor_id
→ DOCTOR.doctor_id

referring_clinic_id
→ CLINIC.clinic_id

target_specialty_id
→ SPECIALTY.specialty_id

source_visit_id
→ PATIENT_VISIT.visit_id
```

### Attributes

| Attribute                | Type           | Key | Description                         |
| ------------------------ | -------------- | --- | ----------------------------------- |
| referral_id              | INTEGER / UUID | PK  | Unique referral                     |
| patient_id               | INTEGER / UUID | FK  | Patient                             |
| referring_doctor_id      | INTEGER / UUID | FK  | Referring doctor                    |
| referring_clinic_id      | INTEGER / UUID | FK  | Referring clinic                    |
| target_specialty_id      | INTEGER / UUID | FK  | Requested specialty                 |
| source_visit_id          | INTEGER / UUID | FK  | Source visit                        |
| referral_date            | DATE           |     | Referral date                       |
| initial_destination_type | VARCHAR        |     | INTERNAL / EXTERNAL                 |
| referral_status          | VARCHAR        |     | Current referral status             |
| final_outcome            | VARCHAR        |     | INTERNAL / EXTERNAL / NOT_COMPLETED |

### Business Rules

A referral must identify:

* patient,
* referring doctor,
* referring clinic,
* target specialty,
* referral date,
* referral status.

`initial_destination_type` represents the initial routing decision.

`final_outcome` represents the terminal business result.

These are intentionally separate.

---

## 6.3 REFERRAL_OUTCOME

### Purpose

Represents the final resolution of a referral.

### Grain

One row per referral outcome.

### Primary Key

```text
referral_outcome_id
```

### Foreign Key

```text
referral_id
→ REFERRAL.referral_id
```

### Attributes

| Attribute             | Type           | Key | Description                         |
| --------------------- | -------------- | --- | ----------------------------------- |
| referral_outcome_id   | INTEGER / UUID | PK  | Unique outcome                      |
| referral_id           | INTEGER / UUID | FK  | Referral                            |
| outcome_date          | DATE           |     | Resolution date                     |
| outcome_type          | VARCHAR        |     | INTERNAL / EXTERNAL / NOT_COMPLETED |
| completion_status     | VARCHAR        |     | Completed / Not Completed           |
| completion_service_id | INTEGER / UUID | FK  | Delivered service if applicable     |

### Note

The terminal outcome may also be stored on `REFERRAL.final_outcome` for analytical convenience.

The detailed outcome entity provides event-level traceability.

---

## 6.4 PATIENT_DECISION

### Purpose

Represents the patient's behavioral decision during the referral journey.

### Grain

One row per patient decision event.

### Primary Key

```text
patient_decision_id
```

### Foreign Keys

```text
referral_id
→ REFERRAL.referral_id
```

### Attributes

| Attribute           | Type           | Key | Description            |
| ------------------- | -------------- | --- | ---------------------- |
| patient_decision_id | INTEGER / UUID | PK  | Unique decision        |
| referral_id         | INTEGER / UUID | FK  | Referral               |
| decision_date       | DATE           |     | Decision date          |
| decision_type       | VARCHAR        |     | WAIT / EXTERNAL / DROP |
| decision_reason     | VARCHAR        |     | Optional reason        |

### Business Rules

Patient decisions must be distinguishable from referral final outcomes.

Example:

```text
Patient Decision = WAIT
Final Outcome = INTERNAL
```

or:

```text
Patient Decision = EXTERNAL
Final Outcome = EXTERNAL
```

---

# 7. Demand and Waiting List

---

## 7.1 DEMAND_EVENT

### Purpose

Represents demand generated for a healthcare service or specialty.

### Grain

One row per demand event.

### Primary Key

```text
demand_event_id
```

### Foreign Keys

```text
patient_id
→ PATIENT.patient_id

referral_id
→ REFERRAL.referral_id

target_specialty_id
→ SPECIALTY.specialty_id

target_clinic_id
→ CLINIC.clinic_id
```

### Attributes

| Attribute           | Type           | Key | Description         |
| ------------------- | -------------- | --- | ------------------- |
| demand_event_id     | INTEGER / UUID | PK  | Unique demand event |
| patient_id          | INTEGER / UUID | FK  | Patient             |
| referral_id         | INTEGER / UUID | FK  | Referral            |
| target_specialty_id | INTEGER / UUID | FK  | Requested specialty |
| target_clinic_id    | INTEGER / UUID | FK  | Target clinic       |
| demand_date         | DATE           |     | Demand date         |
| urgency             | VARCHAR        |     | Urgency category    |
| demand_status       | VARCHAR        |     | Current status      |

### Business Rules

Demand represents the need for service.

It is distinct from:

```text
Appointments
Actual Service Delivery
```

---

## 7.2 WAITING_LIST_ENTRY

### Purpose

Represents a patient's position or status in a waiting queue.

### Grain

One row per waiting-list episode.

### Primary Key

```text
waiting_list_entry_id
```

### Foreign Keys

```text
demand_event_id
→ DEMAND_EVENT.demand_event_id
```

### Attributes

| Attribute             | Type           | Key | Description                               |
| --------------------- | -------------- | --- | ----------------------------------------- |
| waiting_list_entry_id | INTEGER / UUID | PK  | Unique queue entry                        |
| demand_event_id       | INTEGER / UUID | FK  | Demand event                              |
| clinic_id             | INTEGER / UUID | FK  | Clinic                                    |
| specialty_id          | INTEGER / UUID | FK  | Specialty                                 |
| entry_date            | DATE           |     | Queue entry date                          |
| exit_date             | DATE           |     | Queue exit date                           |
| queue_status          | VARCHAR        |     | WAITING / SCHEDULED / COMPLETED / DROPPED |
| priority              | INTEGER        |     | Queue priority                            |
| position              | INTEGER        |     | Queue position                            |

---

# 8. Appointment and Service Delivery

---

## 8.1 APPOINTMENT

### Purpose

Represents a scheduled appointment.

### Grain

One row per appointment.

### Primary Key

```text
appointment_id
```

### Foreign Keys

```text
patient_id
→ PATIENT.patient_id

doctor_id
→ DOCTOR.doctor_id

clinic_id
→ CLINIC.clinic_id

room_id
→ ROOM.room_id

service_type_id
→ SERVICE_TYPE.service_type_id

demand_event_id
→ DEMAND_EVENT.demand_event_id
```

### Attributes

| Attribute          | Type           | Key | Description                                 |
| ------------------ | -------------- | --- | ------------------------------------------- |
| appointment_id     | INTEGER / UUID | PK  | Unique appointment                          |
| patient_id         | INTEGER / UUID | FK  | Patient                                     |
| doctor_id          | INTEGER / UUID | FK  | Doctor                                      |
| clinic_id          | INTEGER / UUID | FK  | Clinic                                      |
| room_id            | INTEGER / UUID | FK  | Assigned room                               |
| service_type_id    | INTEGER / UUID | FK  | Service                                     |
| demand_event_id    | INTEGER / UUID | FK  | Source demand                               |
| appointment_date   | DATE           |     | Appointment date                            |
| start_time         | TIME           |     | Start time                                  |
| end_time           | TIME           |     | End time                                    |
| appointment_status | VARCHAR        |     | Scheduled / Completed / Cancelled / No-show |

### Business Rules

For standard rooms:

```text
The same room cannot have overlapping appointments.
```

Appointments must be compatible with:

* doctor schedule,
* room availability,
* clinic availability,
* service duration.

---

## 8.2 SERVICE_DELIVERY

### Purpose

Represents a healthcare service actually delivered.

### Grain

One row per delivered service.

### Primary Key

```text
service_delivery_id
```

### Foreign Keys

```text
appointment_id
→ APPOINTMENT.appointment_id

patient_id
→ PATIENT.patient_id

doctor_id
→ DOCTOR.doctor_id

clinic_id
→ CLINIC.clinic_id

service_type_id
→ SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute           | Type           | Key | Description           |
| ------------------- | -------------- | --- | --------------------- |
| service_delivery_id | INTEGER / UUID | PK  | Unique delivery       |
| appointment_id      | INTEGER / UUID | FK  | Source appointment    |
| patient_id          | INTEGER / UUID | FK  | Patient               |
| doctor_id           | INTEGER / UUID | FK  | Doctor                |
| clinic_id           | INTEGER / UUID | FK  | Clinic                |
| service_type_id     | INTEGER / UUID | FK  | Service               |
| delivery_date       | DATE           |     | Delivery date         |
| delivery_status     | VARCHAR        |     | Delivered / Cancelled |
| nfz_eligible_flag   | BOOLEAN        |     | NFZ eligibility       |
| quantity            | INTEGER        |     | Delivered units       |

### Business Rules

A completed appointment may generate a service delivery.

A cancelled appointment must not generate a completed service delivery.

Actual delivered services are used in:

* NFZ utilization,
* revenue,
* capacity analysis,
* doctor compensation.

---

# 9. Capacity and Operational Planning

---

## 9.1 OPERATIONAL CAPACITY — Logical Concept

Operational capacity is not treated as an independent transactional entity in v1.4.

It is a derived concept calculated from operational source data.

### Primary Inputs

```text
DOCTOR_SCHEDULE
+
ROOM
+
SERVICE_TYPE
+
CLINIC AVAILABILITY
```

Conceptually:

```text
Doctor Available Hours
        +
Room Available Capacity
        +
Service Duration
        ↓
Operational Capacity
```

### Example

If:

```text
Doctor Available Hours = 8
Service Duration = 30 minutes
```

Then:

```text
Doctor-based capacity = 16 services
```

If only:

```text
10 room slots
```

are available, then:

```text
Operational Capacity = MIN(16, 10)
                      = 10 services
```

This prevents the model from treating `CAPACITY_FACT.operational_capacity` as an unexplained source value.

---

## 9.2 CAPACITY_FACT

### Purpose

Represents an analytical snapshot of demand and capacity.

### Grain

One row per:

```text
Date
+
Clinic
+
Specialty
+
Service Type
```

### Primary Key

```text
capacity_fact_id
```

### Foreign Keys

```text
clinic_id
→ CLINIC.clinic_id

specialty_id
→ SPECIALTY.specialty_id

service_type_id
→ SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute                 | Type           | Key | Description                                 |
| ------------------------- | -------------- | --- | ------------------------------------------- |
| capacity_fact_id          | INTEGER / UUID | PK  | Unique snapshot                             |
| snapshot_date             | DATE           |     | Snapshot date                               |
| clinic_id                 | INTEGER / UUID | FK  | Clinic                                      |
| specialty_id              | INTEGER / UUID | FK  | Specialty                                   |
| service_type_id           | INTEGER / UUID | FK  | Service                                     |
| demand_volume             | INTEGER        |     | Demand                                      |
| operational_capacity      | INTEGER        |     | Capacity derived from operational resources |
| nfz_funded_capacity       | INTEGER        |     | Contract-funded capacity                    |
| effective_capacity        | INTEGER        |     | Capacity available after constraints        |
| actual_delivered_services | INTEGER        |     | Actual delivered services                   |
| waiting_list_volume       | INTEGER        |     | Waiting list volume                         |

### Business Rules

`operational_capacity` must be derived from operational resource inputs.

`nfz_funded_capacity` must be derived from active NFZ contract terms.

Conceptually:

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

Actual delivered services may be lower than effective capacity.

---

# 10. NFZ Contract

---

## 10.1 NFZ_CONTRACT

### Purpose

Represents a contractual service volume available under an NFZ agreement.

### Grain

One row per contract, clinic, specialty, service and contractual period.

### Primary Key

```text
nfz_contract_id
```

### Foreign Keys

```text
clinic_id
→ CLINIC.clinic_id

specialty_id
→ SPECIALTY.specialty_id

service_type_id
→ SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute            | Type           | Key | Description               |
| -------------------- | -------------- | --- | ------------------------- |
| nfz_contract_id      | INTEGER / UUID | PK  | Unique contract           |
| clinic_id            | INTEGER / UUID | FK  | Clinic                    |
| specialty_id         | INTEGER / UUID | FK  | Specialty                 |
| service_type_id      | INTEGER / UUID | FK  | Service                   |
| effective_from       | DATE           |     | Contract start            |
| effective_to         | DATE           |     | Contract end              |
| contract_period_type | VARCHAR        |     | QUARTER / YEAR            |
| contract_year        | INTEGER        |     | Contract year             |
| contract_quarter     | INTEGER        |     | Contract quarter          |
| contracted_volume    | INTEGER        |     | Contracted service volume |
| unit_price           | DECIMAL        |     | Contract price            |
| is_active            | BOOLEAN        |     | Active contract           |

### Business Rules

For quarterly contracts:

```text
contract_period_type = QUARTER
```

and:

```text
contract_year
contract_quarter
```

identify the settlement period.

Core metrics include:

```text
Delivered Volume
Remaining Contract Capacity
Contract Utilization
Overperformance
Underperformance
```

Conceptually:

```text
Remaining Capacity
=
Contracted Volume
-
Delivered Volume
```

```text
Contract Utilization
=
Delivered Volume
/
Contracted Volume
```

---

# 11. Resource Reallocation

---

## 11.1 RESOURCE_REALLOCATION

### Purpose

Represents an actual operational resource change.

### Grain

One row per resource reallocation event.

### Primary Key

```text
reallocation_id
```

### Attributes

| Attribute         | Type           | Key | Description              |
| ----------------- | -------------- | --- | ------------------------ |
| reallocation_id   | INTEGER / UUID | PK  | Unique reallocation      |
| source_clinic_id  | INTEGER / UUID | FK  | Source clinic            |
| target_clinic_id  | INTEGER / UUID | FK  | Target clinic            |
| resource_type     | VARCHAR        |     | DOCTOR / ROOM / CAPACITY |
| resource_quantity | DECIMAL        |     | Quantity moved           |
| effective_date    | DATE           |     | Effective date           |
| reason            | VARCHAR        |     | Business reason          |

### Business Rules

Resource reallocation represents an actual operational event.

It may affect:

* doctor availability,
* room availability,
* operational capacity,
* effective capacity.

---

# 12. Financial Entities

---

## 12.1 SERVICE_PRICING

### Purpose

Defines service pricing over time.

### Grain

One row per service, payer type and effective pricing period.

### Primary Key

```text
service_pricing_id
```

### Foreign Keys

```text
service_type_id
→ SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute          | Type           | Key | Description           |
| ------------------ | -------------- | --- | --------------------- |
| service_pricing_id | INTEGER / UUID | PK  | Unique pricing rule   |
| service_type_id    | INTEGER / UUID | FK  | Service               |
| payer_type         | VARCHAR        |     | NFZ / PRIVATE / OTHER |
| price              | DECIMAL        |     | Unit price            |
| effective_from     | DATE           |     | Start date            |
| effective_to       | DATE           |     | End date              |
| is_active          | BOOLEAN        |     | Active status         |

### Business Rules

The applicable price is selected using:

```text
SERVICE_TYPE
+
PAYER_TYPE
+
SERVICE_DATE
```

---

## 12.2 REVENUE_TRANSACTION

### Purpose

Represents revenue generated by delivered healthcare services.

### Grain

One row per revenue-generating service transaction.

### Primary Key

```text
revenue_transaction_id
```

### Foreign Keys

```text
service_delivery_id
→ SERVICE_DELIVERY.service_delivery_id

clinic_id
→ CLINIC.clinic_id

service_pricing_id
→ SERVICE_PRICING.service_pricing_id
```

### Attributes

| Attribute              | Type           | Key | Description        |
| ---------------------- | -------------- | --- | ------------------ |
| revenue_transaction_id | INTEGER / UUID | PK  | Unique transaction |
| service_delivery_id    | INTEGER / UUID | FK  | Delivered service  |
| clinic_id              | INTEGER / UUID | FK  | Clinic             |
| service_pricing_id     | INTEGER / UUID | FK  | Applied pricing    |
| transaction_date       | DATE           |     | Transaction date   |
| payer_type             | VARCHAR        |     | Payer              |
| quantity               | INTEGER        |     | Quantity           |
| unit_price             | DECIMAL        |     | Applied price      |
| total_amount           | DECIMAL        |     | Total revenue      |

### Business Rule

```text
Total Revenue
=
Quantity × Unit Price
```

---

## 12.3 DOCTOR_COMPENSATION_RULE

### Purpose

Defines how doctors are compensated.

### Grain

One row per doctor compensation rule and effective period.

### Primary Key

```text
compensation_rule_id
```

### Foreign Keys

```text
doctor_id
→ DOCTOR.doctor_id
```

### Attributes

| Attribute            | Type           | Key | Description                      |
| -------------------- | -------------- | --- | -------------------------------- |
| compensation_rule_id | INTEGER / UUID | PK  | Unique rule                      |
| doctor_id            | INTEGER / UUID | FK  | Doctor                           |
| compensation_model   | VARCHAR        |     | PERCENTAGE / HOURLY_RATE / MIXED |
| percentage_rate      | DECIMAL        |     | Revenue percentage               |
| hourly_rate          | DECIMAL        |     | Hourly compensation              |
| effective_from       | DATE           |     | Start                            |
| effective_to         | DATE           |     | End                              |
| is_active            | BOOLEAN        |     | Active status                    |

---

## 12.4 DOCTOR_COMPENSATION_TRANSACTION

### Purpose

Represents actual doctor compensation generated by services or hours worked.

### Grain

One row per compensation transaction.

### Primary Key

```text
compensation_transaction_id
```

### Foreign Keys

```text
doctor_id
→ DOCTOR.doctor_id

service_delivery_id
→ SERVICE_DELIVERY.service_delivery_id

compensation_rule_id
→ DOCTOR_COMPENSATION_RULE.compensation_rule_id
```

### Attributes

| Attribute                   | Type           | Key | Description                      |
| --------------------------- | -------------- | --- | -------------------------------- |
| compensation_transaction_id | INTEGER / UUID | PK  | Unique transaction               |
| doctor_id                   | INTEGER / UUID | FK  | Doctor                           |
| service_delivery_id         | INTEGER / UUID | FK  | Service                          |
| compensation_rule_id        | INTEGER / UUID | FK  | Applied rule                     |
| transaction_date            | DATE           |     | Transaction date                 |
| calculation_method          | VARCHAR        |     | PERCENTAGE / HOURLY_RATE / MIXED |
| base_amount                 | DECIMAL        |     | Calculation base                 |
| compensation_amount         | DECIMAL        |     | Final compensation               |

---

## 12.5 OPERATING_COST_TRANSACTION

### Purpose

Represents non-doctor operating costs incurred by a clinic.

### Grain

One row per clinic operating cost transaction.

### Primary Key

```text
operating_cost_transaction_id
```

### Foreign Keys

```text
clinic_id
→ CLINIC.clinic_id
```

### Attributes

| Attribute                     | Type           | Key | Description             |
| ----------------------------- | -------------- | --- | ----------------------- |
| operating_cost_transaction_id | INTEGER / UUID | PK  | Unique cost transaction |
| clinic_id                     | INTEGER / UUID | FK  | Clinic                  |
| cost_date                     | DATE           |     | Cost date               |
| cost_type                     | VARCHAR        |     | Cost category           |
| amount                        | DECIMAL        |     | Cost amount             |
| currency                      | VARCHAR        |     | Currency                |
| description                   | VARCHAR        |     | Cost description        |

### Suggested Cost Types

```text
STAFF
RENT
ROOM
EQUIPMENT
UTILITIES
ADMINISTRATION
MAINTENANCE
OTHER
```

### Business Rules

Operating costs are separate from doctor compensation.

This allows:

```text
Clinic Revenue
-
Doctor Compensation
-
Operating Costs
=
Net Clinic Result
```

---

# 13. Financial KPI Logic

The model supports the following logical calculations.

## 13.1 Total Service Revenue

```text
SUM(REVENUE_TRANSACTION.total_amount)
```

---

## 13.2 Doctor Compensation

```text
SUM(DOCTOR_COMPENSATION_TRANSACTION.compensation_amount)
```

---

## 13.3 Operating Costs

```text
SUM(OPERATING_COST_TRANSACTION.amount)
```

---

## 13.4 Clinic Revenue After Doctor Compensation

```text
Total Service Revenue
-
Doctor Compensation
```

---

## 13.5 Net Clinic Result

```text
Total Service Revenue
-
Doctor Compensation
-
Operating Costs
```

This metric is now fully supported by the Entity Dictionary.

---

# 14. Scenario Modeling

---

## 14.1 SCENARIO

### Purpose

Represents a hypothetical operational scenario.

### Grain

One row per scenario.

### Primary Key

```text
scenario_id
```

### Attributes

| Attribute     | Type           | Key | Description                   |
| ------------- | -------------- | --- | ----------------------------- |
| scenario_id   | INTEGER / UUID | PK  | Unique scenario               |
| scenario_name | VARCHAR        |     | Scenario name                 |
| scenario_type | VARCHAR        |     | Capacity / Pricing / Resource |
| created_date  | DATE           |     | Creation date                 |
| description   | VARCHAR        |     | Scenario description          |

---

## 14.2 SCENARIO_RESOURCE_CHANGE

### Purpose

Represents a hypothetical resource adjustment within a scenario.

### Grain

One row per scenario resource change.

### Primary Key

```text
scenario_resource_change_id
```

### Foreign Keys

```text
scenario_id
→ SCENARIO.scenario_id
```

### Attributes

| Attribute                   | Type           | Key | Description              |
| --------------------------- | -------------- | --- | ------------------------ |
| scenario_resource_change_id | INTEGER / UUID | PK  | Unique change            |
| scenario_id                 | INTEGER / UUID | FK  | Scenario                 |
| clinic_id                   | INTEGER / UUID | FK  | Target clinic            |
| resource_type               | VARCHAR        |     | DOCTOR / ROOM / CAPACITY |
| change_quantity             | DECIMAL        |     | Quantity changed         |
| effective_date              | DATE           |     | Scenario date            |
| assumption                  | VARCHAR        |     | Scenario assumption      |

### Business Rules

Scenario changes must not modify actual operational entities.

---

# 15. Relationship Summary

The primary logical relationships are:

```text
PATIENT
   │
   ├── PATIENT_VISIT
   │       │
   │       └── REFERRAL
   │              │
   │              ├── PATIENT_DECISION
   │              ├── REFERRAL_OUTCOME
   │              └── DEMAND_EVENT
   │                       │
   │                       └── WAITING_LIST_ENTRY
   │
   └── APPOINTMENT
            │
            ├── DOCTOR
            ├── ROOM
            ├── CLINIC
            └── SERVICE_TYPE
                    │
                    ▼
             SERVICE_DELIVERY
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
 REVENUE_TRANSACTION   DOCTOR_COMPENSATION_TRANSACTION
          │
          ▼
 OPERATING_COST_TRANSACTION
```

Capacity model:

```text
DOCTOR
   │
   ▼
DOCTOR_SCHEDULE
   │
   ├───────────────┐
   ▼               ▼
ROOM          SERVICE_TYPE
   │               │
   └───────┬───────┘
           ▼
  OPERATIONAL CAPACITY
           │
           ▼
    NFZ_CONTRACT
           │
           ▼
   EFFECTIVE CAPACITY
           │
           ▼
     CAPACITY_FACT
```

---

# 16. Capacity Model

The capacity model follows this conceptual hierarchy:

```text
1. Doctor Availability
        ↓
2. Room Availability
        ↓
3. Service Duration
        ↓
4. Operational Capacity
        ↓
5. NFZ Funded Capacity
        ↓
6. Effective Capacity
        ↓
7. Actual Delivered Services
```

### Definitions

**Doctor Availability**

Hours during which doctors are scheduled and available.

**Room Availability**

Physical room capacity available for appointments.

**Operational Capacity**

Maximum service volume that can be delivered given available doctors, rooms and service duration.

**NFZ Funded Capacity**

Maximum volume supported by the active NFZ contract.

**Effective Capacity**

Capacity available after operational and contractual constraints.

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

**Actual Delivered Services**

Services actually completed.

The expected relationship is:

```text
Actual Delivered Services
≤
Effective Capacity
```

although overperformance may occur under specific business rules.

---

# 17. Data Integrity Rules

The following integrity constraints should be enforced in the logical and later physical model.

## 17.1 Patient Integrity

```text
PATIENT.patient_id
must be unique.
```

---

## 17.2 Referral Integrity

Every referral must reference:

```text
PATIENT
DOCTOR
CLINIC
SPECIALTY
```

---

## 17.3 Appointment Integrity

Every appointment must reference:

```text
PATIENT
DOCTOR
CLINIC
ROOM
SERVICE_TYPE
```

---

## 17.4 Room Conflict Integrity

For standard rooms:

```text
No overlapping appointments
for the same room_id.
```

---

## 17.5 Doctor Schedule Integrity

Appointments should fall within the doctor's available schedule.

Conceptually:

```text
Appointment Date
=
Schedule Date
```

and:

```text
Appointment Time
⊆
Doctor Available Time
```

---

## 17.6 Service Delivery Integrity

A completed service delivery must reference a valid appointment.

Cancelled appointments should not generate completed service delivery records.

---

## 17.7 Revenue Integrity

Revenue transactions should reference an actual service delivery.

---

## 17.8 Compensation Integrity

Doctor compensation should reference:

* a doctor,
* an applicable compensation rule,
* and, where service-based, a delivered service.

---

## 17.9 Operating Cost Integrity

Every operating cost must belong to a valid clinic.

---

## 17.10 Capacity Integrity

`CAPACITY_FACT.operational_capacity` must be derived from operational source data.

It must not be treated as an independently generated random value without relationship to:

* doctor schedules,
* room availability,
* service duration.

---

# 18. Derived KPI Layer

The Entity Dictionary supports the following analytical KPIs.

## Operational

```text
Demand Volume
Operational Capacity
Effective Capacity
Actual Delivered Services
Capacity Utilization
Room Utilization
Doctor Utilization
Waiting List Volume
Average Waiting Time
```

## Referral

```text
Referral Volume
Internal Referral Rate
External Referral Rate
Not Completed Rate
Patient Drop Rate
Patient External Decision Rate
```

## NFZ

```text
Contracted Volume
Delivered Volume
Remaining Contract Capacity
Contract Utilization
Overperformance
Underperformance
```

## Financial

```text
Total Service Revenue
Doctor Compensation
Operating Costs
Clinic Revenue
Net Clinic Result
Revenue per Doctor
Revenue per Service
```

---

# 19. Model Readiness Assessment

After version 1.4, the Entity Dictionary is considered structurally ready for the next modeling stage.

### Completed

```text
Business Rules
        ↓
Entity Dictionary
        ↓
Core Entities
        ↓
Capacity Sources
        ↓
Room Constraints
        ↓
Doctor Availability
        ↓
Financial Cost Model
```

### Remaining Design Work

The following topics are intentionally deferred to later documents:

* detailed process sequencing,
* state transitions,
* ERD cardinality visualization,
* physical database design,
* SQL data types,
* database constraints,
* indexing,
* Python implementation,
* generator architecture.

---

# 20. Next Step

The next document should be:

```text
05_Process_Flows.md
```

The Process Flows document should define the end-to-end operational lifecycle:

```text
Patient Registration
        ↓
Patient Visit
        ↓
Referral
        ↓
Demand Creation
        ↓
Internal Availability Check
        ↓
Waiting List
        ↓
Patient Decision
        ↓
Appointment Scheduling
        ↓
Service Delivery
        ↓
Revenue
        ↓
Doctor Compensation
        ↓
Operating Costs
        ↓
Clinic Financial Result
```

It should also define:

```text
Capacity Calculation
        ↓
NFZ Contract Validation
        ↓
Effective Capacity
        ↓
Waiting List Impact
```

and:

```text
Resource Reallocation
        ↓
Capacity Change
        ↓
Operational Impact
```

Scenario modeling should remain separate from actual operational flows.

---

# 21. Version History

| Version | Description                                                                                                                                |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| v1.0    | Initial Entity Dictionary                                                                                                                  |
| v1.1    | Business Rules alignment                                                                                                                   |
| v1.2    | Improved entity definitions and relationships                                                                                              |
| v1.3    | Extended logical model and financial entities                                                                                              |
| v1.4    | Added ROOM, DOCTOR_SCHEDULE, OPERATING_COST_TRANSACTION; clarified operational capacity derivation; strengthened referral and NFZ modeling |
