# FCFEAS Algorithm Design

**Project:** Freetown Community Flood Early Alert System (FCFEAS)
**Module:** PROG101 Principles of Programming Logic and Design
**Author:** Emmanuel Alvin Macarthy (905006553)

## 1. Purpose

FCFEAS is a logical design (no source code) of a system that repeats a monitoring cycle every 15 minutes. It collects rainfall, water level and resident SMS reports, classifies each ward into one of four alert levels (GREEN, YELLOW, ORANGE, RED), and sends short SMS alerts in English and Krio. For serious alerts it also notifies radio, volunteers and sirens, and it escalates automatically if nobody acknowledges. It addresses the lack of timely, ward-specific flood warnings in Freetown (SDG 11 and SDG 13).

## 2. Input, Processing, Output

| Stage | Details |
|---|---|
| **Input** | rain24 (24-hour rainfall), rainHr (hourly rainfall), waterPct (water level as % of channel capacity), resident SMS (FLOOD, JOIN, STOP) |
| **Processing** | validation, high-risk factor, report confirmation, risk classification, message building, retries, escalation |
| **Output** | SMS alerts, radio and volunteer notifications, sirens (RED only), acknowledgement and escalation calls, log entries |

## 3. Constants

| Constant | Value | Data type | Meaning |
|---|---|---|---|
| CYCLE_MINUTES | 15 | Integer | Time between cycles |
| RAIN_YELLOW, RAIN_ORANGE, RAIN_RED | 50, 100, 150 | Real (mm) | 24-hour rainfall thresholds |
| INTENSITY_RED | 30 | Real (mm/h) | Hourly rainfall for RED when confirmed |
| WATER_YELLOW, WATER_ORANGE, WATER_RED | 60, 80, 100 | Real (%) | Water level thresholds |
| CONFIRM_COUNT | 3 | Integer | Reports needed to confirm flooding |
| HIGH_RISK_FACTOR | 0.8 | Real | Lowers rainfall thresholds in high-risk wards |
| ACK_WAIT_MINUTES | 10 | Integer | Wait for acknowledgement before escalation |
| MAX_RETRIES | 3 | Integer | SMS retry attempts |
| RAIN24_MAX, RAINHR_MAX, WATER_MAX | 500, 200, 300 | Real | Sanity limits for validation |

Values are illustrative and must be calibrated with SLMet and NDMA.

## 4. Main Variables

| Variable | Data type | Purpose |
|---|---|---|
| ward (code, displayName, highRisk) | Record: String, String, Boolean | A monitored ward |
| rain24, rainHr, waterPct | Real | Rainfall and water level readings |
| reportCount | Integer | Resident flood reports in the cycle |
| level, previousLevel | String (GREEN, YELLOW, ORANGE, RED) | Current and last alert level |
| confirmed, acknowledged, systemRunning | Boolean | Report confirmation, acknowledgement, loop control |
| factor | Real | 0.8 for high-risk wards, otherwise 1.0 |
| phone, language | String | Subscriber details (EN or KR) |
| sentCount, failCount, attempts | Integer | SMS delivery counters |
| wards, subscribers, retryQueue | List | Collections processed by loops |

## 5. Step-by-Step Algorithm

1. START the system.
2. Load constants, the ward list (with high-risk flag) and the subscriber list.
3. For every ward, set previousLevel to GREEN. Set systemRunning to TRUE.
4. WHILE systemRunning is TRUE, repeat steps 5 to 19.
5. INPUT: collect rain24 and rainHr, waterPct and incoming SMS. Count FLOOD reports per ward as reportCount.
6. Validate all readings against the sanity limits. If any value is invalid, log the error, flag the duty officer and go to step 19.
7. FOR EACH ward in the ward list, repeat steps 8 to 17.
8. If the ward is high-risk, set factor to 0.8; otherwise set factor to 1.0.
9. Set confirmed to TRUE if reportCount is at least 3.
10. Decision cascade: if the RED condition is met, level is RED; else if the ORANGE condition is met, level is ORANGE; else if the YELLOW condition is met, level is YELLOW; otherwise level is GREEN.
11. If level equals previousLevel, go to step 17 (no new alert).
12. Build the alert message in English and Krio for the new level (GREEN produces an "all clear").
13. Loop: for every subscriber in the ward, choose the language, send the SMS, count successes and queue failures. Retry queued messages up to 3 times.
14. If level is ORANGE or RED, notify community radio, ward volunteers and the NDMA focal point. If RED, also trigger the siren and door-to-door alert.
15. If level is ORANGE or RED, wait 10 minutes for acknowledgement. If none arrives, escalate to the District Disaster Management Committee and the NDMA duty officer.
16. Set previousLevel to level.
17. Move to the next ward (end of FOR EACH loop).
18. Save the cycle results to the log and update the dashboard.
19. WAIT 15 minutes. If a stop signal was received, set systemRunning to FALSE.
20. When the loop ends, save the final report and END.

## 6. Control Structures Used

| Structure | Where it appears |
|---|---|
| Repetition: WHILE | Main cycle (steps 4 and 19); retry loop (step 13) |
| Repetition: FOR EACH | Ward loop (step 7); subscriber loop (step 13); SMS intake |
| Decision: IF / ELSE | Validation (6), high-risk factor (8), level change (11), language choice (13), escalation (15) |
| Decision cascade | RED, ORANGE, YELLOW, GREEN (step 10) |
| Functions | Each task is a function, grouped into modules M1 to M7 |

## 7. Boolean Logic

| Expression | Result when TRUE |
|---|---|
| confirmed = (reportCount >= CONFIRM_COUNT) | Community reports support the sensor data |
| rain24 >= RAIN_RED * factor OR waterPct >= WATER_RED OR (rainHr >= INTENSITY_RED AND confirmed) | Level is RED |
| rain24 >= RAIN_ORANGE * factor OR waterPct >= WATER_ORANGE OR (rain24 >= RAIN_YELLOW * factor AND confirmed) | Level is ORANGE |
| rain24 >= RAIN_YELLOW * factor OR waterPct >= WATER_YELLOW | Level is YELLOW |
| level <> previousLevel | An alert must be sent |
| level = "ORANGE" OR level = "RED" | Notify radio and volunteers, check escalation |
| NOT acknowledged | Escalate to district committee and NDMA duty officer |

## 8. Functional Modules

| Module | Purpose | Functions |
|---|---|---|
| M1 Data Collection | Gather and validate readings and SMS | collectData, receiveSMSReport, isValidData |
| M2 Subscriber Management | Register and remove residents | registerSubscriber, removeSubscriber, getSubscribers |
| M3 Risk Analysis | Classify each ward | calculateRiskLevel, isHighRiskWard |
| M4 Message Builder | Create English and Krio alert text | buildMessage |
| M5 Alert Dissemination | Deliver alerts on all channels | sendAlerts, notifyRadioAndVolunteers, triggerSiren |
| M6 Escalation | Make sure serious alerts are acted on | checkEscalation |
| M7 Logging and Reporting | Keep an audit trail and final report | logEvent, saveFinalReport |

## 9. Sample Trace (Scenario 4: Portee)

**Inputs:** rain24 = 85, rainHr = 10, waterPct = 70, reportCount = 4, previousLevel = YELLOW. PORTEE is a high-risk ward.

| Step | Action | Result |
|---|---|---|
| 8 | Ward is high-risk | factor = 0.8 |
| 9 | 4 >= 3 | confirmed = TRUE |
| 10 | RED test: 85 >= 150 x 0.8 = 120? No. 70 >= 100? No. 10 >= 30? No. | RED not met |
| 10 | ORANGE test: 85 >= 100 x 0.8 = 80? Yes. | level = ORANGE |
| 11 | ORANGE <> YELLOW | Level changed, alert needed |
| 12 to 13 | Build message and send SMS to PORTEE subscribers | English or Krio per subscriber |
| 14 | Level is ORANGE | Radio, volunteers and NDMA focal point notified. No siren. |
| 15 | Wait 10 minutes for acknowledgement | Escalate if none received |
| 16 | previousLevel = ORANGE | Stored for the next cycle |

