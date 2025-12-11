# kholoud7
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ------------ Hardware pins ------------
const int MQ2   = A0;
const int MQ3   = A1;
const int MQ4   = A2;
const int MQ5   = A3;
const int MQ9   = A4;
const int MQ135 = A5;

const int fanPin   = 22;
const int greenLED = 7;
const int redLED   = 8;
const int buzzer   = 6; // passive buzzer (use tone)

// ------------ LCD I2C ------------
LiquidCrystal_I2C lcd(0x27, 16, 2); // change address if needed

// ------------ Sampling params ------------
const int samples = 10;
const unsigned long sampleDelay = 10;
const unsigned long loopDelay = 300; // ms

// ------------ Baseline values ------------
float base_MQ2=0, base_MQ3=0, base_MQ4=0, base_MQ5=0, base_MQ9=0, base_MQ135=0;

// ------------ temp variables ------------
int v2,v3,v4,v5,v9,v135;
float d2,d3,d4,d5,d9,d135;

String deviceState = "IDLE";

// ------------ SMS / SIM ------------
// SIM800L on Serial2 (Mega): TX2 pin 16, RX2 pin 17
const char phoneNumber[] = "+966530564202"; // your phone number in international format

// --------------- helper: readAverage ----------------
int readAverage(int pin) {
  long total = 0;
  for (int i=0;i<samples;i++){
    total += analogRead(pin);
    delay(sampleDelay);
  }
  return total / samples;
}

// --------------- helper: read SIM response (basic) --------------
String readSIMResponse(unsigned long timeout) {
  unsigned long t0 = millis();
  String res = "";
  while (millis() - t0 < timeout) {
    while (Serial2.available()) {
      char c = Serial2.read();
      res += c;
    }
  }
  Serial.print("SIM_RESP: ");
  Serial.println(res);
  return res;
}

// --------------- helper: sendSMS via SIM800L ----------------
void sendSMS(const char* msg) {
  Serial.println("Sending SMS...");
  Serial2.println("AT");
  delay(500);
  readSIMResponse(500);
  Serial2.println("AT+CMGF=1"); delay(500);
  readSIMResponse(500);
  Serial2.print("AT+CMGS=\""); Serial2.print(phoneNumber); Serial2.println("\"");
  delay(300);
  Serial2.print(msg);
  delay(200);
  Serial2.write(26); // CTRL+Z
  readSIMResponse(8000);
  Serial.println("SMS command done.");
}

// --------------- helper: try get location via SIM800L (CIPGSMLOC) --------------
String getSIMLocation() {
  Serial2.println("AT+CIPGSMLOC=1,1");
  String r = readSIMResponse(4000);
  int idx = r.indexOf("+CIPGSMLOC:");
  if (idx >= 0) {
    int nl = r.indexOf('\n', idx);
    String line;
    if (nl > idx) line = r.substring(idx, nl);
    else line = r.substring(idx);
    line.replace("+CIPGSMLOC:",""); line.trim();
    int p1 = line.indexOf(',');
    int p2 = (p1>=0)?line.indexOf(',', p1+1):-1;
    if (p1>0 && p2>p1) {
      String lat = line.substring(p1+1, p2);
      String rest = line.substring(p2+1);
      int p3 = rest.indexOf(',');
      String lon = (p3>0)? rest.substring(0,p3):rest;
      lat.trim(); lon.trim();
      return String(lat) + "," + String(lon);
    }
  }
  return String("N/A");
}

// ---------------- detection logic (rule-based, needs calibration) ----------------
struct Detection {
  String state;         // "SAFE", "ALERT", "DANGER"
  String type;          // "None", "Cigarette", "Incense", "Paper", "Alcohol", "Cannabis Oil"
  int percentCannabis;  // 0..100
};

Detection detectFromDeltas() {
  Detection out; out.state="SAFE"; out.type="None"; out.percentCannabis=0;
  // thresholds (estimates - calibrate)
  const float TH_ALCOHOL = 120.0;
  const float TH_CIGARETTE_MQ9 = 100.0;
  const float TH_GENERAL_SMOKE = 80.0;
  const float TH_INCENSE = 140.0;
  const float TH_PAPER = 200.0;
  const float TH_CANNABIS = 130.0;

  // Alcohol (MQ3 dominant)
  if (d3 > TH_ALCOHOL && d2 < TH_GENERAL_SMOKE) {
    out.state = "ALERT";
    out.type = "Alcohol";
    return out;
  }

  // Cannabis oil (heuristic: high MQ135 and moderate MQ3)
  if (d135 > TH_CANNABIS && d3 > 40) {
    out.state = "DANGER";
    out.type = "Cannabis Oil";
    int perc = (int)constrain((d135/3.0), 1, 100); // rough scale
    out.percentCannabis = perc;
    return out;
  }

  // Incense
  if (d135 > TH_INCENSE && d2 > TH_GENERAL_SMOKE) {
    out.state = "ALERT";
    out.type = "Incense";
    return out;
  }

  // Cigarette (CO / MQ9 high + MQ2)
  if (d9 > TH_CIGARETTE_MQ9 && d2 > TH_GENERAL_SMOKE) {
    out.state = "ALERT";
    out.type = "Cigarette";
    return out;
  }

  // Burning paper
  if (d2 > TH_PAPER && d9 < TH_CIGARETTE_MQ9) {
    out.state = "ALERT";
    out.type = "Burning Paper";
    return out;
  }

  // General smoke detection (unknown)
  if (d2 > TH_GENERAL_SMOKE || d135 > TH_GENERAL_SMOKE) {
    out.state = "ALERT";
    out.type = "Unknown Smoke";
    return out;
  }

  // Default safe
  out.state="SAFE";
  out.type="None";
  return out;
}

// ---------------- setup ----------------
void setup() {
  Serial.begin(9600);           // USB monitor
  Serial1.begin(9600);          // Bluetooth HC-05 (TX1/RX1)
  Serial2.begin(9600);          // SIM800L (TX2/RX2)

  pinMode(fanPin, OUTPUT); digitalWrite(fanPin, LOW);
  pinMode(greenLED, OUTPUT); digitalWrite(greenLED, LOW);
  pinMode(redLED, OUTPUT); digitalWrite(redLED, LOW);
  pinMode(buzzer, OUTPUT); digitalWrite(buzzer, LOW);

  // init LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print("System Starting...");
  lcd.setCursor(0,1);
  lcd.print("Calibrating...");
  delay(1500);

  // measure baseline (in clean air)
  base_MQ2   = readAverage(MQ2);
  base_MQ3   = readAverage(MQ3);
  base_MQ4   = readAverage(MQ4);
  base_MQ5   = readAverage(MQ5);
  base_MQ9   = readAverage(MQ9);
  base_MQ135 = readAverage(MQ135);

  Serial.println("Baseline values:");
  Serial.print("b2:"); Serial.println(base_MQ2);
  Serial.print("b3:"); Serial.println(base_MQ3);
  Serial.print("b4:"); Serial.println(base_MQ4);
  Serial.print("b5:"); Serial.println(base_MQ5);
  Serial.print("b9:"); Serial.println(base_MQ9);
  Serial.print("b135:"); Serial.println(base_MQ135);

  lcd.clear();
  lcd.setCursor(0,0); lcd.print("Baseline Ready");
  delay(800);
  deviceState = "SAMPLING";
}

// ---------------- main loop ----------------
unsigned long lastLoop = 0;
void loop() {
  unsigned long now = millis();
  if (now - lastLoop < loopDelay) return;
  lastLoop = now;

  // control fan if sampling
  if (deviceState == "SAMPLING") digitalWrite(fanPin, HIGH);
  else digitalWrite(fanPin, LOW);

  // read sensors
  v2   = readAverage(MQ2);
  v3   = readAverage(MQ3);
  v4   = readAverage(MQ4);
  v5   = readAverage(MQ5);
  v9   = readAverage(MQ9);
  v135 = readAverage(MQ135);

  // deltas
  d2   = v2   - base_MQ2;
  d3   = v3   - base_MQ3;
  d4   = v4   - base_MQ4;
  d5   = v5   - base_MQ5;
  d9   = v9   - base_MQ9;
  d135 = v135 - base_MQ135;

  // detect
  Detection res = detectFromDeltas();

  // prepare display text and bluetooth/SMS payload
  String line1 = "";
  String line2 = "";

  if (res.state == "DANGER" && res.type == "Cannabis Oil") {
    line1 = "DANGER!";
    line2 = "Cannabis Oil";
    digitalWrite(greenLED, LOW);
    digitalWrite(redLED, HIGH);
    tone(buzzer, 1500); // continuous tone while detected
  } else {
    noTone(buzzer);
    digitalWrite(redLED, LOW);
    digitalWrite(greenLED, HIGH); // green on when not cannabis danger
    if (res.state == "SAFE") {
      line1 = "SAFE";
      line2 = "No Smoke";
    } else {
      // ALERT states
      if (res.type == "Cigarette") { line1 = "Cigarette"; line2 = "Smoke Detected"; }
      else if (res.type == "Incense") { line1 = "Incense"; line2 = "Smoke Detected"; }
      else if (res.type == "Burning Paper") { line1 = "Burning Paper"; line2 = "Smoke Detected"; }
      else if (res.type == "Alcohol") { line1 = "Alcohol"; line2 = "Vapor Detected"; }
      else if (res.type == "Unknown Smoke") { line1 = "Unknown"; line2 = "Smoke Detected"; }
      else { line1 = "SAFE"; line2 = "No Smoke"; }
    }
  }

  // update LCD (16x2)
  lcd.clear();
  lcd.setCursor(0,0);
  lcd.print(line1);
  lcd.setCursor(0,1);
  // ensure second line fits 16 chars
  if (line2.length() > 16) line2 = line2.substring(0,16);
  lcd.print(line2);

  // prepare JSON-like message for Bluetooth
  unsigned long tsec = millis()/1000;
  String btMsg = "{";
  btMsg += "\"timestamp\":" + String(tsec) + ",";
  btMsg += "\"state\":\"" + res.state + "\",";
  btMsg += "\"type\":\"" + res.type + "\",";
  btMsg += "\"cannabis_percent\":" + String(res.percentCannabis) + ",";
  btMsg += "\"d2\":" + String((int)d2) + ",";
  btMsg += "\"d3\":" + String((int)d3) + ",";
  btMsg += "\"d4\":" + String((int)d4) + ",";
  btMsg += "\"d5\":" + String((int)d5) + ",";
  btMsg += "\"d9\":" + String((int)d9) + ",";
  btMsg += "\"d135\":" + String((int)d135);
  btMsg += "}";

  // send via Bluetooth and USB Serial
  Serial1.println(btMsg);
  Serial.println(btMsg);

  // If cannabis detected -> send SMS with percentage + time + location + name
  static unsigned long lastSmsTime = 0;
  const unsigned long smsCooldown = 5UL * 60UL * 1000UL; // 5 min
  if (res.state == "DANGER" && res.type == "Cannabis Oil") {
    if (millis() - lastSmsTime > smsCooldown) {
      lastSmsTime = millis();
      String loc = getSIMLocation(); // may return N/A
      String sms = "DANGER ALERT!\nCannabis Oil Detected\nLevel: " + String(res.percentCannabis) + "%\nTime(s): " + String(tsec) + "\nLocation: " + loc;
      sendSMS(sms.c_str());
    }
  }

  // loop end
}
