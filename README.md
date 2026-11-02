# Freetown Community Flood Early Alert System

**PROG101 · Principles of Programming Logic and Design · Assignment 1**

A flood warning is most useful when residents know which ward it applies to and what action to take. This project designs a way to turn rainfall measurements, water levels, and residents’ SMS reports into local warnings in English or Krio. Residents could receive those warnings on a basic mobile phone.

## How a ward gets an alert

Every 15 minutes, the proposed system checks the readings and assigns each ward a level: **Green**, **Yellow**, **Orange**, or **Red**. A change in level triggers a message to that ward’s subscribers. A return to Green triggers an all-clear. If a reading fails validation, the system flags it for review and keeps the previous levels in effect.

In the report’s Portee example, 85 mm of rain in 24 hours exceeds the ward’s illustrative Orange threshold of 80 mm. Because Portee was previously Yellow, the system would text subscribers and notify community radio, volunteers, and the National Disaster Management Agency (NDMA) focal point. If nobody acknowledges the alert within 10 minutes, the system would contact the district disaster management committee and the NDMA duty officer.

**Scope.** This repository contains the assignment design and pseudocode. The example thresholds would need calibration with the Sierra Leone Meteorological Agency and NDMA before use in a live warning service.

## Read the design

The [assignment report (Word)](Freetown_Flood_Early_Alert_System_Report.docx) contains the decision rules in Section 2, pseudocode in Section 3, and worked scenarios in Section 6.

| Flowchart | What to look for |
| --- | --- |
| [1. Main system](Flowchart_1_Main_System.png) | Where readings are validated in each monitoring cycle. |
| [2. Ward processing loop](Flowchart_2_Ward_Processing_Loop.png) | How a change from the previous level triggers a warning. |
| [3. Risk classification](Flowchart_3_Risk_Classification.png) | Why the algorithm checks Red first when several thresholds are met. |
| [4. Sending SMS alerts](Flowchart_4_Sending_SMS_Alerts.png) | How the system selects each subscriber’s language and retries failed messages. |
| [5. Notification and escalation](Flowchart_5_Notification_and_Escalation.png) | Who is contacted when a serious alert goes unacknowledged. |
| [6. Module structure](Flowchart_6_Module_Structure.png) | How the seven modules divide responsibility for collecting data, making decisions, and delivering alerts. |
