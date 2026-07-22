# 05 — Process Flows

**Version:** 1.1
**Status:** Draft — Ready for Final Review
**Last Updated:** 2026-07-22

---

## 1. Purpose

This document defines the operational process flows of the **Healthcare Operations BI Simulator**.

The purpose of this document is to translate the business rules defined in:

* `01_Project_Overview.md`
* `02_Business_Model.md`
* `03_Business_Rules.md`
* `04_Entity_Dictionary.md`

into explicit end-to-end business processes.

The Process Flows document defines:

* how demand enters the healthcare system,
* how patients move through the care pathway,
* how referrals are created,
* how specialty availability is evaluated,
* how operational capacity is calculated,
* how NFZ-funded capacity affects effective capacity,
* how waiting time is estimated,
* how patient decisions affect internal capture and leakage,
* how waiting lists evolve,
* how appointments are scheduled,
* how services are delivered,
* how revenue and compensation are generated,
* how operating costs affect financial results,
* how NFZ contracts are monitored,
* how resource reallocations affect future capacity,
* how strategic scenarios are evaluated.

This document does not define the physical database schema or Python implementation.

The relationship between the project layers is:

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
ERD / Logical Data Model
       ↓
Generator Architecture
       ↓
Python Data Generator
       ↓
SQL
       ↓
Tableau
```

---

# 2. Process Flow Principles

The simulator follows several core principles.

## 2.1 Demand is not equal to delivered service

A patient demand event does not automatically result in a completed appointment.

The process may result in:

```text
Demand
   ↓
Care Pathway
   ↓
Referral
   ↓
Waiting
   ↓
Appointment
   ↓
Service Delivery
```

or:

```text
Demand
   ↓
Care Pathway
   ↓
Referral
   ↓
External Provider
```

or:

```text
Demand
   ↓
Care Pathway
   ↓
Referral
   ↓
Patient Drop
```

Therefore:

```text
Demand Volume
≠
Referral Volume
≠
Appointment Volume
≠
Delivered Service Volume
```

---

## 2.2 Capacity is constrained

The simulator must respect:

* doctor availability,
* doctor working hours,
* doctor schedules,
* specialty assignment,
* room availability,
* appointment duration,
* operational capacity,
* NFZ-funded capacity.

The simulator must never generate appointments that violate defined capacity constraints.

---

## 2.3 Patient behavior affects operational outcomes

Patients may:

* accept internal waiting,
* choose external care,
* abandon the process.

Therefore patient behavior influences:

* waiting list size,
* internal capture rate,
* referral leakage,
* delivered services,
* revenue.

---

## 2.4 Operational and financial processes are connected

A completed service affects:

```text
Service Delivery
       ↓
Revenue
       ↓
Doctor Compensation
       ↓
Clinic Revenue
       ↓
Operating Costs
       ↓
Net Clinic Result
```

---

# 3. High-Level System Process

The overall simulator process is:

```text
1. Generate Organization Structure
        ↓
2. Generate Patients and Demand
        ↓
3. Determine Care Pathway
        ↓
4. Create Referrals Where Required
        ↓
5. Check Specialty Availability
        ↓
6. Calculate Operational Capacity
        ↓
7. Calculate NFZ Funded Capacity
        ↓
8. Calculate Effective Capacity
        ↓
9. Estimate Waiting Time
        ↓
10. Calculate Patient Leakage Probability
        ↓
11. Simulate Patient Decision
        ↓
12. Create / Update Waiting List
        ↓
13. Schedule Appointment
        ↓
14. Deliver Service
        ↓
15. Record Revenue
        ↓
16. Calculate Doctor Compensation
        ↓
17. Record Operating Costs
        ↓
18. Calculate Financial Results
        ↓
19. Calculate Operational KPIs
        ↓
20. Perform Periodic / Quarterly Review
        ↓
21. Evaluate Scenarios and Management Actions
```

---

# 4. Process Flow 1 — Patient Demand Generation

## 4.1 Purpose

The demand generation process creates simulated healthcare demand.

Demand represents patients who require or seek healthcare services.

Demand is the starting point of the healthcare journey but does not automatically create a specialist referral.

---

## 4.2 Demand Sources

Demand may enter through different care pathways, including:

* primary care / POZ,
* pediatrics,
* specialist demand,
* follow-up demand,
* diagnostic demand,
* other configured demand sources.

The exact demand mix is determined by simulation parameters.

---

## 4.3 Inputs

Demand generation may depend on:

* patient population,
* clinic,
* specialty,
* age group,
* seasonality,
* historical demand,
* referral patterns,
* random variation,
* scenario assumptions.

---

## 4.4 Process

```text
Simulation Date
       ↓
Select Patient / Population Segment
       ↓
Select Demand Source
       ↓
Select Clinic
       ↓
Select Specialty or Service
       ↓
Generate Demand Event
       ↓
Create DEMAND_EVENT
```

---

## 4.5 Output

The process creates:

```text
DEMAND_EVENT
```

A demand event may:

```text
Demand
   ├── Require No Referral
   │       ↓
   │   Direct Care Pathway
   │
   └── Require Referral
           ↓
       Referral Process
```

---

# 5. Process Flow 2 — Care Pathway Determination

## 5.1 Purpose

This process determines what happens after a demand event is generated.

The simulator distinguishes between demand that can be handled directly and demand that requires referral.

---

## 5.2 Process

```text
DEMAND_EVENT
       ↓
Determine Care Pathway
       ├── Direct Care
       │      ↓
       │  Service / Appointment Process
       │
       └── Referral Required
              ↓
          REFERRAL CREATION
```

The care pathway may depend on:

* demand source,
* service type,
* specialty,
* patient characteristics,
* configured business rules.

---

# 6. Process Flow 3 — Referral Creation

## 6.1 Purpose

The referral process converts eligible demand into a structured referral.

---

## 6.2 Inputs

Required inputs include:

* patient,
* referring doctor,
* referring clinic,
* target specialty,
* referral date.

---

## 6.3 Process

```text
DEMAND_EVENT
       ↓
Referral Required?
       ↓
YES
       ↓
Create REFERRAL
       ↓
Assign Target Specialty
       ↓
Assign Referring Doctor
       ↓
Assign Referring Clinic
       ↓
Assign Referral Date
```

---

## 6.4 Output

```text
REFERRAL
```

The referral enters the internal routing process.

---

# 7. Process Flow 4 — Specialty Availability Check

## 7.1 Purpose

The simulator determines whether the requested specialty is available internally.

---

## 7.2 Process

```text
REFERRAL
       ↓
Identify Target Specialty
       ↓
Check CLINIC_SPECIALTY
       ↓
Is Specialty Available Internally?
       ├── NO
       │    ↓
       │  EXTERNAL ROUTING
       │
       └── YES
            ↓
        Continue to Capacity Check
```

---

## 7.3 Internal Specialty Available

If the specialty is available internally:

```text
REFERRAL
       ↓
Operational Capacity Check
```

---

## 7.4 Internal Specialty Not Available

If the specialty is not available internally:

```text
REFERRAL
       ↓
EXTERNAL
       ↓
REFERRAL_OUTCOME = EXTERNAL
```

The referral does not consume internal appointment capacity.

---

# 8. Process Flow 5 — Operational Capacity Calculation

## 8.1 Purpose

The simulator calculates the operational capacity available to deliver healthcare services.

---

## 8.2 Capacity Sources

Operational capacity is derived from:

```text
DOCTOR
    +
DOCTOR_SCHEDULE
    +
ROOM
    +
SERVICE_TYPE
    +
APPOINTMENT_DURATION
```

The calculation considers:

* eligible doctors,
* doctor availability,
* doctor working hours,
* doctor schedule,
* specialty qualification,
* room availability,
* existing appointments,
* appointment duration.

---

## 8.3 Process

```text
REFERRAL / DEMAND
       ↓
Identify Specialty
       ↓
Identify Required Service Type
       ↓
Identify Eligible Doctors
       ↓
Check DOCTOR_SCHEDULE
       ↓
Check ROOM Availability
       ↓
Check Existing Appointments
       ↓
Calculate Available Doctor Time
       ↓
Calculate Available Room Time
       ↓
Calculate Operational Capacity
```

---

## 8.4 Capacity Constraints

The simulator must enforce:

```text
Doctor Available Time
       ≥
Required Appointment Duration
```

and:

```text
Room Available Time
       ≥
Required Appointment Duration
```

The same doctor cannot have overlapping appointments.

The same room cannot host overlapping appointments.

---

# 9. Process Flow 6 — NFZ Funded Capacity

## 9.1 Purpose

NFZ contracts define the volume of services that can be funded under the applicable contract.

---

## 9.2 Process

```text
NFZ_CONTRACT
       ↓
Identify Clinic
       ↓
Identify Specialty
       ↓
Identify Service Type
       ↓
Identify Contract Period
       ↓
Read Contracted Volume
       ↓
Calculate Remaining Funded Capacity
```

The process is evaluated over the relevant contract period.

---

# 10. Process Flow 7 — Effective Capacity

## 10.1 Purpose

Effective capacity represents the actual capacity available after considering both operational constraints and NFZ-funded constraints.

---

## 10.2 Calculation

The core rule is:

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

Conceptually:

```text
              Operational Capacity
                       │
                       │
                       ├─────────────┐
                       │             │
                       ↓             │
                ┌──────────────┐     │
                │              │     │
                │     MIN      │ ←───┤
                │              │     │
                └──────┬───────┘     │
                       │             │
                       ↓             ↓
               EFFECTIVE CAPACITY
```

Therefore:

```text
Operational Capacity = 80
NFZ Funded Capacity  = 60
Effective Capacity   = 60
```

and:

```text
Operational Capacity = 50
NFZ Funded Capacity  = 80
Effective Capacity   = 50
```

The effective capacity is therefore constrained by the lower of:

* available operational capacity,
* available NFZ-funded capacity.

---

# 11. Process Flow 8 — Waiting Time Estimation

## 11.1 Purpose

The simulator estimates the expected waiting time before the patient can receive internal care.

---

## 11.2 Inputs

Waiting time depends on:

* current waiting list,
* available future effective capacity,
* specialty,
* service type,
* appointment duration,
* doctor schedules,
* room availability,
* existing appointments.

---

## 11.3 Process

```text
REFERRAL
       ↓
Identify Service Requirement
       ↓
Read Current Waiting List
       ↓
Read Future Effective Capacity
       ↓
Find Earliest Available Slot
       ↓
Estimate Waiting Time
```

The output is:

```text
Estimated Waiting Time
```

This value is passed to the patient decision process.

---

# 12. Process Flow 9 — Waiting Time and Leakage Probability

## 12.1 Purpose

Waiting time influences the probability that a patient will seek external care or abandon the process.

The leakage parameters are configurable simulation parameters.

---

## 12.2 Process

```text
Estimated Waiting Time
       ↓
Identify Waiting Time Band
       ↓
Read Leakage Probability Parameter
       ↓
Calculate Patient Outcome Probability
       ↓
Patient Decision
```

The conceptual relationship is:

```text
Waiting Time
       ↓
Leakage Probability
       ↓
Patient Decision
```

The business rules define configurable waiting-time bands and associated leakage probabilities.

The values are simulation parameters and may be adjusted during calibration.

---

# 13. Process Flow 10 — Patient Decision

## 13.1 Purpose

The patient decision process determines the outcome after referral routing and waiting-time estimation.

---

## 13.2 Possible Decisions

The patient may:

```text
WAIT
EXTERNAL
DROP
```

---

## 13.3 Process

```text
REFERRAL
       ↓
Waiting Time Estimate
       ↓
Leakage Probability
       ↓
Patient Decision
       ├── WAIT
       │    ↓
       │  WAITING_LIST_ENTRY
       │
       ├── EXTERNAL
       │    ↓
       │  REFERRAL_OUTCOME = EXTERNAL
       │
       └── DROP
            ↓
          REFERRAL_OUTCOME = NOT_COMPLETED
```

---

# 14. Process Flow 11 — Waiting List Management

## 14.1 Purpose

The waiting list tracks patients who have chosen to wait for internal care.

---

## 14.2 Entry

```text
PATIENT_DECISION = WAIT
       ↓
Create WAITING_LIST_ENTRY
```

---

## 14.3 Queue Evolution

The waiting list evolves over time:

```text
Previous Queue
       +
New Waiting Demand
       -
Completed Services
       -
External Leakage
       -
Dropped Patients
       =
Current Queue
```

---

## 14.4 Queue Processing

At each simulation period:

```text
WAITING_LIST_ENTRY
       ↓
Check Available Effective Capacity
       ↓
Select Eligible Patient
       ↓
Assign Appointment
       ↓
Close Waiting List Entry
```

The queue may be processed using:

* FIFO,
* priority rules,
* urgency rules,
* specialty-specific rules.

The selected queue discipline must be defined in generator configuration.

---

# 15. Process Flow 12 — Appointment Scheduling

## 15.1 Purpose

The appointment scheduling process converts an eligible waiting-list entry or direct-care demand event into a scheduled appointment.

---

## 15.2 Process

```text
WAITING_LIST_ENTRY / DIRECT DEMAND
       ↓
Identify Specialty
       ↓
Identify Service Type
       ↓
Identify Eligible Doctor
       ↓
Check DOCTOR_SCHEDULE
       ↓
Check ROOM
       ↓
Find Available Time Slot
       ↓
Create APPOINTMENT
```

---

## 15.3 Scheduling Constraints

An appointment is valid only if:

```text
Appointment Start
    ≥
Doctor Available Start
```

and:

```text
Appointment End
    ≤
Doctor Available End
```

and:

```text
Appointment Start
    ≥
Room Available Start
```

and:

```text
Appointment End
    ≤
Room Available End
```

Additionally:

```text
Same Doctor
+
Overlapping Appointments
=
INVALID
```

and:

```text
Same Room
+
Overlapping Appointments
=
INVALID
```

---

## 15.4 Appointment Duration

Appointment duration is determined by the selected service type.

```text
SERVICE_TYPE
       ↓
Standard Duration
       ↓
APPOINTMENT
       ↓
Reserved Doctor Time
       +
Reserved Room Time
```

---

# 16. Process Flow 13 — Service Delivery

## 16.1 Purpose

The service delivery process records whether a scheduled appointment was completed.

---

## 16.2 Process

```text
APPOINTMENT
       ↓
Appointment Date Reached
       ↓
Service Completed?
       ├── YES
       │    ↓
       │  SERVICE_DELIVERY
       │
       └── NO
            ↓
          Appointment Not Completed
```

---

## 16.3 Service Completion

A completed service generates:

```text
SERVICE_DELIVERY
       ↓
REVENUE_TRANSACTION
       ↓
DOCTOR_COMPENSATION_TRANSACTION
       ↓
CLINIC REVENUE
```

---

# 17. Process Flow 14 — First Visit and Follow-Up

The simulator distinguishes between:

```text
FIRST_VISIT
FOLLOW_UP
```

These service types may have different:

* appointment duration,
* revenue,
* NFZ-funded capacity,
* compensation,
* clinic revenue,
* operational capacity consumption.

---

## 17.1 First Visit

```text
Referral / New Demand
       ↓
FIRST_VISIT
       ↓
Appointment
       ↓
Service Delivery
```

---

## 17.2 Follow-Up

```text
Existing Patient
       ↓
FOLLOW_UP
       ↓
Appointment
       ↓
Service Delivery
```

The generator must preserve the distinction between first visits and follow-ups.

---

# 18. Process Flow 15 — Referral Outcome

The referral lifecycle ends with one of the following outcomes:

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

---

## 18.1 Internal Outcome

```text
REFERRAL
       ↓
WAIT
       ↓
WAITING LIST
       ↓
APPOINTMENT
       ↓
SERVICE_DELIVERY
       ↓
REFERRAL_OUTCOME = INTERNAL
```

---

## 18.2 External Outcome

```text
REFERRAL
       ↓
EXTERNAL
       ↓
REFERRAL_OUTCOME = EXTERNAL
```

---

## 18.3 Not Completed

```text
REFERRAL
       ↓
DROP
       ↓
REFERRAL_OUTCOME = NOT_COMPLETED
```

---

# 19. Process Flow 16 — Referral Leakage

Referral leakage represents demand that could potentially have been captured internally but was instead completed externally.

The primary flow is:

```text
Referral
       ↓
Internal Specialty Available
       ↓
Effective Capacity / Waiting Time
       ↓
Leakage Probability
       ↓
Patient Decision
       ↓
EXTERNAL
       ↓
Referral Leakage
```

The primary analytical metric is:

```text
Internal Capture Rate
=
Internal Referrals
/
Total Referrals
```

The complementary leakage measure is:

```text
Referral Leakage Rate
=
External Referrals
/
Total Referrals
```

The exact KPI definitions must be applied consistently across the reporting layer.

---

# 20. Process Flow 17 — Capacity Consumption

Every completed appointment consumes operational capacity.

```text
APPOINTMENT
       ↓
Doctor Time Consumed
       +
Room Time Consumed
       +
Service Capacity Consumed
       ↓
Update Capacity Utilization
```

The simulator tracks:

```text
Operational Capacity
NFZ Funded Capacity
Effective Capacity
Actual Delivered Services
```

The conceptual relationship is:

```text
Operational Capacity
        │
        ├──────────────┐
        │              │
        ↓              ↓
Operational       NFZ Funded
Capacity          Capacity
        │              │
        └──────┬───────┘
               ↓
              MIN
               ↓
       Effective Capacity
               ↓
       Delivered Services
```

---

# 21. Process Flow 18 — NFZ Contract Utilization

## 21.1 Purpose

NFZ contracts define funded service volumes and are monitored over the applicable contract period.

---

## 21.2 Process

```text
NFZ_CONTRACT
       ↓
Contracted Volume
       ↓
Track Funded Delivered Services
       ↓
Calculate Remaining Capacity
       ↓
Calculate Contract Utilization
       ↓
Evaluate Overperformance / Underperformance
```

---

## 21.3 Contract Utilization

Conceptually:

```text
Contract Utilization
=
Delivered Funded Services
/
Contracted Volume
```

---

## 21.4 Remaining Contract Capacity

```text
Remaining Contract Capacity
=
Contracted Volume
-
Delivered Funded Services
```

Where applicable, remaining capacity cannot be negative for the purpose of calculating unused contracted volume.

Overperformance is tracked separately.

---

# 22. Process Flow 19 — Quarterly NFZ Settlement

## 22.1 Purpose

The quarterly settlement process evaluates NFZ contract performance at the end of the relevant reporting period.

---

## 22.2 Process

```text
Quarter End
       ↓
Collect Funded Service Delivery
       ↓
Aggregate by:
    Clinic
    Specialty
    Service Type
       ↓
Compare Delivered Volume
with Contracted Volume
       ↓
Calculate:
    Contract Utilization
    Remaining Capacity
    Underperformance
    Overperformance
       ↓
Generate Management Indicators
```

---

## 22.3 Quarterly Management Review

The quarterly settlement may trigger management actions such as:

```text
Underutilization
       ↓
Investigate Demand / Capacity / Routing
```

or:

```text
Overperformance
       ↓
Evaluate Contract Amendment
```

or:

```text
Persistent Capacity Constraint
       ↓
Evaluate Resource Reallocation
```

The settlement itself does not retroactively modify historical service delivery.

---

# 23. Process Flow 20 — NFZ Contract Amendment

Contract amendments may change future funded capacity.

The causal chain is:

```text
Existing NFZ Contract
       ↓
Contract Amendment
       ↓
Updated NFZ Funded Capacity
       ↓
Updated Effective Capacity
       ↓
Waiting Time
       ↓
Patient Leakage
       ↓
Delivered Services
       ↓
Revenue
```

The amendment applies from its effective date.

Historical periods remain unchanged.

---

# 24. Process Flow 21 — Revenue Generation

A completed service generates revenue.

```text
SERVICE_DELIVERY
       ↓
Identify Service Type
       ↓
Identify Revenue Rule
       ↓
Calculate Service Revenue
       ↓
Create REVENUE_TRANSACTION
```

Revenue may depend on:

* service type,
* specialty,
* clinic,
* payer,
* NFZ contract,
* pricing rules.

---

# 25. Process Flow 22 — Doctor Compensation

Doctor compensation is calculated from completed services and applicable compensation rules.

---

## 25.1 Percentage Model

```text
Service Revenue
       ↓
Doctor Compensation Rule
       ↓
Percentage × Revenue
       ↓
Doctor Compensation
```

---

## 25.2 Hourly Model

```text
Doctor Time
       ↓
Hourly Rate
       ↓
Doctor Compensation
```

---

## 25.3 Mixed Model

The mixed model combines multiple compensation components according to the configured business rule.

```text
Service Delivery
       ↓
Compensation Rule
       ↓
Calculate Compensation Components
       ↓
Doctor Compensation Transaction
```

The exact mathematical formula for `MIXED` compensation is configuration-dependent.

---

# 26. Process Flow 23 — Clinic Revenue

Clinic revenue is calculated after doctor compensation.

```text
Total Service Revenue
       -
Doctor Compensation
       =
Clinic Revenue
```

The result may be aggregated by:

* clinic,
* specialty,
* doctor,
* service type,
* period.

---

# 27. Process Flow 24 — Operating Costs

## 27.1 Purpose

Operating costs represent the costs required to operate clinics and deliver services.

---

## 27.2 Cost Behavior

Costs are classified as:

```text
FIXED
VARIABLE
SEMI_VARIABLE
```

---

## 27.3 Process

```text
Operating Activity
       ↓
Identify Cost Behavior
       ├── FIXED
       ├── VARIABLE
       └── SEMI_VARIABLE
       ↓
Calculate / Record Cost
       ↓
OPERATING_COST_TRANSACTION
```

---

## 27.4 Cost Behavior Logic

### Fixed Costs

Fixed costs are primarily independent of short-term service volume.

```text
Fixed Cost
       ↓
Period / Clinic
       ↓
OPERATING_COST_TRANSACTION
```

### Variable Costs

Variable costs change with activity or service volume.

```text
Service Volume
       ↓
Variable Cost Rule
       ↓
Variable Cost
```

### Semi-Variable Costs

Semi-variable costs contain both fixed and variable components.

```text
Base Cost
       +
Activity-Dependent Component
       ↓
Semi-Variable Cost
```

The exact cost formulas are configuration-dependent.

---

# 28. Process Flow 25 — Net Clinic Result

The financial result is calculated as:

```text
Clinic Revenue
       -
Operating Costs
       =
Net Clinic Result
```

At a higher level:

```text
Service Revenue
       -
Doctor Compensation
       -
Operating Costs
       =
Net Clinic Result
```

This metric is used to evaluate operational and strategic performance.

---

# 29. Process Flow 26 — Resource Reallocation

Resource reallocation allows the simulator to evaluate changes in resource distribution between clinics.

Resources may include:

* doctors,
* doctor hours,
* rooms,
* operational capacity.

---

## 29.1 Process

```text
Current State
       ↓
Define Resource Reallocation
       ↓
RESOURCE_REALLOCATION
       ↓
Apply Effective Date
       ↓
Update Future Resource Availability
       ↓
Recalculate Operational Capacity
       ↓
Recalculate Effective Capacity
       ↓
Recalculate Waiting Time
       ↓
Recalculate Referral Outcomes
       ↓
Recalculate Financial Impact
```

Historical periods before the effective date remain unchanged.

---

## 29.2 Before vs After Comparison

The simulator must support:

```text
Current State
        VS
Post-Reallocation State
```

Possible comparison metrics include:

* waiting time,
* waiting list size,
* internal capture rate,
* referral leakage,
* delivered services,
* capacity utilization,
* revenue,
* doctor compensation,
* operating costs,
* net clinic result.

---

# 30. Process Flow 27 — Strategic Specialty Expansion

The simulator can evaluate whether a clinic should introduce a new specialty.

---

## 30.1 Current State

```text
External Referrals
       ↓
Potential Internal Demand
       ↓
Estimated Capture Opportunity
```

---

## 30.2 Expansion Scenario

```text
New Specialty Scenario
       ↓
Assign Doctors
       ↓
Assign Doctor Capacity
       ↓
Assign Rooms
       ↓
Estimate Operational Capacity
       ↓
Estimate NFZ Funded Capacity
       ↓
Calculate Effective Capacity
       ↓
Estimate Internal Capture
       ↓
Estimate Waiting Time
       ↓
Estimate Revenue
       ↓
Estimate Costs
       ↓
Estimate Net Financial Impact
```

---

## 30.3 Decision Metrics

The scenario may evaluate:

* potential demand,
* external referral volume,
* expected internal capture,
* expected waiting time,
* required doctors,
* required rooms,
* required capacity,
* expected revenue,
* doctor compensation,
* operating costs,
* net clinic result.

---

# 31. Process Flow 28 — Scenario Analysis

The simulator supports what-if analysis.

A scenario defines a change to the current operating model.

Examples include:

```text
Scenario A
Add Doctor
```

```text
Scenario B
Add Room
```

```text
Scenario C
Reallocate Doctor Hours
```

```text
Scenario D
Add New Specialty
```

```text
Scenario E
Increase NFZ Funded Capacity
```

---

## 31.1 Scenario Process

```text
Current State
       ↓
Create SCENARIO
       ↓
Define Scenario Changes
       ↓
Apply Changes
       ↓
Recalculate Future Operational Capacity
       ↓
Recalculate Future Effective Capacity
       ↓
Recalculate Operational Outcomes
       ↓
Recalculate Financial Outcomes
       ↓
Compare with Baseline
```

---

## 31.2 Scenario Comparison

The simulator compares:

```text
BASELINE
    VS
SCENARIO
```

Metrics may include:

```text
Demand
Waiting List
Waiting Time
Delivered Services
Internal Capture Rate
Referral Leakage
Operational Capacity
NFZ Funded Capacity
Effective Capacity
Capacity Utilization
Revenue
Doctor Compensation
Operating Costs
Net Clinic Result
```

---

# 32. Process Flow 29 — Daily Operational Cycle

The simulator operates through repeated daily or short-period operational cycles.

The daily cycle is:

```text
START DAY
    ↓
Generate Demand
    ↓
Determine Care Pathway
    ↓
Create Referrals Where Required
    ↓
Evaluate Specialty Availability
    ↓
Calculate Operational Capacity
    ↓
Calculate NFZ Funded Capacity
    ↓
Calculate Effective Capacity
    ↓
Estimate Waiting Time
    ↓
Calculate Leakage Probability
    ↓
Simulate Patient Decisions
    ↓
Update Waiting Lists
    ↓
Schedule Appointments
    ↓
Deliver Scheduled Services
    ↓
Record Revenue
    ↓
Calculate Doctor Compensation
    ↓
Record Operating Costs
    ↓
Update Operational KPIs
    ↓
END DAY
```

This process repeats throughout the simulation horizon.

---

# 33. Process Flow 30 — Periodic / Quarterly Management Cycle

At the end of each reporting period, the simulator performs aggregation and management analysis.

```text
END REPORTING PERIOD
       ↓
Aggregate Operational Data
       ↓
Aggregate Financial Data
       ↓
Calculate KPI Results
       ↓
Evaluate NFZ Contract Performance
       ↓
Evaluate Capacity Constraints
       ↓
Evaluate Waiting List Trends
       ↓
Evaluate Referral Leakage
       ↓
Evaluate Financial Performance
       ↓
Management Review
```

At quarter-end:

```text
Quarter End
       ↓
NFZ Settlement
       ↓
Contract Utilization
       ↓
Underperformance / Overperformance
       ↓
Management Decision
```

Possible management actions include:

```text
No Action
```

```text
Resource Reallocation
```

```text
NFZ Contract Amendment
```

```text
Specialty Expansion
```

```text
Capacity Investment
```

These actions affect future simulation periods only.

---

# 34. Process Flow 31 — Data Lineage

The main demand and referral lineage is:

```text
PATIENT / POPULATION
       ↓
DEMAND_EVENT
       ↓
CARE PATHWAY
       ↓
REFERRAL
       ↓
PATIENT_DECISION
       ↓
WAITING_LIST_ENTRY
       ↓
APPOINTMENT
       ↓
SERVICE_DELIVERY
       ↓
REVENUE_TRANSACTION
       ↓
DOCTOR_COMPENSATION_TRANSACTION
       ↓
CLINIC REVENUE
       ↓
OPERATING_COST_TRANSACTION
       ↓
NET CLINIC RESULT
```

Capacity lineage:

```text
DOCTOR
       +
DOCTOR_SCHEDULE
       +
ROOM
       +
SERVICE_TYPE
       ↓
OPERATIONAL CAPACITY
       +
NFZ_CONTRACT
       ↓
NFZ FUNDED CAPACITY
       ↓
MIN
       ↓
EFFECTIVE CAPACITY
       ↓
APPOINTMENT
       ↓
SERVICE DELIVERY
```

NFZ lineage:

```text
NFZ_CONTRACT
       ↓
CONTRACTED VOLUME
       ↓
FUNDED DELIVERED SERVICES
       ↓
CONTRACT UTILIZATION
       ↓
QUARTERLY SETTLEMENT
       ↓
MANAGEMENT ACTION
```

---

# 35. Key Process Controls

The simulator must enforce the following controls.

## 35.1 Doctor Availability

An appointment cannot be scheduled outside the doctor's available schedule.

---

## 35.2 Room Availability

A room cannot host overlapping appointments.

---

## 35.3 Doctor Overlap

A doctor cannot have overlapping appointments.

---

## 35.4 Specialty Eligibility

A doctor must be eligible to deliver the selected specialty and service.

---

## 35.5 Appointment Duration

The appointment must fit within the available doctor and room time window.

---

## 35.6 Historical Integrity

Scenario changes, resource reallocations and NFZ contract amendments affect future periods only.

Historical data must not be retroactively modified.

---

## 35.7 Capacity Integrity

The simulator must not generate delivered services beyond available effective capacity unless the business rules explicitly allow overperformance.

---

## 35.8 Referral Integrity

Every referral must ultimately resolve to a valid outcome:

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

---

## 35.9 Effective Capacity Integrity

Effective capacity must always follow:

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

---

# 36. KPI Process Flow

The primary KPI calculation flow is:

```text
Raw Operational Data
       ↓
Aggregate by Period
       ↓
Aggregate by Clinic
       ↓
Aggregate by Specialty
       ↓
Calculate Operational KPIs
       ↓
Calculate Referral KPIs
       ↓
Calculate Financial KPIs
       ↓
Calculate Strategic KPIs
```

---

## 36.1 Operational KPIs

Examples:

```text
Total Demand
Total Referrals
Delivered Services
Waiting List Size
Average Waiting Time
Operational Capacity
NFZ Funded Capacity
Effective Capacity
Capacity Utilization
Room Utilization
Doctor Utilization
```

---

## 36.2 Referral KPIs

Examples:

```text
Internal Capture Rate
Referral Leakage Rate
External Referral Volume
Not Completed Referral Volume
```

---

## 36.3 Financial KPIs

Examples:

```text
Total Revenue
Doctor Compensation
Clinic Revenue
Operating Costs
Net Clinic Result
Revenue per Doctor
Revenue per Room
```

---

## 36.4 Strategic KPIs

Examples:

```text
Potential Demand
Uncaptured Demand
Potential Revenue
Expected Capacity Requirement
Expected Waiting Time
Expected Internal Capture
Scenario Net Impact
```

---

# 37. Complete Process Flow Summary

The complete operational model can be summarized as:

```text
                    ┌─────────────────────┐
                    │       DEMAND        │
                    └──────────┬──────────┘
                               ↓
                    ┌─────────────────────┐
                    │  CARE PATHWAY       │
                    └──────────┬──────────┘
                               ↓
                    ┌─────────────────────┐
                    │      REFERRAL       │
                    └──────────┬──────────┘
                               ↓
                 ┌───────────────────────────┐
                 │ Specialty Available?     │
                 └────────────┬──────────────┘
                              │
                  ┌───────────┴───────────┐
                  │                       │
                 NO                      YES
                  │                       │
                  ↓                       ↓
             EXTERNAL              CAPACITY CHECK
                                          │
                                          ↓
                             OPERATIONAL CAPACITY
                                          │
                                          ↓
                               NFZ FUNDED CAPACITY
                                          │
                                          ↓
                                        MIN
                                          │
                                          ↓
                                 EFFECTIVE CAPACITY
                                          │
                                          ↓
                                  WAITING TIME
                                          │
                                          ↓
                               LEAKAGE PROBABILITY
                                          │
                                          ↓
                                  PATIENT DECISION
                                          │
                         ┌────────────────┼───────────────┐
                         │                │               │
                       WAIT            EXTERNAL          DROP
                         │                │               │
                         ↓                ↓               ↓
                  WAITING LIST       EXTERNAL       NOT COMPLETED
                         │
                         ↓
                    APPOINTMENT
                         │
                         ↓
                 SERVICE DELIVERY
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
          REVENUE              CAPACITY USE
              │                     │
              ↓                     ↓
       DOCTOR COMP.          OPERATIONAL KPIs
              │
              ↓
       CLINIC REVENUE
              │
              ↓
      OPERATING COSTS
              │
              ↓
     NET CLINIC RESULT
              │
              ↓
       BI / ANALYTICS
```

---

# 38. Management Feedback Loop

The simulator is not only a transactional model.

Operational results feed back into management decisions.

```text
Operational Performance
       ↓
KPI Analysis
       ↓
Quarterly Review
       ↓
Identify Bottlenecks
       ↓
Management Decision
       ├── Resource Reallocation
       ├── NFZ Contract Amendment
       ├── Capacity Investment
       ├── Specialty Expansion
       └── No Action
       ↓
Future Operating Model
       ↓
New Simulation Period
```

This creates the core management simulation loop:

```text
OPERATE
   ↓
MEASURE
   ↓
ANALYZE
   ↓
DECIDE
   ↓
CHANGE
   ↓
OPERATE AGAIN
```

---

# 39. Process Flow Summary by Time Horizon

The simulator operates at three logical time horizons.

## Daily / Short-Term

```text
Demand
→ Referral
→ Capacity
→ Waiting
→ Patient Decision
→ Waiting List
→ Appointment
→ Service
```

## Monthly / Reporting

```text
Operational Aggregation
→ KPI Calculation
→ Financial Aggregation
→ Performance Analysis
```

## Quarterly / Management

```text
NFZ Settlement
→ Contract Performance
→ Capacity Review
→ Financial Review
→ Management Decision
→ Resource / Contract / Specialty Change
```

---

# 40. Next Documentation Step

After approval of this Process Flows document, the next documentation step is:

```text
01 Project Overview
        ↓
02 Business Model
        ↓
03 Business Rules
        ↓
04 Entity Dictionary v1.4
        ↓
05 Process Flows v1.1
        ↓
06 ERD / Logical Data Model
        ↓
07 Generator Architecture
        ↓
Python Data Generator
        ↓
SQL Data Layer
        ↓
Tableau BI Layer
```

The ERD should validate that all entities and relationships required by these process flows are represented correctly.

The Python Generator Architecture should then translate these process flows into deterministic simulation logic.

---

# 41. Document Status

**Current Version:** 1.1

**Status:** Draft — Ready for Final Review

**Dependencies:**

```text
01_Project_Overview.md
02_Business_Model.md
03_Business_Rules.md
04_Entity_Dictionary.md
```

**Key Validation Rule:**

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

**Next Review:**

Final cross-audit against:

* `03_Business_Rules.md`
* `04_Entity_Dictionary.md`

The purpose of the final review is to confirm that every major business rule has a corresponding process flow and that every process flow is supported by the Entity Dictionary before proceeding to ERD design.
