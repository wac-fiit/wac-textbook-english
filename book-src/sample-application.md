# Practice Application

The goal of the practice application is to demonstrate development procedures on a specific example that can be linked to real-world practices. The practice application consists of several micro-applications (can be understood as micro-services within a [12-factor][twelves] application), which are ultimately connected into a cohesive system.

## Problem Definition

The district hospital currently does not have any central information system, causing inefficiency in managing its operations and dissatisfaction among customers. The hospital has decided to invest in a central information system, which, in its final configuration, would cover all aspects of its activities. From an economic perspective, the requirement for system implementation is that the system is modular, can be developed gradually, and can be continuously updated based on current requirements and involving multiple suppliers at different times. Another requirement is that the system can be deployed in a data center available to the hospital (the specification intentionally left open, various deployment options will be addressed gradually).

From a functional perspective, the system should include the following applications:

* Waiting room management for individual clinics.
* Ordering system based on patient requests, doctor dispatch, or requests from a specialist doctor (e.g., regular check-ups).
* Management of medical documentation within individual clinics.
* Management of medical documentation coming from external clinics in electronic form, its processing, (automated) verification, and allocation to the respective clinics.
* Management of the hospital's bed section by individual departments and their current and planned patient occupancy.
* Management of medicines in clinics, their dispensing to patients, and ordering in the hospital pharmacy.
* Monitoring of performances within individual clinics and their economic costs.
* Portal for patients, where they can get an overview of their acute and chronic diseases, as well as scheduled and completed hospital visits.
* Professional online counseling corresponding to the current health condition of a registered or foreign patient.
* Support for remote examination in selected diseases and/or in specific patients.
* Record and ordering of hospital and clinic equipment, with an overview of the location and lifespan of individual items.
* Record of hospital spaces and their allocation to individual clinics or departments.
* Personnel management, overview of hospital employees, their assignment to clinics and hospital departments, performance overview, and personal documentation management.

Some of these applications are partially covered within this textbook. Students have the opportunity to choose one of the applications for their individual work during exercises and as an assignment for the final evaluation of mastering the study material.
