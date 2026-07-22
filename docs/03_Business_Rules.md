# Business Rules

## 1. Purpose

This document defines the core business rules governing the Healthcare Operations BI Simulator.

The purpose of these rules is to ensure that the simulated healthcare organization behaves according to a consistent and realistic operational model. The rules will serve as the foundation for:

* the synthetic data generator,
* the relational data model,
* SQL transformations and analytical queries,
* Tableau dashboards,
* operational KPI calculations,
* financial analysis,
* and strategic "What-if?" scenarios.

The simulator is designed to model the relationship between patient demand, referrals, healthcare capacity, NFZ funding, waiting times, patient behavior, operational performance, revenue, costs, and strategic investment decisions.

The model should not generate independent random records without business logic. Generated data must be internally consistent and follow the rules defined in this document.

---

# 2. Healthcare Organization Structure

## 2.1 Organization Type

The simulated organization represents a multi-specialty outpatient healthcare provider operating in Poland.

The organization provides healthcare services through:

* Primary Care / POZ,
* Pediatric care,
* Specialist outpatient clinics.

The organization operates under a combination of:

* NFZ-funded services,
* commercial services,
* and internal operational resources.

---

## 2.2 Available Specialist Clinics

The initial organization operates a fixed set of 11 specialist clinics.

Each clinic is represented as an operational unit with its own:

* specialty,
* doctors,
* rooms,
* working schedules,
* service types,
* capacity,
* NFZ contract,
* demand,
* waiting list,
* waiting time,
* revenue,
* operating costs.

The model should also contain a broader catalog of approximately 20–25 medical specialties.

Each specialty must have an availability status:

```text
AVAILABLE = TRUE
AVAILABLE = FALSE
```

The 11 specialties currently operated by the organization are marked as available.

Other specialties remain in the catalog but are not currently provided internally.

This distinction is essential for modelling referral leakage.

---

# 3. Patient Demand

Patient demand represents the number of patients seeking healthcare services during a given period.

Demand may originate from:

* direct primary care visits,
* pediatric visits,
* specialist referrals,
* follow-up care,
* diagnostic requirements.

Demand should vary over time and may be influenced by:

* seasonality,
* specialty,
* patient population,
* historical referral patterns,
* operational availability,
* and simulated demand trends.

Demand is measured at different levels:

```text
Daily Demand
Weekly Demand
Monthly Demand
Quarterly Demand
Annual Demand
```

The simulator should preserve the relationship between demand and actual service delivery.

Demand does not automatically equal completed visits.

A patient may:

* receive care internally,
* remain in a waiting list,
* choose an external provider,
* or abandon the referral.

---

# 4. Primary Care and Referral Rules

## 4.1 Primary Care Visits

Patients may first enter the organization through:

```text
Patient
↓
POZ Visit
```

or:

```text
Patient
↓
Pediatric Visit
```

During a primary care visit, the physician may determine that specialist care is required.

The patient may then receive a referral.

---

## 4.2 Referral Generation

A referral must contain at least:

* patient,
* referring doctor,
* referring clinic,
* target specialty,
* referral date,
* referral status.

A referral may point to:

* an internally available specialty,
* an internally unavailable specialty.

The target specialty is determined by the patient's simulated medical need.

The referral itself does not guarantee that the patient will receive treatment internally.

---

# 5. Referral Destination and Patient Leakage

Each referral must ultimately be classified according to its destination.

The core referral destination categories are:

```text
INTERNAL
EXTERNAL
NOT_COMPLETED
```

### INTERNAL

The patient remains within the organization and completes the specialist care pathway.

### EXTERNAL

The patient leaves the organization and receives care from an external provider.

### NOT_COMPLETED

The patient does not complete the referral pathway.

This may represent:

* abandonment,
* failure to schedule,
* cancellation without rescheduling,
* or another form of non-completion.

---

## 5.1 Internal Capture Rate

The organization should measure its ability to retain referred patients.

```text
Internal Capture Rate
=
Internal Referrals
/
Total Referrals
```

---

## 5.2 External Leakage Rate

```text
External Leakage Rate
=
External Referrals
/
Total Referrals
```

---

## 5.3 Referral Non-Completion Rate

```text
Non-Completion Rate
=
Not Completed Referrals
/
Total Referrals
```

These metrics should be available by:

* specialty,
* referring department,
* doctor,
* clinic,
* month,
* quarter,
* year.

The model should allow management to identify which specialties generate the highest patient leakage.

---

# 6. Specialty Availability

When a patient receives a referral, the system must determine whether the organization provides the required specialty.

### Scenario A — Specialty Available

```text
Referral
↓
Specialty Available
↓
Check Capacity
↓
Calculate Waiting Time
↓
Patient Decision
```

The patient may:

* wait for an internal appointment,
* choose an external provider,
* abandon the referral.

### Scenario B — Specialty Not Available

```text
Referral
↓
Specialty Not Available
↓
External Provider
```

If the organization does not provide the required specialty, the referral is classified as external leakage.

This mechanism allows the simulator to identify potential strategic opportunities for opening new clinics.

---

# 7. Waiting List

When demand exceeds the effective capacity of a specialty, patients enter a waiting list.

The waiting list should be influenced by:

* new referrals,
* current queue size,
* number of doctors,
* doctor availability,
* room capacity,
* appointment duration,
* NFZ contract limits,
* first-visit capacity,
* completed visits,
* external leakage,
* patient drop-off.

The waiting list is dynamic and changes over time.

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

The model should support daily or weekly queue simulation.

---

# 8. Capacity Model

The model defines three separate capacity concepts.

## 8.1 Demand

Demand represents the number of patients who require a service.

```text
DEMAND
```

---

## 8.2 Operational Capacity

Operational capacity represents the maximum number of visits that can be physically delivered based on available resources.

It depends on:

* number of doctors,
* doctor working hours,
* doctor schedules,
* number of rooms,
* room availability,
* appointment duration.

Conceptually:

```text
Operational Capacity
=
Capacity of Doctors
limited by
Room Capacity
```

The effective operational capacity cannot exceed the number of appointments that can physically be delivered by the available doctors and rooms.

---

## 8.3 NFZ Funded Capacity

NFZ funded capacity represents the number of services that can be delivered within the applicable simulated NFZ contract limit.

This is a financial/contractual constraint rather than a physical capacity constraint.

---

## 8.4 Effective Capacity

The effective capacity is the lower of operational capacity and NFZ funded capacity.

```text
Effective Capacity
=
MIN(
    Operational Capacity,
    NFZ Funded Capacity
)
```

This rule creates two important bottleneck scenarios.

### Operational Bottleneck

```text
Operational Capacity < NFZ Capacity
```

The organization has insufficient:

* doctors,
* working hours,
* rooms,
* or other operational resources.

### Funding Bottleneck

```text
NFZ Capacity < Operational Capacity
```

The organization has sufficient physical capacity but cannot use it fully for NFZ-funded services because of the contract limit.

---

# 9. Doctor Capacity

Doctor capacity depends on:

* doctor availability,
* working hours,
* working days,
* specialty,
* appointment duration,
* room availability.

For example:

```text
2 Doctors
×
20 Hours / Week
×
2 Visits / Hour
=
80 Operational Visits / Week
```

If the NFZ contract allows only:

```text
60 First Visits / Week
```

then:

```text
Operational Capacity = 80
NFZ Capacity = 60
Effective Capacity = 60
```

The model must distinguish between:

```text
Operational Capacity
Funded Capacity
Effective Capacity
Actual Delivered Services
```

These values are not necessarily equal.

---

# 10. Room Capacity

A room can support only a limited number of simultaneous appointments.

A room cannot host two appointments at the same time.

The same physical room cannot be assigned to overlapping appointments.

Therefore:

```text
Room Capacity
```

may become a bottleneck even when sufficient doctors are available.

The simulator should prevent:

* overlapping appointments in the same room,
* overlapping appointments for the same doctor.

---

# 11. Waiting Time

Waiting time is a dynamic operational KPI.

It depends on:

```text
Demand
Doctor Availability
Room Capacity
NFZ Capacity
First Visit Capacity
Current Queue
```

Conceptually:

```text
Waiting Time
=
f(
    Demand,
    Doctor Capacity,
    Room Capacity,
    NFZ Capacity,
    Queue Size
)
```

Waiting time should increase when:

```text
Demand > Effective Capacity
```

and decrease when:

```text
Effective Capacity > Demand
```

The model should support the accumulation of waiting lists over time.

Example:

```text
Demand = 100 / week
Effective Capacity = 60 / week
```

The theoretical weekly shortage is:

```text
100 - 60 = 40 patients
```

Without additional capacity or patient leakage, the queue will grow over time.

---

# 12. Patient Behavior

Patients do not all respond to waiting times in the same way.

The simulator models three primary patient behaviors:

```text
WAIT
EXTERNAL
DROP
```

### WAIT

The patient remains in the internal system and waits for an appointment.

### EXTERNAL

The patient chooses an external provider.

### DROP

The patient abandons the referral process.

---

## 12.1 Waiting Time and Leakage

The probability of patient leakage increases as waiting time increases.

The following values are simulation parameters and should be configurable:

```text
0–7 days       → 5% leakage
8–30 days      → 10%
31–60 days     → 20%
61–90 days     → 35%
91–180 days    → 55%
180+ days      → 75%
```

These values are not intended to represent real-world clinical statistics.

They are configurable assumptions used to generate realistic synthetic behavior.

The model should allow these parameters to be changed without rewriting the core generator logic.

---

## 12.2 Patient Decision

For an available specialty:

```text
Referral
↓
Waiting Time
↓
Patient Decision
```

The decision is probabilistic and depends primarily on waiting time.

The outcome is:

```text
WAIT
EXTERNAL
DROP
```

For an unavailable specialty:

```text
Referral
↓
No Internal Clinic
↓
EXTERNAL
```

---

# 13. First Visits and Follow-up Visits

The model distinguishes between:

```text
FIRST_VISIT
FOLLOW_UP
```

These services have different operational and financial characteristics.

A first visit represents the initial specialist consultation.

A follow-up represents subsequent care related to an existing treatment pathway.

The model should allow different:

* appointment durations,
* revenue values,
* NFZ limits,
* doctor compensation,
* clinic revenue.

The distinction is critical because the mix between first visits and follow-ups affects both:

* capacity,
* and financial performance.

---

# 14. NFZ Contracts

Each relevant service line may have a simulated NFZ contract.

The contract defines the maximum funded volume for a given period.

The model should track:

```text
Contracted Volume
Delivered Volume
Remaining Contract Capacity
Contract Utilization
Overperformance
Underperformance
```

---

## 14.1 Contract Utilization

```text
Contract Utilization
=
Delivered Funded Services
/
Contracted Services
```

Contract utilization should be analyzed by:

* specialty,
* clinic,
* service type,
* quarter,
* year.

---

## 14.2 Underutilization

If:

```text
Delivered Services < Contracted Services
```

the contract is underutilized.

This may indicate:

* insufficient demand,
* insufficient referrals,
* operational inefficiency,
* staffing issues,
* scheduling problems.

---

## 14.3 Overdemand

If:

```text
Demand > Effective Capacity
```

the organization has more demand than it can effectively serve.

This may result in:

* longer waiting times,
* growing queues,
* external leakage,
* patient drop-off.

---

# 15. Quarterly NFZ Settlement

NFZ performance should be evaluated on a quarterly basis.

The model should support:

```text
Quarterly Contract
↓
Services Delivered
↓
Contract Utilization
↓
Underuse / Overdemand
↓
Management Decision
```

The organization may respond to performance by:

* adjusting capacity,
* reallocating resources,
* requesting contract amendments,
* increasing or decreasing operational capacity.

---

# 16. Resource Reallocation

The organization may reallocate resources between clinics.

Resources may include:

* doctors,
* doctor hours,
* rooms,
* operational capacity.

Resource reallocation may be triggered by:

* low utilization,
* high demand,
* long waiting times,
* high leakage,
* strategic priorities.

A reallocation decision should affect future capacity.

The model should preserve the distinction between:

```text
Current State
```

and:

```text
Post-Reallocation State
```

This enables before-and-after analysis.

---

# 17. Contract Amendments

The model should support simulated changes to NFZ contracts.

An amendment may:

* increase funded capacity,
* decrease funded capacity,
* change service volumes,
* change the effective capacity of a specialty.

A contract amendment should affect:

```text
NFZ Capacity
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

This creates a causal relationship between contract decisions and operational outcomes.

---

# 18. Revenue Model

The simulator separates:

```text
Total Service Revenue
Doctor Compensation
Clinic Revenue
Operating Costs
Net Clinic Result
```

Total service revenue represents the economic value of delivered healthcare services.

The revenue is allocated between:

```text
Doctor Share
Clinic Share
```

The exact split may vary by doctor or compensation model.

---

# 19. Doctor Compensation

Doctors may operate under different compensation models.

Supported models include:

```text
PERCENTAGE
HOURLY_RATE
MIXED
```

### Percentage Model

A percentage of service revenue is allocated to the doctor.

Example:

```text
Service Revenue = 500 PLN
Doctor Share = 55%
Clinic Share = 45%
```

Result:

```text
Doctor = 275 PLN
Clinic = 225 PLN
```

### Hourly Rate Model

The doctor is compensated based on worked hours.

### Mixed Model

The doctor's compensation consists of a combination of:

* fixed/hourly compensation,
* variable revenue-based compensation.

The compensation model should be stored at doctor level or contract level.

---

# 20. Service Revenue by Visit Type

First visits and follow-ups may have different service values.

Example simulation parameters:

```text
First Visit = 500 PLN
Follow-up = 200 PLN
Doctor Share = 55%
Clinic Share = 45%
```

For:

```text
10 First Visits
+
10 Follow-ups
```

Total revenue is:

```text
10 × 500
+
10 × 200
=
7,000 PLN
```

Doctor compensation:

```text
7,000 × 55%
=
3,850 PLN
```

Clinic share:

```text
7,000 × 45%
=
3,150 PLN
```

The actual values are configurable simulation assumptions.

---

# 21. Operating Costs

Clinic revenue must not be treated as profit.

The clinic share may be reduced by operating costs, including:

* nursing staff,
* registration staff,
* administration,
* cleaning,
* utilities,
* rent,
* IT,
* medical systems,
* insurance,
* equipment,
* depreciation,
* other operating expenses.

The simplified financial flow is:

```text
Total Service Revenue
        ↓
Doctor Compensation
        ↓
Clinic Revenue
        ↓
Operating Costs
        ↓
Net Clinic Result
```

The model should allow costs to be classified as:

```text
FIXED
VARIABLE
SEMI_VARIABLE
```

Where possible, costs should be attributable to:

* clinic,
* department,
* specialty,
* or organization-wide operations.

---

# 22. Financial KPIs

The model should support calculation of:

```text
Total Revenue
Doctor Compensation
Clinic Revenue
Operating Costs
Net Clinic Result
Margin %
Revenue per Patient
Revenue per Doctor
Revenue per Room
Cost per Visit
```

Additional KPIs may be introduced as the financial model develops.

---

# 23. Strategic Clinic Expansion

One of the primary objectives of the simulator is to evaluate whether the organization should open a new specialist clinic.

A candidate specialty may currently be unavailable internally.

The model should identify potential opportunities based on:

* number of referrals,
* external leakage,
* uncompleted referrals,
* potential internal demand,
* lost revenue,
* available doctors,
* available rooms,
* potential NFZ contract,
* expected capacity,
* expected waiting time,
* expected patient capture,
* operating costs.

The strategic question is:

> Which one new specialty would create the greatest strategic and economic value for the organization?

---

# 24. New Clinic Scenario

A new clinic scenario may introduce:

```text
New Specialty
+
New Doctors
+
New Rooms
+
New Equipment
+
Potential NFZ Contract
```

The simulator should compare:

```text
CURRENT STATE
vs.
NEW CLINIC SCENARIO
```

The scenario should estimate:

```text
Patients Captured
Revenue
Clinic Revenue
Contract Utilization
Patient Leakage
Waiting Time
Doctor Utilization
Room Utilization
Operating Cost
Net Impact
```

---

# 25. What-if Analysis

The model should support multiple strategic scenarios.

Example:

```text
Scenario A
No New Clinic
```

```text
Scenario B
Open Ophthalmology
1 Doctor
1 Room
```

```text
Scenario C
Open Ophthalmology
2 Doctors
2 Rooms
```

```text
Scenario D
Open ENT
```

Each scenario should be evaluated against a consistent set of KPIs.

The objective is to determine not only whether a new clinic captures patients, but whether the investment creates sustainable operational and financial value.

---

# 26. Net Strategic Impact

The strategic impact of opening a new clinic should consider:

```text
Additional Revenue
-
Additional Doctor Compensation
-
Additional Staff Costs
-
Additional Room Costs
-
Additional Equipment Costs
-
Additional Operating Costs
=
Net Impact
```

The model should distinguish between:

```text
Potential Demand
Actual Captured Demand
Actual Delivered Services
```

Opening a clinic does not automatically mean that all potential patients will be captured.

The final result depends on:

* effective capacity,
* waiting time,
* NFZ funding,
* patient behavior,
* operational resources.

---

# 27. Core Causal Chain

The core business logic of the simulator is:

```text
PATIENT DEMAND
        ↓
PRIMARY CARE
        ↓
REFERRAL
        ↓
TARGET SPECIALTY
        ↓
SPECIALTY AVAILABLE?
        │
        ├── NO → EXTERNAL LEAKAGE
        │
        └── YES
              ↓
        WAITING LIST
              ↓
      OPERATIONAL CAPACITY
              +
         NFZ CAPACITY
              ↓
       EFFECTIVE CAPACITY
              ↓
        WAITING TIME
              ↓
       PATIENT DECISION
          │     │     │
          ↓     ↓     ↓
        WAIT  EXTERNAL DROP
          │
          ↓
      FIRST VISIT
          ↓
      FOLLOW-UP
          ↓
   SERVICE DELIVERY
          ↓
    NFZ SETTLEMENT
          ↓
   REVENUE & COSTS
          ↓
  OPERATIONAL RESULT
          ↓
 STRATEGIC ANALYSIS
          ↓
 "SHOULD WE OPEN IT?"
```

---

# 28. Core Business KPIs

The simulator should support, at minimum, the following KPIs:

### Patient Flow

* Total Patients
* Total Referrals
* Internal Referrals
* External Referrals
* Not Completed Referrals
* Internal Capture Rate
* External Leakage Rate
* Referral Non-Completion Rate

### Operations

* Demand
* Operational Capacity
* NFZ Funded Capacity
* Effective Capacity
* Delivered Services
* Waiting List Size
* Average Waiting Time
* Doctor Utilization
* Room Utilization

### NFZ

* Contracted Volume
* Delivered Volume
* Contract Utilization
* Underutilization
* Overdemand

### Financial

* Total Service Revenue
* Doctor Compensation
* Clinic Revenue
* Operating Costs
* Net Clinic Result
* Margin %

### Strategic

* Potential Patients Captured
* Additional Revenue
* Additional Costs
* Net Impact
* Expected Waiting Time
* Expected Patient Leakage
* Resource Requirements

---

# 29. Model Design Principle

The most important principle of the simulator is:

> Operational events must have business consequences.

For example:

```text
More Referrals
↓
Higher Demand
↓
Higher Queue
↓
Longer Waiting Time
↓
Higher Patient Leakage
↓
Lower Internal Capture
↓
Lower Revenue
```

Alternatively:

```text
More Doctors
+
More Rooms
↓
Higher Operational Capacity
↓
Lower Waiting Time
↓
Higher Patient Capture
↓
More Delivered Services
↓
Higher Revenue
```

However:

```text
Higher Operational Capacity
+
Insufficient NFZ Contract
↓
Funding Bottleneck
↓
Limited Effective Capacity
```

Therefore, every major operational decision should affect downstream KPIs.

This ensures that the simulator behaves as an integrated business system rather than a collection of unrelated synthetic datasets.

---

# 30. Summary

The Healthcare Operations BI Simulator models a healthcare organization as a connected operational, financial, and strategic system.

The central business logic is based on the interaction between:

```text
Demand
+
Referrals
+
Specialty Availability
+
Doctors
+
Rooms
+
NFZ Contracts
+
Effective Capacity
+
Waiting Lists
+
Waiting Time
+
Patient Behavior
+
Service Delivery
+
Revenue
+
Operating Costs
```

The resulting model enables the organization to answer questions such as:

* Where are we losing patients?
* Why are patients leaving?
* Which specialties have the highest unmet demand?
* Is the bottleneck caused by doctors, rooms, or NFZ funding?
* Which clinics are underutilized?
* Which specialties generate the greatest potential internal demand?
* How much revenue is potentially lost through patient leakage?
* What would happen if we increased doctor capacity?
* What would happen if we added a new room?
* What would happen if the NFZ contract increased?
* Which new specialty should the organization open?
* Is opening that specialty economically justified?

The ultimate objective is to transform operational healthcare data into actionable management insight.

The project should therefore demonstrate not only data generation and visualization, but the complete analytical chain:

```text
Business Rules
→
Synthetic Data
→
Data Model
→
SQL Analysis
→
BI Metrics
→
Tableau Visualization
→
Strategic Decision Support
```
