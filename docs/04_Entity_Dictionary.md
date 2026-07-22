# 04 — Entity Dictionary

**Version:** 1.3
**Status:** Draft — Ready for Final Audit
**Document Type:** Logical Data Model
**Project:** Healthcare Operations BI Simulator

---

## 1. Purpose

This document defines the logical data entities required by the **Healthcare Operations BI Simulator**.

The Entity Dictionary translates the business concepts and rules defined in:

* `01_Project_Overview.md`
* `02_Business_Model.md`
* `03_Business_Rules.md`

into a structured logical data model.

It serves as the bridge between:

```text
Business Requirements
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
Data Generation
        ↓
SQL Data Model
        ↓
BI / Tableau
```

The Entity Dictionary defines:

* logical entities,
* entity purpose,
* grain,
* primary keys,
* foreign keys,
* attributes,
* relationships,
* business meaning,
* temporal behavior,
* data integrity rules,
* derived values,
* simulation-specific assumptions.

This document does **not** define the physical database implementation.

SQL data types, indexes, constraints, partitioning, database-specific syntax, and physical optimization decisions will be defined later.

---

# 2. Modeling Principles

The logical model follows these principles:

1. Every entity has a clearly defined business purpose.
2. Every entity has a declared grain.
3. Every transactional entity represents a business event or state.
4. Master/reference entities define reusable business dimensions.
5. Historical behavior is represented through effective dates or event records where required.
6. Derived values should not be duplicated unless required for performance or auditability.
7. Business rules must be traceable to one or more entities.
8. Scenario assumptions must be separated from observed operational events.
9. Behavioral events must be separated from final business outcomes.
10. Operational transactions must be separated from analytical snapshots.
11. Foreign keys must reference valid parent entities.
12. Temporal relationships must use effective dates where business definitions can change over time.

---

# 3. Entity Classification

The model is divided into the following categories.

## 3.1 Organization and Reference Entities

* `CLINIC`
* `SPECIALTY`
* `CLINIC_SPECIALTY`
* `SERVICE_TYPE`
* `PAYER_TYPE`
* `DOCTOR`
* `DOCTOR_SPECIALTY`
* `DOCTOR_COMPENSATION_RULE`
* `BEHAVIOR_RULE`

## 3.2 Patient and Demand Entities

* `PATIENT`
* `DEMAND_EVENT`
* `REFERRAL`
* `PATIENT_DECISION`

## 3.3 Operational Entities

* `WAITING_LIST_ENTRY`
* `APPOINTMENT`
* `PATIENT_VISIT`
* `SERVICE_DELIVERY`

## 3.4 Capacity and Resource Entities

* `CAPACITY_FACT`
* `RESOURCE_REALLOCATION`

## 3.5 Financial Entities

* `SERVICE_PRICING`
* `REVENUE_TRANSACTION`
* `DOCTOR_COMPENSATION_TRANSACTION`
* `NFZ_CONTRACT`
* `NFZ_CONTRACT_AMENDMENT`

## 3.6 Scenario and Simulation Entities

* `SCENARIO`
* `SCENARIO_RESOURCE_CHANGE`

---

# 4. Entity Dependency Overview

The core operational flow is:

```text
PATIENT
   ↓
PATIENT_VISIT
   ↓
REFERRAL
   ↓
DEMAND_EVENT
   ↓
PATIENT_DECISION
   ↓
WAITING_LIST_ENTRY
   ↓
APPOINTMENT
   ↓
PATIENT_VISIT
   ↓
SERVICE_DELIVERY
   ↓
REVENUE_TRANSACTION
   ↓
DOCTOR_COMPENSATION_TRANSACTION
```

However, not every demand event follows the referral pathway.

The model supports:

```text
POZ Referral
Pediatric Referral
Internal Referral
External Referral
Follow-up Demand
Diagnostic Demand
Other Operational Demand
```

Therefore:

```text
DEMAND_EVENT
   ├── may originate from PATIENT_VISIT
   ├── may originate from REFERRAL
   ├── may represent FOLLOW_UP demand
   ├── may represent DIAGNOSTIC demand
   └── may be generated directly by the simulation
```

For referral-driven demand:

```text
PATIENT_VISIT
      ↓
REFERRAL
      ↓
DEMAND_EVENT
```

For non-referral demand:

```text
PATIENT_VISIT / Simulation Logic
      ↓
DEMAND_EVENT
```

The model therefore does **not** require every `DEMAND_EVENT` to have a `REFERRAL`.

---

# 5. Master and Reference Entities

---

## 5.1 CLINIC

### Purpose

Represents a healthcare clinic or organizational unit participating in the simulation.

The baseline simulation contains a fixed initial operating footprint of approximately **11 specialist clinics**.

The model is designed to support future expansion or changes in the clinic network.

### Grain

One record per clinic.

### Primary Key

```text
clinic_id
```

### Attributes

| Attribute      | Description                                 | Required |
| -------------- | ------------------------------------------- | -------- |
| `clinic_id`    | Unique clinic identifier                    | Yes      |
| `clinic_code`  | Business-readable clinic code               | Yes      |
| `clinic_name`  | Clinic name                                 | Yes      |
| `clinic_role`  | Organizational role, e.g. specialist clinic | Yes      |
| `location`     | Geographic location                         | Yes      |
| `opening_date` | Date clinic became operational              | Yes      |
| `closing_date` | Date clinic ceased operations               | No       |
| `is_active`    | Current operational status                  | Yes      |

### Business Rules

* The baseline scenario contains approximately 11 specialist clinics.
* A clinic may support multiple specialties.
* A specialty may be available in multiple clinics.
* Clinic availability may change over time.
* `clinic_role` distinguishes organizational roles if the simulation expands beyond specialist clinics.
* A clinic must not receive operational transactions before `opening_date`.
* A closed clinic must not receive new operational transactions after `closing_date`.

---

## 5.2 SPECIALTY

### Purpose

Represents the complete catalog of medical specialties used by the simulation.

The baseline specialty catalog contains approximately **20–25 specialties**.

Not every specialty must be available internally.

### Grain

One record per medical specialty.

### Primary Key

```text
specialty_id
```

### Attributes

| Attribute        | Description                                | Required |
| ---------------- | ------------------------------------------ | -------- |
| `specialty_id`   | Unique specialty identifier                | Yes      |
| `specialty_code` | Standardized specialty code                | Yes      |
| `specialty_name` | Specialty name                             | Yes      |
| `is_active`      | Whether specialty is active in the catalog | Yes      |

### Business Rules

* The specialty catalog is broader than the internally available specialty footprint.
* Internal availability is determined through `CLINIC_SPECIALTY`.
* A specialty may exist in the catalog without being available internally.
* A specialty may be available in multiple clinics.

---

## 5.3 CLINIC_SPECIALTY

### Purpose

Represents the temporal relationship between a clinic and a specialty.

This entity determines whether a specialty is available internally at a given clinic and time.

### Grain

One record per clinic-specialty relationship and effective period.

### Primary Key

```text
clinic_specialty_id
```

### Foreign Keys

```text
clinic_id → CLINIC.clinic_id
specialty_id → SPECIALTY.specialty_id
```

### Attributes

| Attribute             | Description                    | Required |
| --------------------- | ------------------------------ | -------- |
| `clinic_specialty_id` | Unique relationship identifier | Yes      |
| `clinic_id`           | Clinic                         | Yes      |
| `specialty_id`        | Specialty                      | Yes      |
| `effective_from`      | Start date of availability     | Yes      |
| `effective_to`        | End date of availability       | No       |
| `is_active`           | Current active status          | Yes      |

### Business Rules

* Only active relationships represent current internal availability.
* A specialty can be available in multiple clinics.
* Historical availability must be preserved.
* Overlapping active periods for the same clinic-specialty pair should be avoided.
* `effective_to` must be greater than or equal to `effective_from`.
* A referral destination marked as `INTERNAL` must point to a specialty with valid internal availability at the relevant date.

---

## 5.4 SERVICE_TYPE

### Purpose

Defines the types of medical services delivered by the organization.

### Grain

One record per service type.

### Primary Key

```text
service_type_id
```

### Attributes

| Attribute                   | Description                  | Required |
| --------------------------- | ---------------------------- | -------- |
| `service_type_id`           | Unique service identifier    | Yes      |
| `service_code`              | Service code                 | Yes      |
| `service_name`              | Service name                 | Yes      |
| `visit_type`                | `FIRST_VISIT` or `FOLLOW_UP` | Yes      |
| `standard_duration_minutes` | Standard expected duration   | Yes      |
| `is_active`                 | Current active status        | Yes      |

### Business Rules

* `visit_type` distinguishes first visits from follow-up visits.
* Service duration may be used in capacity calculations.
* Service types may have different pricing.
* Service types may have different compensation implications.
* Service types may be subject to different NFZ contractual limits.

---

## 5.5 PAYER_TYPE

### Purpose

Defines the source of payment for a delivered service.

### Grain

One record per payer category.

### Primary Key

```text
payer_type_id
```

### Example Values

```text
NFZ
PRIVATE
INSURANCE
OTHER
```

### Attributes

| Attribute       | Description             | Required |
| --------------- | ----------------------- | -------- |
| `payer_type_id` | Unique payer identifier | Yes      |
| `payer_code`    | Payer code              | Yes      |
| `payer_name`    | Payer name              | Yes      |
| `is_active`     | Active status           | Yes      |

---

## 5.6 DOCTOR

### Purpose

Represents a physician participating in healthcare operations.

### Grain

One record per doctor.

### Primary Key

```text
doctor_id
```

### Foreign Keys

```text
primary_specialty_id → SPECIALTY.specialty_id
```

### Attributes

| Attribute               | Description                   | Required |
| ----------------------- | ----------------------------- | -------- |
| `doctor_id`             | Unique doctor identifier      | Yes      |
| `doctor_code`           | Business-readable doctor code | Yes      |
| `doctor_name`           | Doctor name                   | Yes      |
| `primary_specialty_id`  | Primary specialty             | Yes      |
| `employment_type`       | Employment relationship       | Yes      |
| `employment_start_date` | Start date                    | Yes      |
| `employment_end_date`   | End date                      | No       |
| `is_active`             | Current active status         | Yes      |

### Business Rules

* `primary_specialty_id` represents the doctor's primary specialty.
* Additional specialties are represented through `DOCTOR_SPECIALTY`.
* A doctor may work across multiple clinics if permitted by the simulation.
* A doctor must not deliver a service outside their active employment period.
* A doctor may only perform services for specialties for which they have a valid specialty relationship.

---

## 5.7 DOCTOR_SPECIALTY

### Purpose

Represents all specialty qualifications or assignments associated with a doctor.

### Grain

One record per doctor-specialty relationship and effective period.

### Primary Key

```text
doctor_specialty_id
```

### Foreign Keys

```text
doctor_id → DOCTOR.doctor_id
specialty_id → SPECIALTY.specialty_id
```

### Attributes

| Attribute             | Description                    | Required |
| --------------------- | ------------------------------ | -------- |
| `doctor_specialty_id` | Unique relationship identifier | Yes      |
| `doctor_id`           | Doctor                         | Yes      |
| `specialty_id`        | Specialty                      | Yes      |
| `effective_from`      | Start date                     | Yes      |
| `effective_to`        | End date                       | No       |
| `is_primary`          | Indicates primary specialty    | Yes      |
| `is_active`           | Active relationship            | Yes      |

### Business Rules

* `DOCTOR.primary_specialty_id` represents the primary specialty for simplified reporting.
* `DOCTOR_SPECIALTY` represents the complete specialty relationship.
* Exactly one active specialty relationship should be marked as primary.
* `DOCTOR.primary_specialty_id` must match the active `DOCTOR_SPECIALTY` relationship where `is_primary = TRUE`.
* This model allows future multi-specialty doctors without changing the core `DOCTOR` entity.

---

## 5.8 DOCTOR_COMPENSATION_RULE

### Purpose

Defines how a doctor is compensated.

### Grain

One record per doctor compensation rule and effective period.

### Primary Key

```text
compensation_rule_id
```

### Foreign Keys

```text
doctor_id → DOCTOR.doctor_id
```

### Attributes

| Attribute              | Description                          | Required |
| ---------------------- | ------------------------------------ | -------- |
| `compensation_rule_id` | Unique rule identifier               | Yes      |
| `doctor_id`            | Doctor                               | Yes      |
| `compensation_model`   | `PERCENTAGE`, `HOURLY_RATE`, `MIXED` | Yes      |
| `percentage_rate`      | Percentage component                 | No       |
| `hourly_rate`          | Hourly component                     | No       |
| `effective_from`       | Start date                           | Yes      |
| `effective_to`         | End date                             | No       |
| `is_active`            | Active status                        | Yes      |

### Business Rules

#### PERCENTAGE

```text
Doctor Compensation
=
Gross Service Revenue
×
Percentage Rate
```

#### HOURLY_RATE

```text
Doctor Compensation
=
Doctor Hours
×
Hourly Rate
```

#### MIXED

```text
Percentage Component
=
Gross Service Revenue
×
Percentage Rate

Hourly Component
=
Doctor Hours
×
Hourly Rate

Total Compensation
=
Percentage Component
+
Hourly Component
```

For `PERCENTAGE`:

* `percentage_rate` is required.
* `hourly_rate` should be null.

For `HOURLY_RATE`:

* `hourly_rate` is required.
* `percentage_rate` should be null.

For `MIXED`:

* both components are required.

---

## 5.9 BEHAVIOR_RULE

### Purpose

Defines configurable patient behavior probabilities used by the simulation.

### Grain

One record per behavioral rule and effective period.

### Primary Key

```text
behavior_rule_id
```

### Attributes

| Attribute              | Description                          | Required |
| ---------------------- | ------------------------------------ | -------- |
| `behavior_rule_id`     | Unique rule identifier               | Yes      |
| `rule_name`            | Rule name                            | Yes      |
| `rule_type`            | Behavior category                    | Yes      |
| `base_probability`     | Base probability                     | Yes      |
| `time_dependency_type` | How waiting time affects probability | No       |
| `effective_from`       | Start date                           | Yes      |
| `effective_to`         | End date                             | No       |
| `is_active`            | Active status                        | Yes      |

### Example Behavior Types

```text
WAIT
EXTERNAL
DROP
```

### Business Rules

Patient behavior may change as waiting time increases.

For example:

```text
Waiting Time ↑
      ↓
Probability of EXTERNAL ↑
Probability of DROP ↑
Probability of continuing to WAIT ↓
```

The exact probability curve is a simulation parameter and must be configurable.

---

# 6. Patient and Demand Entities

---

## 6.1 PATIENT

### Purpose

Represents a patient participating in the simulation.

### Grain

One record per patient.

### Primary Key

```text
patient_id
```

### Attributes

| Attribute           | Description                     | Required |
| ------------------- | ------------------------------- | -------- |
| `patient_id`        | Unique patient identifier       | Yes      |
| `patient_code`      | Anonymized business identifier  | Yes      |
| `date_of_birth`     | Date of birth                   | Yes      |
| `gender`            | Gender category                 | Yes      |
| `registration_date` | Date patient entered the system | Yes      |
| `home_location`     | Geographic location             | No       |
| `is_active`         | Active patient status           | Yes      |

### Business Rules

* Patients must be anonymized.
* Patient identity must not contain real personally identifiable information.
* One patient may generate multiple demand events.
* One patient may have multiple visits.
* One patient may have multiple referrals.
* One patient may have multiple behavioral decisions.

---

## 6.2 DEMAND_EVENT

### Purpose

Represents a unit of demand entering the healthcare operations system.

### Grain

One record per demand event.

### Primary Key

```text
demand_event_id
```

### Foreign Keys

```text
patient_id → PATIENT.patient_id
referral_id → REFERRAL.referral_id
specialty_id → SPECIALTY.specialty_id
service_type_id → SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute            | Description                            | Required |
| -------------------- | -------------------------------------- | -------- |
| `demand_event_id`    | Unique demand event identifier         | Yes      |
| `patient_id`         | Patient generating demand              | Yes      |
| `referral_id`        | Associated referral, if applicable     | No       |
| `specialty_id`       | Requested specialty                    | Yes      |
| `service_type_id`    | Requested service                      | Yes      |
| `demand_source`      | Source of demand                       | Yes      |
| `demand_date`        | Date demand entered                    | Yes      |
| `priority`           | Demand priority                        | Yes      |
| `is_internal_target` | Whether target is internally available | Yes      |
| `status`             | Current demand status                  | Yes      |

### Demand Source Examples

```text
POZ
PEDIATRIC
REFERRAL
FOLLOW_UP
DIAGNOSTIC
OTHER
```

### Business Rules

A demand event may be generated:

1. directly by the simulation,
2. from a patient visit,
3. from a referral,
4. as follow-up demand,
5. as diagnostic demand.

Not every demand event requires a referral.

For referral-driven demand:

```text
PATIENT_VISIT
      ↓
REFERRAL
      ↓
DEMAND_EVENT
```

For non-referral demand:

```text
PATIENT_VISIT / Simulation Logic
      ↓
DEMAND_EVENT
```

A demand event is considered internally targetable if the requested specialty has an active `CLINIC_SPECIALTY` relationship at the relevant date.

---

## 6.3 REFERRAL

### Purpose

Represents a referral pathway initiated for a patient.

### Grain

One record per referral.

### Primary Key

```text
referral_id
```

### Foreign Keys

```text
patient_id → PATIENT.patient_id
referring_doctor_id → DOCTOR.doctor_id
target_specialty_id → SPECIALTY.specialty_id
source_visit_id → PATIENT_VISIT.visit_id
```

### Attributes

| Attribute              | Description                     | Required |
| ---------------------- | ------------------------------- | -------- |
| `referral_id`          | Unique referral identifier      | Yes      |
| `patient_id`           | Patient                         | Yes      |
| `referring_doctor_id`  | Referring doctor                | No       |
| `target_specialty_id`  | Requested specialty             | Yes      |
| `source_visit_id`      | Visit that generated referral   | No       |
| `referral_date`        | Referral creation date          | Yes      |
| `referral_destination` | `INTERNAL` or `EXTERNAL` target | Yes      |
| `referral_status`      | Current lifecycle status        | Yes      |
| `final_outcome`        | Final business outcome          | No       |
| `outcome_date`         | Date final outcome occurred     | No       |

### Referral Status Examples

```text
CREATED
WAITING
SCHEDULED
COMPLETED
CANCELLED
```

### Final Outcome Examples

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

### Business Rules

`final_outcome` represents the final business outcome of the referral lifecycle.

It is **not** a behavioral event.

While the referral is still active:

```text
CREATED
WAITING
SCHEDULED
```

the following should normally apply:

```text
final_outcome = NULL
```

When the referral reaches a terminal state:

```text
COMPLETED
CANCELLED
```

the final outcome should be populated.

Example:

```text
Referral Created
      ↓
WAITING
      ↓
Patient Waits
      ↓
Patient Decision = WAIT
      ↓
Appointment Scheduled
      ↓
Referral Completed
      ↓
final_outcome = INTERNAL
```

Or:

```text
Referral Created
      ↓
WAITING
      ↓
Patient Decision = EXTERNAL
      ↓
Referral Cancelled
      ↓
final_outcome = EXTERNAL
```

---

## 6.4 PATIENT_DECISION

### Purpose

Represents a behavioral decision made by a patient during the referral or waiting process.

### Grain

One record per patient decision event.

### Primary Key

```text
patient_decision_id
```

### Foreign Keys

```text
patient_id → PATIENT.patient_id
referral_id → REFERRAL.referral_id
behavior_rule_id → BEHAVIOR_RULE.behavior_rule_id
```

### Attributes

| Attribute                          | Description                      | Required |
| ---------------------------------- | -------------------------------- | -------- |
| `patient_decision_id`              | Unique decision event identifier | Yes      |
| `patient_id`                       | Patient                          | Yes      |
| `referral_id`                      | Referral being evaluated         | Yes      |
| `decision_date`                    | Date of decision                 | Yes      |
| `waiting_time_at_decision_days`    | Waiting duration at decision     | Yes      |
| `decision_type`                    | `WAIT`, `EXTERNAL`, `DROP`       | Yes      |
| `behavior_rule_id`                 | Rule used                        | No       |
| `decision_probability_at_decision` | Probability at decision time     | No       |

### Business Rules

`PATIENT_DECISION` represents a **behavioral event**.

It is not the same as `REFERRAL.final_outcome`.

A patient may generate multiple decisions:

```text
Day 30 → WAIT
Day 60 → WAIT
Day 90 → EXTERNAL
```

These are represented as:

```text
PATIENT_DECISION #1
PATIENT_DECISION #2
PATIENT_DECISION #3
```

The final referral outcome is stored separately:

```text
REFERRAL.final_outcome = EXTERNAL
```

Therefore:

```text
PATIENT_DECISION
=
Behavioral Event

REFERRAL.final_outcome
=
Final Business Outcome
```

This distinction is mandatory for historical behavioral analysis.

---

# 7. Operational Entities

---

## 7.1 WAITING_LIST_ENTRY

### Purpose

Represents a patient's episode of waiting for a service.

### Grain

One record per patient waiting-list episode.

### Primary Key

```text
waiting_list_entry_id
```

### Foreign Keys

```text
patient_id → PATIENT.patient_id
demand_event_id → DEMAND_EVENT.demand_event_id
specialty_id → SPECIALTY.specialty_id
clinic_id → CLINIC.clinic_id
```

### Attributes

| Attribute               | Description               | Required |
| ----------------------- | ------------------------- | -------- |
| `waiting_list_entry_id` | Unique queue entry        | Yes      |
| `patient_id`            | Patient                   | Yes      |
| `demand_event_id`       | Demand event              | Yes      |
| `specialty_id`          | Requested specialty       | Yes      |
| `clinic_id`             | Target clinic             | No       |
| `queue_entry_datetime`  | Entry timestamp           | Yes      |
| `queue_exit_datetime`   | Exit timestamp            | No       |
| `queue_status`          | Current/final queue state | Yes      |
| `waiting_days`          | Total waiting duration    | No       |

### Queue Status Examples

```text
WAITING
SCHEDULED
SERVED
EXTERNAL
DROPPED
CANCELLED
```

### Business Rules

A waiting-list entry represents one waiting episode.

A patient may have multiple waiting episodes over time.

`waiting_days` may be derived as:

```text
queue_exit_datetime
-
queue_entry_datetime
```

For active queue entries:

```text
waiting_days
=
current_date
-
queue_entry_datetime
```

Historical queue size can be derived from active waiting episodes.

A future analytical layer may introduce a daily queue snapshot if required.

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
patient_id → PATIENT.patient_id
doctor_id → DOCTOR.doctor_id
clinic_id → CLINIC.clinic_id
specialty_id → SPECIALTY.specialty_id
service_type_id → SERVICE_TYPE.service_type_id
waiting_list_entry_id → WAITING_LIST_ENTRY.waiting_list_entry_id
```

### Attributes

| Attribute               | Description                   | Required |
| ----------------------- | ----------------------------- | -------- |
| `appointment_id`        | Unique appointment identifier | Yes      |
| `patient_id`            | Patient                       | Yes      |
| `doctor_id`             | Doctor                        | Yes      |
| `clinic_id`             | Clinic                        | Yes      |
| `specialty_id`          | Specialty                     | Yes      |
| `service_type_id`       | Service type                  | Yes      |
| `waiting_list_entry_id` | Queue episode                 | No       |
| `scheduled_datetime`    | Scheduled time                | Yes      |
| `appointment_status`    | Appointment state             | Yes      |
| `cancellation_reason`   | Cancellation reason           | No       |

### Appointment Status Examples

```text
SCHEDULED
COMPLETED
CANCELLED
NO_SHOW
```

---

## 7.3 PATIENT_VISIT

### Purpose

Represents a patient encounter or visit.

### Grain

One record per patient visit.

### Primary Key

```text
visit_id
```

### Foreign Keys

```text
patient_id → PATIENT.patient_id
doctor_id → DOCTOR.doctor_id
clinic_id → CLINIC.clinic_id
appointment_id → APPOINTMENT.appointment_id
```

### Attributes

| Attribute        | Description             | Required |
| ---------------- | ----------------------- | -------- |
| `visit_id`       | Unique visit identifier | Yes      |
| `patient_id`     | Patient                 | Yes      |
| `doctor_id`      | Doctor                  | Yes      |
| `clinic_id`      | Clinic                  | Yes      |
| `appointment_id` | Appointment             | No       |
| `visit_datetime` | Visit timestamp         | Yes      |
| `visit_status`   | Visit state             | Yes      |
| `visit_type`     | First/follow-up         | Yes      |

### Business Rules

A visit may:

* generate a referral,
* generate a new demand event,
* generate follow-up demand,
* generate diagnostic demand,
* result in one or more delivered services.

A referral generated by a visit should reference:

```text
REFERRAL.source_visit_id = PATIENT_VISIT.visit_id
```

---

## 7.4 SERVICE_DELIVERY

### Purpose

Represents the actual delivery of a healthcare service.

### Grain

One record per delivered service.

### Primary Key

```text
service_delivery_id
```

### Foreign Keys

```text
patient_id → PATIENT.patient_id
doctor_id → DOCTOR.doctor_id
clinic_id → CLINIC.clinic_id
specialty_id → SPECIALTY.specialty_id
service_type_id → SERVICE_TYPE.service_type_id
appointment_id → APPOINTMENT.appointment_id
visit_id → PATIENT_VISIT.visit_id
payer_type_id → PAYER_TYPE.payer_type_id
```

### Attributes

| Attribute             | Description          | Required |
| --------------------- | -------------------- | -------- |
| `service_delivery_id` | Unique service event | Yes      |
| `patient_id`          | Patient              | Yes      |
| `doctor_id`           | Doctor               | Yes      |
| `clinic_id`           | Clinic               | Yes      |
| `specialty_id`        | Specialty            | Yes      |
| `service_type_id`     | Service type         | Yes      |
| `appointment_id`      | Appointment          | No       |
| `visit_id`            | Visit                | Yes      |
| `payer_type_id`       | Payer                | Yes      |
| `service_date`        | Delivery date        | Yes      |
| `duration_minutes`    | Actual duration      | Yes      |
| `delivery_status`     | Delivery status      | Yes      |

### Business Rules

A service is considered delivered only if the operational event is completed.

`SERVICE_DELIVERY` is the source transaction for:

```text
Revenue
Doctor Compensation
Capacity Utilization
Service Volume
```

---

# 8. Capacity and Resource Entities

---

## 8.1 CAPACITY_FACT

### Purpose

Represents the operational capacity and utilization state for a specific reporting date.

### Grain

One record per:

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
date_key
clinic_id → CLINIC.clinic_id
specialty_id → SPECIALTY.specialty_id
service_type_id → SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute                   | Description                          | Required |
| --------------------------- | ------------------------------------ | -------- |
| `capacity_fact_id`          | Unique fact identifier               | Yes      |
| `date_key`                  | Reporting/snapshot date              | Yes      |
| `clinic_id`                 | Clinic                               | Yes      |
| `specialty_id`              | Specialty                            | Yes      |
| `service_type_id`           | Service type                         | Yes      |
| `operational_capacity`      | Maximum operational capacity         | Yes      |
| `nfz_funded_capacity`       | NFZ-funded capacity                  | Yes      |
| `effective_capacity`        | Capacity available after constraints | Yes      |
| `actual_delivered_services` | Services actually delivered          | Yes      |
| `utilization_rate`          | Utilization ratio                    | No       |

### Business Rules

`CAPACITY_FACT` represents a period-specific capacity snapshot.

It combines:

```text
Capacity State
+
Actual Utilization
```

for a specific reporting date.

Example:

```text
Effective Capacity = 20
Actual Delivered Services = 18

Utilization Rate
=
18 / 20
=
90%
```

The transaction-level source of actual service delivery is:

```text
SERVICE_DELIVERY
```

`CAPACITY_FACT` is an analytical snapshot and should not replace transactional service records.

---

## 8.2 RESOURCE_REALLOCATION

### Purpose

Represents an actual operational reallocation of resources between clinics.

### Grain

One record per resource reallocation event.

### Primary Key

```text
reallocation_id
```

### Foreign Keys

```text
source_clinic_id → CLINIC.clinic_id
target_clinic_id → CLINIC.clinic_id
```

### Attributes

| Attribute           | Description             | Required |
| ------------------- | ----------------------- | -------- |
| `reallocation_id`   | Unique event identifier | Yes      |
| `source_clinic_id`  | Source clinic           | Yes      |
| `target_clinic_id`  | Target clinic           | Yes      |
| `resource_type`     | Resource category       | Yes      |
| `source_quantity`   | Quantity removed        | Yes      |
| `target_quantity`   | Quantity added          | Yes      |
| `reallocation_date` | Date                    | Yes      |
| `reason`            | Business reason         | No       |

### Resource Types

```text
DOCTORS
DOCTOR_HOURS
ROOMS
OPERATIONAL_CAPACITY
```

### Business Rules

This entity represents an actual operational change.

It is distinct from:

```text
SCENARIO_RESOURCE_CHANGE
```

which represents a hypothetical what-if assumption.

---

# 9. Financial Entities

---

## 9.1 SERVICE_PRICING

### Purpose

Defines the applicable price of a service for a given payer type.

### Grain

One record per:

```text
Service Type
+
Payer Type
+
Effective Period
```

### Primary Key

```text
service_pricing_id
```

### Foreign Keys

```text
service_type_id → SERVICE_TYPE.service_type_id
payer_type_id → PAYER_TYPE.payer_type_id
```

### Attributes

| Attribute            | Description           | Required |
| -------------------- | --------------------- | -------- |
| `service_pricing_id` | Unique pricing record | Yes      |
| `service_type_id`    | Service type          | Yes      |
| `payer_type_id`      | Payer type            | Yes      |
| `price_amount`       | Price                 | Yes      |
| `currency`           | Currency              | Yes      |
| `effective_from`     | Start date            | Yes      |
| `effective_to`       | End date              | No       |
| `is_active`          | Active status         | Yes      |

### Business Rules

Applicable pricing is selected using:

```text
SERVICE_TYPE
+
PAYER_TYPE
+
SERVICE_DATE
```

The applicable record must satisfy:

```text
service_type_id
=
SERVICE_DELIVERY.service_type_id

AND

payer_type_id
=
SERVICE_DELIVERY.payer_type_id

AND

SERVICE_DATE >= effective_from

AND

SERVICE_DATE <= effective_to
```

If `effective_to` is null, the pricing record remains valid until replaced.

At most one active pricing record should apply to the same:

```text
SERVICE_TYPE
+
PAYER_TYPE
+
SERVICE_DATE
```

This prevents ambiguous revenue calculations.

---

## 9.2 REVENUE_TRANSACTION

### Purpose

Represents revenue generated by a delivered healthcare service.

### Grain

One record per revenue-generating service transaction.

### Primary Key

```text
revenue_transaction_id
```

### Foreign Keys

```text
service_delivery_id → SERVICE_DELIVERY.service_delivery_id
service_pricing_id → SERVICE_PRICING.service_pricing_id
payer_type_id → PAYER_TYPE.payer_type_id
```

### Attributes

| Attribute                | Description               | Required |
| ------------------------ | ------------------------- | -------- |
| `revenue_transaction_id` | Unique revenue identifier | Yes      |
| `service_delivery_id`    | Delivered service         | Yes      |
| `service_pricing_id`     | Applied pricing           | Yes      |
| `payer_type_id`          | Payer                     | Yes      |
| `transaction_date`       | Revenue date              | Yes      |
| `gross_service_revenue`  | Gross revenue             | Yes      |
| `currency`               | Currency                  | Yes      |
| `revenue_status`         | Revenue state             | Yes      |

### Business Rules

Gross revenue is derived from:

```text
SERVICE_DELIVERY
+
Applicable SERVICE_PRICING
```

The transaction must reference the exact pricing record used to calculate revenue.

This ensures historical pricing remains auditable.

---

## 9.3 DOCTOR_COMPENSATION_TRANSACTION

### Purpose

Represents compensation owed or paid to a doctor for delivered services.

### Grain

One record per doctor compensation transaction.

### Primary Key

```text
compensation_transaction_id
```

### Foreign Keys

```text
doctor_id → DOCTOR.doctor_id
service_delivery_id → SERVICE_DELIVERY.service_delivery_id
compensation_rule_id → DOCTOR_COMPENSATION_RULE.compensation_rule_id
```

### Attributes

| Attribute                        | Description                          | Required |
| -------------------------------- | ------------------------------------ | -------- |
| `compensation_transaction_id`    | Unique identifier                    | Yes      |
| `doctor_id`                      | Doctor                               | Yes      |
| `service_delivery_id`            | Delivered service                    | Yes      |
| `compensation_rule_id`           | Applied compensation rule            | Yes      |
| `calculation_method`             | `PERCENTAGE`, `HOURLY_RATE`, `MIXED` | Yes      |
| `percentage_compensation_amount` | Percentage component                 | No       |
| `hourly_compensation_amount`     | Hourly component                     | No       |
| `compensation_amount`            | Total compensation                   | Yes      |
| `transaction_date`               | Compensation date                    | Yes      |

### Business Rules

The applied compensation rule must be valid for the service delivery date.

For `PERCENTAGE`:

```text
percentage_compensation_amount
=
gross_service_revenue
×
percentage_rate
```

For `HOURLY_RATE`:

```text
hourly_compensation_amount
=
doctor_hours
×
hourly_rate
```

For `MIXED`:

```text
Total Compensation
=
Percentage Component
+
Hourly Component
```

The transaction stores the applied calculation method to preserve auditability even if the underlying compensation rule changes later.

---

## 9.4 NFZ_CONTRACT

### Purpose

Represents an NFZ contract governing funded service capacity.

### Grain

One record per clinic-specialty-service contract and effective period.

### Primary Key

```text
nfz_contract_id
```

### Foreign Keys

```text
clinic_id → CLINIC.clinic_id
specialty_id → SPECIALTY.specialty_id
service_type_id → SERVICE_TYPE.service_type_id
```

### Attributes

| Attribute           | Description                | Required |
| ------------------- | -------------------------- | -------- |
| `nfz_contract_id`   | Unique contract identifier | Yes      |
| `clinic_id`         | Clinic                     | Yes      |
| `specialty_id`      | Specialty                  | Yes      |
| `service_type_id`   | Service type               | Yes      |
| `contract_number`   | Contract reference         | Yes      |
| `contracted_volume` | Contracted service volume  | Yes      |
| `volume_period`     | Contract period            | Yes      |
| `effective_from`    | Start date                 | Yes      |
| `effective_to`      | End date                   | Yes      |
| `is_active`         | Active status              | Yes      |

### Business Rules

The baseline simulation uses:

```text
QUARTERLY NFZ SETTLEMENT
```

`volume_period` may remain configurable for future scenarios, but the baseline implementation assumes quarterly settlement.

NFZ-funded service capacity must not exceed the applicable contracted volume unless the simulation explicitly models over-delivery.

---

## 9.5 NFZ_CONTRACT_AMENDMENT

### Purpose

Represents a change to an existing NFZ contract.

### Grain

One record per contract amendment.

### Primary Key

```text
amendment_id
```

### Foreign Keys

```text
nfz_contract_id → NFZ_CONTRACT.nfz_contract_id
```

### Attributes

| Attribute                    | Description                       | Required |
| ---------------------------- | --------------------------------- | -------- |
| `amendment_id`               | Unique amendment identifier       | Yes      |
| `nfz_contract_id`            | Parent contract                   | Yes      |
| `amendment_date`             | Date amendment was introduced     | Yes      |
| `effective_from`             | Date new volume becomes effective | Yes      |
| `previous_contracted_volume` | Previous volume                   | Yes      |
| `new_contracted_volume`      | New volume                        | Yes      |
| `reason`                     | Business reason                   | No       |

### Business Rules

An amendment changes the contractual volume from:

```text
previous_contracted_volume
```

to:

```text
new_contracted_volume
```

The baseline model does not require a full administrative approval workflow.

---

# 10. Scenario and Simulation Entities

---

## 10.1 SCENARIO

### Purpose

Represents a simulation scenario or what-if analysis context.

### Grain

One record per scenario.

### Primary Key

```text
scenario_id
```

### Attributes

| Attribute       | Description                 | Required |
| --------------- | --------------------------- | -------- |
| `scenario_id`   | Unique scenario identifier  | Yes      |
| `scenario_name` | Scenario name               | Yes      |
| `scenario_type` | Scenario category           | Yes      |
| `description`   | Scenario description        | No       |
| `created_date`  | Creation date               | Yes      |
| `is_baseline`   | Indicates baseline scenario | Yes      |

### Example Scenarios

```text
BASELINE
ADD_DOCTORS
INCREASE_CAPACITY
REALLOCATE_RESOURCES
INCREASE_NFZ_CONTRACT
DECREASE_NFZ_CONTRACT
INCREASE_PRICES
```

---

## 10.2 SCENARIO_RESOURCE_CHANGE

### Purpose

Represents a hypothetical resource change applied to a simulation scenario.

### Grain

One record per scenario resource change.

### Primary Key

```text
scenario_resource_change_id
```

### Foreign Keys

```text
scenario_id → SCENARIO.scenario_id
clinic_id → CLINIC.clinic_id
```

### Attributes

| Attribute                     | Description                       | Required |
| ----------------------------- | --------------------------------- | -------- |
| `scenario_resource_change_id` | Unique identifier                 | Yes      |
| `scenario_id`                 | Scenario                          | Yes      |
| `clinic_id`                   | Affected clinic                   | Yes      |
| `resource_type`               | Resource category                 | Yes      |
| `quantity_change`             | Delta from baseline/current state | Yes      |
| `effective_from`              | Scenario effective date           | Yes      |
| `effective_to`                | Scenario end date                 | No       |
| `description`                 | Change description                | No       |

### Business Rules

`quantity_change` represents a **delta**, not an absolute replacement value.

Example:

```text
Current Doctors = 10
quantity_change = +2

New Scenario State = 12 Doctors
```

This entity represents a hypothetical scenario assumption.

It is distinct from:

```text
RESOURCE_REALLOCATION
```

which represents an actual operational event.

---

# 11. Relationship Summary

## 11.1 Patient Flow

```text
PATIENT
   │
   ├───────────────┐
   │               │
   ↓               ↓
PATIENT_VISIT   DEMAND_EVENT
   │               │
   ↓               │
REFERRAL ──────────┘
   │
   ↓
PATIENT_DECISION
   │
   ↓
WAITING_LIST_ENTRY
   │
   ↓
APPOINTMENT
   │
   ↓
PATIENT_VISIT
   │
   ↓
SERVICE_DELIVERY
```

The model supports both:

```text
Referral-driven demand
```

and:

```text
Non-referral demand
```

---

## 11.2 Service and Revenue Flow

```text
SERVICE_DELIVERY
        ↓
SERVICE_PRICING
        ↓
REVENUE_TRANSACTION
        ↓
DOCTOR_COMPENSATION_TRANSACTION
```

The service pricing selection is based on:

```text
SERVICE_TYPE
+
PAYER_TYPE
+
SERVICE_DATE
```

---

## 11.3 Capacity Flow

```text
CLINIC
   +
SPECIALTY
   +
SERVICE_TYPE
   +
NFZ_CONTRACT
   +
RESOURCE_REALLOCATION
        ↓
CAPACITY_FACT
        ↓
SERVICE_DELIVERY
```

---

## 11.4 Scenario Flow

```text
SCENARIO
   ↓
SCENARIO_RESOURCE_CHANGE
   ↓
Simulated Capacity / Demand / Operations
```

---

# 12. Business Rule Traceability

| Business Rule Area              | Primary Entity                    |
| ------------------------------- | --------------------------------- |
| Clinic network                  | `CLINIC`                          |
| Specialty catalog               | `SPECIALTY`                       |
| Internal specialty availability | `CLINIC_SPECIALTY`                |
| First visit / follow-up         | `SERVICE_TYPE`                    |
| Patient population              | `PATIENT`                         |
| Demand generation               | `DEMAND_EVENT`                    |
| Referral process                | `REFERRAL`                        |
| Patient behavior                | `PATIENT_DECISION`                |
| Waiting time                    | `WAITING_LIST_ENTRY`              |
| Appointment scheduling          | `APPOINTMENT`                     |
| Patient encounters              | `PATIENT_VISIT`                   |
| Service delivery                | `SERVICE_DELIVERY`                |
| Capacity                        | `CAPACITY_FACT`                   |
| Resource movement               | `RESOURCE_REALLOCATION`           |
| Service price                   | `SERVICE_PRICING`                 |
| Revenue                         | `REVENUE_TRANSACTION`             |
| Doctor compensation             | `DOCTOR_COMPENSATION_RULE`        |
| Compensation transactions       | `DOCTOR_COMPENSATION_TRANSACTION` |
| NFZ funding                     | `NFZ_CONTRACT`                    |
| NFZ contract changes            | `NFZ_CONTRACT_AMENDMENT`          |
| Patient behavior probabilities  | `BEHAVIOR_RULE`                   |
| What-if scenarios               | `SCENARIO`                        |
| Scenario resource assumptions   | `SCENARIO_RESOURCE_CHANGE`        |

---

# 13. Key Data Integrity Rules

## 13.1 Referential Integrity

Every foreign key must reference an existing parent record.

Examples:

```text
SERVICE_DELIVERY.doctor_id
→ DOCTOR.doctor_id
```

```text
REVENUE_TRANSACTION.service_delivery_id
→ SERVICE_DELIVERY.service_delivery_id
```

```text
REFERRAL.patient_id
→ PATIENT.patient_id
```

---

## 13.2 Temporal Integrity

Effective periods must be logically valid.

```text
effective_from <= effective_to
```

when `effective_to` is populated.

Historical rules must be selected based on the transaction date.

Examples:

```text
SERVICE_DELIVERY.service_date
→ SERVICE_PRICING effective period
```

```text
SERVICE_DELIVERY.service_date
→ DOCTOR_COMPENSATION_RULE effective period
```

```text
SERVICE_DELIVERY.service_date
→ CLINIC_SPECIALTY availability
```

---

## 13.3 Referral Integrity

A referral must have:

```text
patient_id
target_specialty_id
referral_date
```

`final_outcome` must remain null while the referral is non-terminal.

---

## 13.4 Patient Decision Integrity

Each `PATIENT_DECISION` represents one behavioral event.

A referral may have multiple patient decisions.

Example:

```text
REFERRAL 1001

PATIENT_DECISION
Day 30 → WAIT

PATIENT_DECISION
Day 60 → WAIT

PATIENT_DECISION
Day 90 → EXTERNAL
```

The final state is stored in:

```text
REFERRAL.final_outcome = EXTERNAL
```

---

## 13.5 Pricing Integrity

At most one applicable pricing record may exist for:

```text
SERVICE_TYPE
+
PAYER_TYPE
+
SERVICE_DATE
```

---

## 13.6 Compensation Integrity

The compensation rule used must be valid on the service delivery date.

The transaction should preserve the applied calculation method.

---

## 13.7 Capacity Integrity

Capacity cannot be negative.

```text
operational_capacity >= 0
nfz_funded_capacity >= 0
effective_capacity >= 0
actual_delivered_services >= 0
```

---

## 13.8 Resource Reallocation Integrity

For a reallocation event:

```text
source_quantity >= 0
target_quantity >= 0
```

The source and target clinic should not be identical unless the simulation explicitly supports internal resource adjustments.

---

# 14. Derived Metrics

The following metrics may be derived from the logical model.

## 14.1 Waiting Time

```text
Waiting Days
=
Queue Exit Date
-
Queue Entry Date
```

For active queue entries:

```text
Waiting Days
=
Current Date
-
Queue Entry Date
```

---

## 14.2 Utilization

```text
Utilization Rate
=
Actual Delivered Services
/
Effective Capacity
```

---

## 14.3 Clinic Revenue

```text
Clinic Revenue
=
Gross Service Revenue
-
Doctor Compensation
```

---

## 14.4 Doctor Compensation

```text
PERCENTAGE
=
Gross Service Revenue
×
Percentage Rate
```

```text
HOURLY_RATE
=
Doctor Hours
×
Hourly Rate
```

```text
MIXED
=
Percentage Component
+
Hourly Component
```

---

## 14.5 Patient Leakage

Patient leakage may be calculated as:

```text
External Outcomes
/
Total Relevant Demand
```

or:

```text
External Referrals
/
Total Referral Population
```

The exact denominator should be explicitly selected at the analytical layer.

---

## 14.6 Referral Completion Rate

```text
Completed Internal Referrals
/
Total Relevant Referrals
```

---

## 14.7 Queue Abandonment Rate

```text
Dropped Patients
+
External Patients
+
Cancelled Patients
--------------------------------
Total Queue Entries
```

The exact business definition may be refined in the analytical layer.

---

# 15. Analytical vs Transactional Separation

The model intentionally distinguishes between transactional and analytical entities.

## Transactional

```text
REFERRAL
PATIENT_DECISION
APPOINTMENT
PATIENT_VISIT
SERVICE_DELIVERY
REVENUE_TRANSACTION
DOCTOR_COMPENSATION_TRANSACTION
RESOURCE_REALLOCATION
```

## Analytical / Snapshot

```text
CAPACITY_FACT
```

## Master / Reference

```text
PATIENT
CLINIC
SPECIALTY
SERVICE_TYPE
PAYER_TYPE
DOCTOR
```

## Configuration / Rules

```text
BEHAVIOR_RULE
DOCTOR_COMPENSATION_RULE
SERVICE_PRICING
NFZ_CONTRACT
```

## Scenario

```text
SCENARIO
SCENARIO_RESOURCE_CHANGE
```

This separation should be preserved when implementing the SQL schema.

---

# 16. Known Modeling Assumptions

The following assumptions are intentionally retained for the simulation:

1. The baseline organization contains approximately 11 specialist clinics.
2. The specialty catalog contains approximately 20–25 specialties.
3. Not every specialty is available internally.
4. Internal specialty availability is represented through `CLINIC_SPECIALTY`.
5. A patient may generate multiple demand events.
6. A patient may generate multiple referrals.
7. A referral may contain multiple behavioral decisions.
8. `PATIENT_DECISION` represents behavioral events.
9. `REFERRAL.final_outcome` represents the final business outcome.
10. Not every demand event requires a referral.
11. First visits and follow-up visits are represented through `SERVICE_TYPE.visit_type`.
12. NFZ settlement is quarterly in the baseline model.
13. Pricing is selected by service type, payer type, and service date.
14. Doctor compensation rules are selected by doctor and service delivery date.
15. Mixed compensation consists of percentage and hourly components.
16. Scenario resource changes represent deltas from the baseline/current scenario state.
17. Actual resource reallocations are represented separately through `RESOURCE_REALLOCATION`.
18. Capacity snapshots are represented through `CAPACITY_FACT`.
19. Transaction-level service delivery remains the source of operational truth.
20. All patient data is synthetic and anonymized.

---

# 17. Future Extensions

The following entities are intentionally not required for the current MVP but may be introduced later.

## 17.1 QUEUE_SNAPSHOT

Would provide daily or periodic snapshots of queue size.

Potential grain:

```text
Date
+
Clinic
+
Specialty
+
Service Type
```

Useful for:

* queue trend analysis,
* historical queue size,
* backlog monitoring.

---

## 17.2 CLINIC_SERVICE_TYPE

Would explicitly represent which service types are available at each clinic.

Potential grain:

```text
Clinic
+
Service Type
+
Effective Period
```

This entity is not required for the MVP because service availability can initially be inferred from capacity, scheduling, and pricing.

---

## 17.3 DOCTOR_CLINIC

Would explicitly represent doctor-to-clinic assignments over time.

Potential grain:

```text
Doctor
+
Clinic
+
Effective Period
```

This may become necessary if doctors work across multiple clinics with changing assignments.

---

## 17.4 RESOURCE

Would provide a normalized resource catalog.

Potential resource types:

```text
DOCTOR
DOCTOR_HOUR
ROOM
EQUIPMENT
CAPACITY
```

---

# 18. Entity Dictionary Completion Criteria

This document is considered complete for the logical modeling phase when:

* all entities have defined grains,
* all primary keys are identified,
* all foreign keys are identified,
* core business rules are represented,
* temporal rules are documented,
* patient behavior and final referral outcomes are separated,
* pricing selection logic is defined,
* compensation logic is defined,
* scenario logic is separated from operational events,
* transactional and analytical entities are distinguished.

At version 1.3, the model is considered:

```text
READY FOR FINAL AUDIT
```

The next recommended document is:

```text
05_Process_Flows.md
```

After the process flows are validated, the project should proceed to:

```text
06_ERD.md
```

The ERD should be generated only after the process flows confirm the direction and cardinality of the main relationships.
