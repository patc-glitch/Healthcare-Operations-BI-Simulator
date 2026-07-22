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
Entity Relationship Diagram (ERD)
       ↓
Data Generator Architecture
       ↓
Python Data Generator
       ↓
PostgreSQL Data Model
       ↓
BI / Tableau Analytics
```

The purpose of this document is to establish:

* the core entities of the simulator,
* the business meaning of each entity,
* primary keys,
* foreign keys,
* key attributes,
* logical relationships,
* ownership of business concepts,
* data-generation dependencies.

This document describes the **logical model**.

It does not yet define:

* final PostgreSQL data types,
* indexes,
* partitioning,
* physical table optimization,
* SQL implementation details,
* exact Tableau data sources.

Those decisions will be addressed during the ERD and technical architecture phases.

---

# 2. Modeling Principles

The data model follows several principles.

## 2.1 Separate Master Data from Operational Data

Stable reference entities should be separated from events and transactions.

Examples:

```text
Master Data
├── Facility
├── Clinic
├── Specialty
├── Doctor
├── Room
└── Service Type
```

Operational data includes:

```text
Operational Data
├── Patient
├── Referral
├── Appointment
├── Visit
├── Waiting List Entry
└── Service Delivery
```

Financial and analytical data is modeled separately from operational activity.

---

## 2.2 Model Demand and Capacity Separately

Healthcare operations are driven by the relationship between:

```text
Demand
    ↕
Capacity
```

Demand should not be inferred exclusively from completed visits.

A patient may:

* request a service,
* generate a referral,
* enter a waiting list,
* wait for an appointment,
* cancel,
* fail to attend,
* complete a visit.

Therefore, demand and capacity must remain analytically distinguishable.

---

## 2.3 Preserve Operational Events

The model should retain the sequence of important operational events.

For example:

```text
Referral Created
      ↓
Referral Accepted
      ↓
Waiting List Entry
      ↓
Appointment Scheduled
      ↓
Appointment Completed
      ↓
Visit Delivered
      ↓
Revenue Recognized
```

This enables calculation of:

* referral conversion,
* waiting time,
* cancellation rate,
* no-show rate,
* capacity utilization,
* throughput,
* revenue per visit.

---

## 2.4 Support Multiple Funding Models

The model must distinguish between:

```text
NFZ-funded activity
        vs.
Commercial activity
```

The same clinical service may therefore have different:

* funding sources,
* prices,
* reimbursement rules,
* capacity constraints,
* revenue outcomes.

---

## 2.5 Support Scenario Analysis

The simulator is intended to support operational and strategic scenarios.

The model should therefore allow scenario parameters to modify:

* demand,
* capacity,
* doctor availability,
* room availability,
* referral volume,
* pricing,
* NFZ capacity,
* commercial capacity,
* operating costs.

Scenario logic should be separated from historical or baseline operational facts.

---

# 3. Entity Domains

The logical model is divided into the following domains.

```text
1. Organization & Master Data
2. Patient & Demand
3. Referral Management
4. Capacity & Scheduling
5. Healthcare Operations
6. Funding & Revenue
7. Costs
8. Analytics
9. Scenario Modeling
```

---

# 4. Organization & Master Data

## 4.1 FACILITY

### Purpose

Represents the healthcare organization or physical operating location modeled by the simulator.

### Key

```text
facility_id — PK
```

### Core Attributes

| Attribute     | Key | Description                               |
| ------------- | --- | ----------------------------------------- |
| facility_id   | PK  | Unique facility identifier                |
| facility_name |     | Facility name                             |
| facility_type |     | Type of healthcare facility               |
| city          |     | City                                      |
| region        |     | Geographic region                         |
| active_flag   |     | Indicates whether facility is operational |

### Relationships

```text
FACILITY
   ├── CLINIC
   ├── ROOM
   └── DOCTOR
```

### Business Rules

* A facility may contain multiple clinics.
* A facility may contain multiple rooms.
* A facility may employ or host multiple doctors.
* Operational activity should occur only during the facility's active period.

---

## 4.2 CLINIC

### Purpose

Represents an operational clinical unit or clinic within the healthcare organization.

Examples include:

* POZ,
* Pediatrics,
* Cardiology,
* Dermatology,
* Orthopedics,
* other specialist clinics.

### Key

```text
clinic_id — PK
```

### Foreign Keys

```text
facility_id — FK → FACILITY
```

### Core Attributes

| Attribute   | Key | Description                      |
| ----------- | --- | -------------------------------- |
| clinic_id   | PK  | Unique clinic identifier         |
| facility_id | FK  | Parent facility                  |
| clinic_name |     | Clinic name                      |
| clinic_type |     | POZ, specialist, pediatric, etc. |
| active_flag |     | Operational status               |

### Relationships

```text
FACILITY
   │
   └── CLINIC
          ├── DOCTOR
          ├── ROOM
          ├── SERVICE_TYPE
          ├── APPOINTMENT
          └── VISIT
```

---

## 4.3 SPECIALTY

### Purpose

Represents a medical specialty or clinical discipline.

### Key

```text
specialty_id — PK
```

### Core Attributes

| Attribute       | Key | Description                 |
| --------------- | --- | --------------------------- |
| specialty_id    | PK  | Unique specialty identifier |
| specialty_name  |     | Specialty name              |
| specialty_group |     | Higher-level grouping       |
| active_flag     |     | Whether specialty is active |

### Relationships

```text
SPECIALTY
   ├── DOCTOR
   ├── REFERRAL
   ├── SERVICE_TYPE
   └── WAITING_LIST_ENTRY
```

### Business Rules

* A specialty may be available or unavailable within the organization.
* A specialty may have different levels of demand.
* A specialty may have dedicated or shared capacity.
* Specialty availability affects referral destinations and waiting lists.

---

## 4.4 DOCTOR

### Purpose

Represents a healthcare professional providing services within the organization.

### Key

```text
doctor_id — PK
```

### Foreign Keys

```text
clinic_id — FK → CLINIC
specialty_id — FK → SPECIALTY
```

### Core Attributes

| Attribute          | Key | Description                |
| ------------------ | --- | -------------------------- |
| doctor_id          | PK  | Unique doctor identifier   |
| clinic_id          | FK  | Primary clinic             |
| specialty_id       | FK  | Primary specialty          |
| doctor_name        |     | Display name               |
| employment_type    |     | Employment arrangement     |
| compensation_model |     | Compensation structure     |
| active_flag        |     | Current operational status |

### Relationships

```text
DOCTOR
   ├── SCHEDULE
   ├── APPOINTMENT
   ├── VISIT
   └── REFERRAL
```

### Business Rules

* A doctor must be associated with a valid specialty.
* A doctor may provide multiple visits.
* A doctor may receive referrals.
* Doctor availability contributes to clinic capacity.
* Compensation model affects cost calculations.

---

## 4.5 ROOM

### Purpose

Represents a physical room or treatment space used to provide healthcare services.

### Key

```text
room_id — PK
```

### Foreign Keys

```text
facility_id — FK → FACILITY
clinic_id — FK → CLINIC
```

### Core Attributes

| Attribute   | Key | Description                               |
| ----------- | --- | ----------------------------------------- |
| room_id     | PK  | Unique room identifier                    |
| facility_id | FK  | Facility                                  |
| clinic_id   | FK  | Clinic                                    |
| room_name   |     | Room identifier                           |
| room_type   |     | Consultation, procedure, diagnostic, etc. |
| active_flag |     | Operational status                        |

### Business Rules

* Room availability contributes to operational capacity.
* A room may be unavailable during maintenance or other downtime.
* Room capacity may constrain the number of services delivered.

---

## 4.6 SERVICE_TYPE

### Purpose

Defines a healthcare service that can be requested, scheduled, delivered, and potentially reimbursed.

### Key

```text
service_type_id — PK
```

### Foreign Keys

```text
specialty_id — FK → SPECIALTY
```

### Core Attributes

| Attribute                 | Key | Description                       |
| ------------------------- | --- | --------------------------------- |
| service_type_id           | PK  | Unique service identifier         |
| specialty_id              | FK  | Related specialty                 |
| service_name              |     | Service name                      |
| service_category          |     | Service classification            |
| funding_type              |     | NFZ, commercial, or both          |
| standard_duration_minutes |     | Expected service duration         |
| standard_price            |     | Commercial price where applicable |
| active_flag               |     | Service availability              |

### Business Rules

* A service belongs to a specialty or clinical service line.
* A service may be available under NFZ, commercially, or both.
* Standard duration contributes to capacity calculations.
* Commercial services may generate direct revenue.

---

# 5. Patient & Demand

## 5.1 PATIENT

### Purpose

Represents an individual who generates healthcare demand and may receive services.

### Key

```text
patient_id — PK
```

### Core Attributes

| Attribute         | Key | Description                     |
| ----------------- | --- | ------------------------------- |
| patient_id        | PK  | Unique patient identifier       |
| date_of_birth     |     | Date of birth                   |
| gender            |     | Gender                          |
| registration_date |     | Date registered in organization |
| residence_region  |     | Geographic area                 |
| patient_segment   |     | Patient segmentation category   |

### Relationships

```text
PATIENT
   ├── DEMAND
   ├── REFERRAL
   ├── WAITING_LIST_ENTRY
   ├── APPOINTMENT
   └── VISIT
```

### Business Rules

* A patient may generate multiple demand events.
* A patient may have multiple referrals.
* A patient may enter multiple waiting lists over time.
* A patient may have multiple appointments and visits.

---

## 5.2 PATIENT_DEMAND

### Purpose

Represents a demand event for healthcare services.

This entity is critical for distinguishing **demand** from **completed healthcare activity**.

### Key

```text
demand_id — PK
```

### Foreign Keys

```text
patient_id — FK → PATIENT
specialty_id — FK → SPECIALTY
service_type_id — FK → SERVICE_TYPE
```

### Core Attributes

| Attribute       | Key | Description                              |
| --------------- | --- | ---------------------------------------- |
| demand_id       | PK  | Unique demand event                      |
| patient_id      | FK  | Patient generating demand                |
| specialty_id    | FK  | Requested specialty                      |
| service_type_id | FK  | Requested service                        |
| demand_date     |     | Date demand was generated                |
| demand_source   |     | Referral, direct booking, internal, etc. |
| urgency         |     | Demand urgency                           |
| demand_status   |     | Current demand status                    |

### Business Rules

* Demand does not necessarily result in a completed visit.
* Demand may result in a referral.
* Demand may enter a waiting list.
* Demand may be lost or converted to an external provider.
* Demand volume should be measurable independently of throughput.

---

# 6. Referral Management

## 6.1 REFERRAL

### Purpose

Represents a referral from one healthcare pathway to another.

### Key

```text
referral_id — PK
```

### Foreign Keys

```text
patient_id — FK → PATIENT
referring_doctor_id — FK → DOCTOR
target_specialty_id — FK → SPECIALTY
```

### Core Attributes

| Attribute            | Key | Description              |
| -------------------- | --- | ------------------------ |
| referral_id          | PK  | Unique referral          |
| patient_id           | FK  | Referred patient         |
| referring_doctor_id  | FK  | Referring doctor         |
| target_specialty_id  | FK  | Target specialty         |
| referral_date        |     | Date referral was issued |
| referral_status      |     | Referral status          |
| referral_destination |     | Internal or external     |
| urgency              |     | Referral priority        |

### Relationships

```text
REFERRAL
   └── WAITING_LIST_ENTRY
           └── APPOINTMENT
                   └── VISIT
```

### Business Rules

* A referral must belong to a patient.
* A referral should identify a target specialty.
* A referral may be fulfilled internally.
* A referral may leak to an external provider.
* Referral leakage is an important operational KPI.

---

## 6.2 REFERRAL_OUTCOME

### Purpose

Represents the final outcome of a referral.

### Key

```text
referral_outcome_id — PK
```

### Foreign Keys

```text
referral_id — FK → REFERRAL
```

### Core Attributes

| Attribute           | Key | Description                        |
| ------------------- | --- | ---------------------------------- |
| referral_outcome_id | PK  | Unique outcome record              |
| referral_id         | FK  | Related referral                   |
| outcome_type        |     | Completed, leaked, cancelled, etc. |
| outcome_date        |     | Date outcome occurred              |
| internal_flag       |     | Internal vs. external completion   |

### Business Rules

A referral should ultimately be classified into a measurable outcome.

---

# 7. Capacity & Scheduling

## 7.1 DOCTOR_SCHEDULE

### Purpose

Represents the planned availability of a doctor.

### Key

```text
doctor_schedule_id — PK
```

### Foreign Keys

```text
doctor_id — FK → DOCTOR
clinic_id — FK → CLINIC
```

### Core Attributes

| Attribute          | Key | Description               |
| ------------------ | --- | ------------------------- |
| doctor_schedule_id | PK  | Schedule record           |
| doctor_id          | FK  | Doctor                    |
| clinic_id          | FK  | Clinic                    |
| schedule_date      |     | Date                      |
| available_hours    |     | Planned working hours     |
| available_slots    |     | Planned appointment slots |
| unavailable_hours  |     | Non-available time        |

### Business Rules

* Doctor schedules define part of available capacity.
* Capacity must not be greater than available working time.
* Schedules may vary by day and specialty.

---

## 7.2 CLINIC_CAPACITY

### Purpose

Represents planned healthcare service capacity available to patients.

### Key

```text
capacity_id — PK
```

### Foreign Keys

```text
clinic_id — FK → CLINIC
specialty_id — FK → SPECIALTY
service_type_id — FK → SERVICE_TYPE
```

### Core Attributes

| Attribute       | Key | Description             |
| --------------- | --- | ----------------------- |
| capacity_id     | PK  | Capacity record         |
| clinic_id       | FK  | Clinic                  |
| specialty_id    | FK  | Specialty               |
| service_type_id | FK  | Service                 |
| capacity_date   |     | Date                    |
| available_slots |     | Available service slots |
| available_hours |     | Available service hours |
| funding_type    |     | NFZ or commercial       |

### Business Rules

Capacity is affected by:

* doctor availability,
* room availability,
* working hours,
* service duration,
* funding constraints.

Capacity should be measurable independently of actual utilization.

---

## 7.3 APPOINTMENT

### Purpose

Represents a scheduled patient appointment.

### Key

```text
appointment_id — PK
```

### Foreign Keys

```text
patient_id — FK → PATIENT
doctor_id — FK → DOCTOR
clinic_id — FK → CLINIC
service_type_id — FK → SERVICE_TYPE
referral_id — FK → REFERRAL
```

### Core Attributes

| Attribute           | Key | Description                              |
| ------------------- | --- | ---------------------------------------- |
| appointment_id      | PK  | Unique appointment                       |
| patient_id          | FK  | Patient                                  |
| doctor_id           | FK  | Doctor                                   |
| clinic_id           | FK  | Clinic                                   |
| service_type_id     | FK  | Service                                  |
| referral_id         | FK  | Related referral                         |
| scheduled_datetime  |     | Scheduled date/time                      |
| appointment_status  |     | Scheduled, completed, cancelled, no-show |
| booking_datetime    |     | Booking timestamp                        |
| cancellation_reason |     | Cancellation reason                      |

### Business Rules

* An appointment consumes planned capacity.
* A cancelled appointment releases or wastes capacity depending on timing.
* A no-show consumes capacity without producing equivalent throughput.
* A completed appointment should normally result in a visit.

---

## 7.4 WAITING_LIST_ENTRY

### Purpose

Represents a patient waiting for access to a healthcare service.

### Key

```text
waiting_list_entry_id — PK
```

### Foreign Keys

```text
patient_id — FK → PATIENT
referral_id — FK → REFERRAL
specialty_id — FK → SPECIALTY
service_type_id — FK → SERVICE_TYPE
```

### Core Attributes

| Attribute             | Key | Description                              |
| --------------------- | --- | ---------------------------------------- |
| waiting_list_entry_id | PK  | Unique waiting list record               |
| patient_id            | FK  | Patient                                  |
| referral_id           | FK  | Related referral                         |
| specialty_id          | FK  | Specialty                                |
| service_type_id       | FK  | Requested service                        |
| entry_date            |     | Date added to waiting list               |
| appointment_date      |     | Date eventually scheduled                |
| exit_date             |     | Date removed from waiting list           |
| waiting_status        |     | Waiting, scheduled, completed, cancelled |
| priority              |     | Priority level                           |

### Business Rules

Waiting list data must support calculation of:

* current waiting list size,
* average waiting time,
* median waiting time,
* maximum waiting time,
* waiting time by specialty,
* waiting time by service,
* waiting list aging.

---

# 8. Healthcare Operations

## 8.1 VISIT

### Purpose

Represents a completed healthcare interaction.

This is the primary operational throughput event.

### Key

```text
visit_id — PK
```

### Foreign Keys

```text
appointment_id — FK → APPOINTMENT
patient_id — FK → PATIENT
doctor_id — FK → DOCTOR
clinic_id — FK → CLINIC
service_type_id — FK → SERVICE_TYPE
```

### Core Attributes

| Attribute        | Key | Description                |
| ---------------- | --- | -------------------------- |
| visit_id         | PK  | Unique visit               |
| appointment_id   | FK  | Source appointment         |
| patient_id       | FK  | Patient                    |
| doctor_id        | FK  | Doctor                     |
| clinic_id        | FK  | Clinic                     |
| service_type_id  | FK  | Service                    |
| visit_datetime   |     | Actual service date/time   |
| visit_status     |     | Completed or other outcome |
| funding_type     |     | NFZ or commercial          |
| duration_minutes |     | Actual duration            |

### Business Rules

* A completed visit consumes capacity.
* A completed visit should normally originate from an appointment.
* Visit volume represents throughput.
* Visits should be traceable to funding and revenue where applicable.

---

## 8.2 SERVICE_DELIVERY

### Purpose

Represents the detailed delivery of a healthcare service when one visit may contain multiple service components.

### Key

```text
service_delivery_id — PK
```

### Foreign Keys

```text
visit_id — FK → VISIT
service_type_id — FK → SERVICE_TYPE
```

### Core Attributes

| Attribute           | Key | Description             |
| ------------------- | --- | ----------------------- |
| service_delivery_id | PK  | Unique service delivery |
| visit_id            | FK  | Visit                   |
| service_type_id     | FK  | Delivered service       |
| quantity            |     | Quantity delivered      |
| unit_price          |     | Applicable price        |
| gross_value         |     | Gross service value     |

### Business Rules

* A visit may contain one or more delivered services.
* Service delivery is the preferred grain for detailed revenue analysis where required.

---

# 9. Funding & Revenue

## 9.1 PAYER

### Purpose

Represents the funding source responsible for reimbursing healthcare services.

### Key

```text
payer_id — PK
```

### Core Attributes

| Attribute   | Key | Description                     |
| ----------- | --- | ------------------------------- |
| payer_id    | PK  | Unique payer                    |
| payer_name  |     | Payer name                      |
| payer_type  |     | NFZ, commercial, patient, other |
| active_flag |     | Active status                   |

---

## 9.2 NFZ_CONTRACT

### Purpose

Represents an agreement or funding arrangement governing NFZ-funded healthcare activity.

### Key

```text
nfz_contract_id — PK
```

### Foreign Keys

```text
clinic_id — FK → CLINIC
specialty_id — FK → SPECIALTY
```

### Core Attributes

| Attribute             | Key | Description                |
| --------------------- | --- | -------------------------- |
| nfz_contract_id       | PK  | Contract identifier        |
| clinic_id             | FK  | Clinic                     |
| specialty_id          | FK  | Specialty                  |
| contract_period_start |     | Contract start             |
| contract_period_end   |     | Contract end               |
| contracted_volume     |     | Contracted service volume  |
| contracted_value      |     | Contracted financial value |
| active_flag           |     | Contract status            |

### Business Rules

* NFZ activity is constrained by contractual terms.
* Contracted volume may differ from actual demand.
* Contract utilization should be measurable.
* Unused capacity and exceeded contract limits should be analytically visible.

---

## 9.3 NFZ_CONTRACT_LIMIT

### Purpose

Represents operational limits associated with an NFZ contract.

### Key

```text
nfz_contract_limit_id — PK
```

### Foreign Keys

```text
nfz_contract_id — FK → NFZ_CONTRACT
service_type_id — FK → SERVICE_TYPE
```

### Core Attributes

| Attribute             | Key | Description           |
| --------------------- | --- | --------------------- |
| nfz_contract_limit_id | PK  | Limit identifier      |
| nfz_contract_id       | FK  | Contract              |
| service_type_id       | FK  | Service               |
| period_start          |     | Period start          |
| period_end            |     | Period end            |
| volume_limit          |     | Maximum funded volume |
| value_limit           |     | Maximum funded value  |

---

## 9.4 REVENUE_TRANSACTION

### Purpose

Represents financial value generated by healthcare activity.

### Key

```text
revenue_transaction_id — PK
```

### Foreign Keys

```text
visit_id — FK → VISIT
service_delivery_id — FK → SERVICE_DELIVERY
payer_id — FK → PAYER
```

### Core Attributes

| Attribute              | Key | Description            |
| ---------------------- | --- | ---------------------- |
| revenue_transaction_id | PK  | Revenue record         |
| visit_id               | FK  | Related visit          |
| service_delivery_id    | FK  | Related service        |
| payer_id               | FK  | Funding source         |
| transaction_date       |     | Revenue date           |
| funding_type           |     | NFZ or commercial      |
| gross_revenue          |     | Gross revenue          |
| net_revenue            |     | Net recognized revenue |

### Business Rules

Revenue should support analysis by:

* clinic,
* specialty,
* doctor,
* service,
* payer,
* funding type,
* period.

---

# 10. Costs

## 10.1 COST_CATEGORY

### Purpose

Defines categories used to classify operating costs.

### Key

```text
cost_category_id — PK
```

### Core Attributes

| Attribute        | Key | Description       |
| ---------------- | --- | ----------------- |
| cost_category_id | PK  | Cost category     |
| category_name    |     | Category name     |
| cost_type        |     | Fixed or variable |
| active_flag      |     | Active status     |

Examples:

```text
Doctor Compensation
Room Costs
Facility Costs
Administrative Costs
Technology
Supplies
Other Operating Costs
```

---

## 10.2 OPERATING_COST

### Purpose

Represents operating expenses incurred by the healthcare organization.

### Key

```text
operating_cost_id — PK
```

### Foreign Keys

```text
cost_category_id — FK → COST_CATEGORY
clinic_id — FK → CLINIC
```

### Core Attributes

| Attribute           | Key | Description                      |
| ------------------- | --- | -------------------------------- |
| operating_cost_id   | PK  | Cost record                      |
| cost_category_id    | FK  | Cost category                    |
| clinic_id           | FK  | Related clinic                   |
| cost_date           |     | Cost date                        |
| cost_amount         |     | Cost amount                      |
| fixed_variable_flag |     | Fixed or variable classification |

### Business Rules

Operating costs should support calculation of:

```text
Revenue
- Operating Costs
= Operating Profit
```

Costs should be attributable to the appropriate organizational level where possible.

---

# 11. Analytics

## 11.1 OPERATIONAL_KPI

### Purpose

Represents calculated operational performance indicators.

This entity should generally be treated as a derived analytical layer rather than a source transaction.

### Key

```text
operational_kpi_id — PK
```

### Possible Metrics

```text
Patient Volume
Referral Volume
Referral Conversion Rate
Referral Leakage Rate
Appointment Volume
Cancellation Rate
No-Show Rate
Visit Volume
Capacity
Capacity Utilization
Waiting List Size
Average Waiting Time
Median Waiting Time
```

### Business Rules

KPIs should be calculated from source entities whenever possible.

The source-of-truth entities are:

```text
Demand      → PATIENT_DEMAND
Referrals   → REFERRAL
Capacity    → CLINIC_CAPACITY
Scheduling  → APPOINTMENT
Throughput  → VISIT
Revenue     → REVENUE_TRANSACTION
Costs       → OPERATING_COST
```

---

## 11.2 FINANCIAL_KPI

### Purpose

Represents calculated financial performance metrics.

### Possible Metrics

```text
Total Revenue
NFZ Revenue
Commercial Revenue
Revenue per Visit
Operating Costs
Operating Margin
Revenue Mix
Cost per Visit
```

Financial KPIs should be derived from:

```text
REVENUE_TRANSACTION
        +
OPERATING_COST
```

---

# 12. Scenario Modeling

## 12.1 SCENARIO

### Purpose

Represents a simulated operational or strategic scenario.

Examples:

```text
Baseline
Increase Doctor Capacity
Add New Specialty
Increase Commercial Pricing
Increase NFZ Contract
Reduce Waiting Time
Increase Referral Conversion
Add Additional Rooms
```

### Key

```text
scenario_id — PK
```

### Core Attributes

| Attribute     | Key | Description                 |
| ------------- | --- | --------------------------- |
| scenario_id   | PK  | Unique scenario             |
| scenario_name |     | Scenario name               |
| scenario_type |     | Strategic or operational    |
| description   |     | Scenario description        |
| baseline_flag |     | Indicates baseline scenario |

---

## 12.2 SCENARIO_PARAMETER

### Purpose

Defines variables modified by a scenario.

### Key

```text
scenario_parameter_id — PK
```

### Foreign Keys

```text
scenario_id — FK → SCENARIO
```

### Possible Parameters

```text
Doctor Capacity
Room Capacity
Appointment Slots
Demand Growth
Referral Volume
Referral Conversion
NFZ Contract Volume
Commercial Price
Operating Cost
```

---

## 12.3 SCENARIO_RESULT

### Purpose

Stores or represents calculated outputs from a scenario simulation.

### Key

```text
scenario_result_id — PK
```

### Foreign Keys

```text
scenario_id — FK → SCENARIO
```

### Possible Outputs

```text
Total Visits
Waiting List Size
Average Waiting Time
Capacity Utilization
Revenue
Operating Costs
Operating Margin
Referral Leakage
```

Scenario results should be directly comparable with the baseline scenario.

---

# 13. Entity Relationship Overview

The high-level logical relationship structure is:

```text
FACILITY
   │
   ├── CLINIC
   │      │
   │      ├── DOCTOR
   │      │      ├── DOCTOR_SCHEDULE
   │      │      ├── APPOINTMENT
   │      │      └── VISIT
   │      │
   │      ├── ROOM
   │      ├── CLINIC_CAPACITY
   │      └── SERVICE_TYPE
   │
   └── ...

SPECIALTY
   ├── DOCTOR
   ├── SERVICE_TYPE
   ├── REFERRAL
   └── CLINIC_CAPACITY


PATIENT
   │
   ├── PATIENT_DEMAND
   ├── REFERRAL
   ├── WAITING_LIST_ENTRY
   ├── APPOINTMENT
   └── VISIT


REFERRAL
   │
   └── WAITING_LIST_ENTRY
           │
           └── APPOINTMENT
                   │
                   └── VISIT
                           │
                           └── SERVICE_DELIVERY
                                   │
                                   └── REVENUE_TRANSACTION


NFZ_CONTRACT
   │
   └── NFZ_CONTRACT_LIMIT


COST_CATEGORY
   │
   └── OPERATING_COST


SCENARIO
   ├── SCENARIO_PARAMETER
   └── SCENARIO_RESULT
```

---

# 14. Core Operational Flow

The primary operational flow is:

```text
PATIENT
   ↓
PATIENT_DEMAND
   ↓
REFERRAL
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

The operational flow is constrained by:

```text
DOCTOR_SCHEDULE
        +
ROOM
        +
CLINIC_CAPACITY
        +
NFZ_CONTRACT
```

Financial performance is affected by:

```text
REVENUE_TRANSACTION
        -
OPERATING_COST
        =
OPERATING_RESULT
```

Strategic scenarios modify selected parameters and generate alternative outcomes:

```text
SCENARIO
   ↓
SCENARIO_PARAMETER
   ↓
Operational Simulation
   ↓
SCENARIO_RESULT
```

---

# 15. Data Generation Dependency Order

The synthetic data generator should respect logical dependencies.

Recommended generation order:

```text
1. FACILITY
      ↓
2. CLINIC
      ↓
3. SPECIALTY
      ↓
4. SERVICE_TYPE
      ↓
5. ROOM
      ↓
6. DOCTOR
      ↓
7. DOCTOR_SCHEDULE
      ↓
8. PATIENT
      ↓
9. PAYER
      ↓
10. NFZ_CONTRACT
      ↓
11. NFZ_CONTRACT_LIMIT
      ↓
12. PATIENT_DEMAND
      ↓
13. REFERRAL
      ↓
14. REFERRAL_OUTCOME
      ↓
15. CLINIC_CAPACITY
      ↓
16. WAITING_LIST_ENTRY
      ↓
17. APPOINTMENT
      ↓
18. VISIT
      ↓
19. SERVICE_DELIVERY
      ↓
20. REVENUE_TRANSACTION
      ↓
21. COST_CATEGORY
      ↓
22. OPERATING_COST
      ↓
23. SCENARIO
      ↓
24. SCENARIO_PARAMETER
      ↓
25. SCENARIO_RESULT
```

The exact dependency order may be adjusted during ERD design.

---

# 16. Entity Grain

The following grain definitions should be preserved.

| Entity              | Grain                                               |
| ------------------- | --------------------------------------------------- |
| FACILITY            | One row per facility                                |
| CLINIC              | One row per clinic                                  |
| SPECIALTY           | One row per specialty                               |
| DOCTOR              | One row per doctor                                  |
| ROOM                | One row per room                                    |
| SERVICE_TYPE        | One row per service type                            |
| PATIENT             | One row per patient                                 |
| PATIENT_DEMAND      | One row per demand event                            |
| REFERRAL            | One row per referral                                |
| REFERRAL_OUTCOME    | One row per referral outcome                        |
| DOCTOR_SCHEDULE     | One row per doctor/date schedule                    |
| CLINIC_CAPACITY     | One row per clinic/service/date capacity definition |
| WAITING_LIST_ENTRY  | One row per patient waiting-list episode            |
| APPOINTMENT         | One row per appointment                             |
| VISIT               | One row per completed healthcare interaction        |
| SERVICE_DELIVERY    | One row per service delivered within a visit        |
| PAYER               | One row per payer                                   |
| NFZ_CONTRACT        | One row per contract                                |
| NFZ_CONTRACT_LIMIT  | One row per contract/service/period limit           |
| REVENUE_TRANSACTION | One row per revenue event                           |
| COST_CATEGORY       | One row per cost category                           |
| OPERATING_COST      | One row per cost event                              |
| SCENARIO            | One row per scenario                                |
| SCENARIO_PARAMETER  | One row per scenario parameter                      |
| SCENARIO_RESULT     | One row per scenario result                         |

---

# 17. Data Quality Rules

The generated dataset must maintain logical consistency.

## Referential Integrity

Every foreign key must reference an existing parent entity.

## Temporal Integrity

Examples:

```text
appointment_date >= booking_date

visit_date >= appointment_date

waiting_list_exit_date >= waiting_list_entry_date

contract_end_date >= contract_start_date
```

## Capacity Integrity

Operational activity should not systematically exceed available capacity unless the business rules explicitly allow overload or overbooking.

## Referral Integrity

Referral outcomes must be consistent with referral status.

## Appointment Integrity

Cancelled and no-show appointments should not generate completed visits.

## Financial Integrity

Revenue and cost values must be non-negative.

## Scenario Integrity

Scenario results must be generated from the parameters defined for the corresponding scenario.

---

# 18. Open Design Decisions

The following decisions should be finalized during the ERD phase.

### 1. Doctor-to-Specialty Relationship

Should one doctor have:

* exactly one specialty,
* one primary specialty plus multiple secondary specialties,
* multiple specialties?

### 2. Doctor-to-Clinic Relationship

Should doctors be able to work in multiple clinics?

If yes, a bridge entity such as:

```text
DOCTOR_CLINIC
```

may be required.

### 3. Room-to-Clinic Relationship

Should rooms belong exclusively to one clinic, or can they be shared?

### 4. Capacity Grain

Should capacity be modeled at:

```text
Clinic + Specialty + Date
```

or:

```text
Clinic + Specialty + Service + Doctor + Date
```

The second option provides more detailed utilization analysis.

### 5. Appointment and Visit

Should every visit require an appointment?

If walk-in visits are supported, `appointment_id` should be nullable.

### 6. Referral and Waiting List

Should every referral automatically create a waiting-list entry?

This depends on the referral workflow.

### 7. Revenue Grain

Should revenue be recorded:

```text
per visit
```

or:

```text
per service delivered
```

The preferred approach for detailed BI analysis is service-level revenue.

### 8. Cost Allocation

Should costs be recorded directly at clinic level or allocated across:

```text
Facility
Clinic
Specialty
Doctor
Service
```

### 9. Scenario Storage

Should scenario results be physically stored or calculated dynamically?

### 10. Analytical Layer

The final architecture should determine whether KPI entities are:

* physical database tables,
* SQL views,
* materialized views,
* Tableau calculations.

---

# 19. Next Step

The next document should be:

```text
05_Process_Flows.md
```

The process-flow documentation should define the lifecycle of the major business processes:

```text
Patient Demand
      ↓
Referral
      ↓
Waiting List
      ↓
Capacity Allocation
      ↓
Appointment Scheduling
      ↓
Visit
      ↓
Service Delivery
      ↓
Revenue Recognition
```

Additional processes should cover:

```text
NFZ Contract Management
Commercial Service Delivery
Capacity Planning
Referral Leakage
Operating Cost Generation
Strategic Scenario Simulation
```

After the process flows are validated, the project should proceed to:

```text
06_ERD.md
```

The ERD will validate:

* primary keys,
* foreign keys,
* cardinalities,
* bridge tables,
* fact and dimension boundaries,
* operational vs. analytical entities.

Only after the ERD is approved should the project proceed to:

```text
07_Generator_Architecture.md
```

followed by implementation of the Python synthetic data generator.
