==============================================================
PROG101 - Principles of Programming Logic and Design
Freetown Community Flood Early Alert System (FCFEAS)
Pseudocode (no source code)
Author: Emmanuel Alvin Macarthy (905006553)
==============================================================

NOTE: Threshold values are illustrative and must be calibrated
with SLMet and NDMA. Lines starting with // are comments.

--------------------------------------------------------------
1. MAIN PROGRAM
--------------------------------------------------------------

PROGRAM FreetownFloodAlertSystem

  // ----- Constants -----
  CONSTANT CYCLE_MINUTES = 15
  CONSTANT RAIN_YELLOW = 50, RAIN_ORANGE = 100, RAIN_RED = 150   // mm in 24 hours
  CONSTANT INTENSITY_RED = 30                                    // mm per hour
  CONSTANT WATER_YELLOW = 60, WATER_ORANGE = 80, WATER_RED = 100 // % of channel
  CONSTANT CONFIRM_COUNT = 3
  CONSTANT HIGH_RISK_FACTOR = 0.8
  CONSTANT ACK_WAIT_MINUTES = 10, MAX_RETRIES = 3
  CONSTANT RAIN24_MAX = 500, RAINHR_MAX = 200, WATER_MAX = 300

  // ----- Variables -----
  DECLARE wards AS ARRAY OF Ward
  DECLARE data AS ARRAY OF WardData
  DECLARE previousLevel AS ARRAY OF STRING
  DECLARE level AS STRING
  DECLARE pair AS MessagePair          // pair.english and pair.krio
  DECLARE systemRunning AS BOOLEAN

  CALL loadWards(wards)
  FOR EACH ward IN wards DO
    SET previousLevel[ward.code] = "GREEN"
  END FOR

  SET systemRunning = TRUE

  WHILE systemRunning = TRUE DO
    SET data = collectData(wards)

    IF isValidData(data) = FALSE THEN
      CALL logEvent("INVALID DATA", "Duty officer review needed")
    ELSE
      FOR EACH ward IN wards DO
        SET level = calculateRiskLevel(ward, data[ward.code])

        IF level <> previousLevel[ward.code] THEN
          SET pair = buildMessage(ward, level)
          CALL sendAlerts(ward, level, pair)
          CALL checkEscalation(ward, level)
          SET previousLevel[ward.code] = level
        END IF
      END FOR
      CALL logEvent("CYCLE COMPLETE", previousLevel)
    END IF

    WAIT CYCLE_MINUTES minutes
    IF stopSignalReceived() = TRUE THEN SET systemRunning = FALSE
  END WHILE

  CALL saveFinalReport()
END PROGRAM


--------------------------------------------------------------
2. MODULE 1: DATA COLLECTION
--------------------------------------------------------------

FUNCTION collectData(wards) RETURNS ARRAY OF WardData
  DECLARE result AS ARRAY OF WardData
  DECLARE incoming AS LIST OF STRING

  FOR EACH ward IN wards DO
    SET result[ward.code].rain24 = readRainfall24h(ward)        // INPUT: gauge or volunteer
    SET result[ward.code].rainHr = readRainfallHourly(ward)
    SET result[ward.code].waterPct = readWaterLevelPercent(ward)
    SET result[ward.code].reportCount = 0
  END FOR

  SET incoming = fetchIncomingSMS()
  FOR EACH sms IN incoming DO
    CALL receiveSMSReport(sms, result)
  END FOR

  RETURN result
END FUNCTION


FUNCTION receiveSMSReport(sms, result)
  DECLARE command, wardCode AS STRING
  DECLARE severity AS INTEGER        // 1 = street, 2 = doorstep, 3 = inside home

  SET command = firstWord(sms.text)

  IF command = "FLOOD" THEN
    SET wardCode = secondWord(sms.text)
    SET severity = toNumber(thirdWord(sms.text))

    IF wardExists(wardCode) = TRUE AND severity >= 1 AND severity <= 3 THEN
      SET result[wardCode].reportCount = result[wardCode].reportCount + 1
      OUTPUT sendSMS(sms.phone, "Thank you. Your flood report was received.")
    ELSE
      OUTPUT sendSMS(sms.phone, "Wrong format. Send: FLOOD WARDCODE 1-3")
    END IF

  ELSE IF command = "JOIN" THEN
    CALL registerSubscriber(sms.phone, secondWord(sms.text), thirdWord(sms.text))

  ELSE IF command = "STOP" THEN
    CALL removeSubscriber(sms.phone)
  END IF
END FUNCTION


FUNCTION isValidData(data) RETURNS BOOLEAN
  FOR EACH entry IN data DO
    IF entry.rain24 < 0 OR entry.rain24 > RAIN24_MAX THEN RETURN FALSE
    IF entry.rainHr < 0 OR entry.rainHr > RAINHR_MAX THEN RETURN FALSE
    IF entry.waterPct < 0 OR entry.waterPct > WATER_MAX THEN RETURN FALSE
  END FOR
  RETURN TRUE
END FUNCTION


--------------------------------------------------------------
3. MODULE 2: SUBSCRIBER MANAGEMENT
--------------------------------------------------------------

FUNCTION registerSubscriber(phone, wardCode, languageCode) RETURNS BOOLEAN
  IF isValidPhone(phone) = FALSE OR wardExists(wardCode) = FALSE THEN
    OUTPUT sendSMS(phone, "Could not join. Send: JOIN WARDCODE EN or KR")
    RETURN FALSE
  END IF

  IF languageCode <> "KR" THEN SET languageCode = "EN"

  IF subscriberExists(phone) = TRUE THEN
    CALL updateSubscriber(phone, wardCode, languageCode)
  ELSE
    CALL addSubscriber(phone, wardCode, languageCode)
  END IF

  OUTPUT sendSMS(phone, "You will receive flood alerts for " + wardCode + ". Send STOP to leave.")
  RETURN TRUE
END FUNCTION


FUNCTION removeSubscriber(phone)
  IF subscriberExists(phone) = TRUE THEN CALL deleteSubscriber(phone)
  OUTPUT sendSMS(phone, "You have been removed from flood alerts.")
END FUNCTION


FUNCTION getSubscribers(wardCode) RETURNS LIST OF Subscriber
  DECLARE result AS LIST OF Subscriber

  FOR EACH sub IN allSubscribers DO
    IF sub.wardCode = wardCode THEN
      APPEND sub TO result
    END IF
  END FOR

  RETURN result
END FUNCTION


--------------------------------------------------------------
4. MODULE 3: RISK ANALYSIS
--------------------------------------------------------------

FUNCTION calculateRiskLevel(ward, entry) RETURNS STRING
  DECLARE factor AS REAL
  DECLARE confirmed AS BOOLEAN

  IF isHighRiskWard(ward) = TRUE THEN
    SET factor = HIGH_RISK_FACTOR
  ELSE
    SET factor = 1.0
  END IF

  SET confirmed = (entry.reportCount >= CONFIRM_COUNT)

  IF entry.rain24 >= RAIN_RED * factor OR entry.waterPct >= WATER_RED
     OR (entry.rainHr >= INTENSITY_RED AND confirmed) THEN
    RETURN "RED"
  ELSE IF entry.rain24 >= RAIN_ORANGE * factor OR entry.waterPct >= WATER_ORANGE
     OR (entry.rain24 >= RAIN_YELLOW * factor AND confirmed) THEN
    RETURN "ORANGE"
  ELSE IF entry.rain24 >= RAIN_YELLOW * factor OR entry.waterPct >= WATER_YELLOW THEN
    RETURN "YELLOW"
  ELSE
    RETURN "GREEN"
  END IF
END FUNCTION


FUNCTION isHighRiskWard(ward) RETURNS BOOLEAN
  RETURN ward.highRisk = TRUE
END FUNCTION


--------------------------------------------------------------
5. MODULE 4: MESSAGE BUILDER
--------------------------------------------------------------

FUNCTION buildMessage(ward, level) RETURNS MessagePair
