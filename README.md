# Water-Leakage-Detection-and-Recognition-System
The water leakage detection and recognition system is a pioneering initiative designed to revolutionize leak detection in water distribution networks by leveraging cutting-edge technologies, including machine learning algorithms and Internet of Things (IoT) solutions.
////////////////Arduino CODE///////////////////////////////////
#include <LiquidCrystal.h>
LiquidCrystal lcd(8, 9, 4, 5, 6, 7);
#include <SoftwareSerial.h>
SoftwareSerial ESP11(10, 11); // RX, TX
#define DEBUG true
String ssid = "\"wifi001\"";
String pass = "\"12345678\"";
String tcp = "\"SSL\"";
String remoteip = "\"iot-projects.eweb.org.in\"";
String portnum = "443";
int buz = 13;
const int flowMeter1Pin = 2;  
const int flowMeter2Pin = 3;  
volatile unsigned long pulseCount1 = 0;  
volatile unsigned long pulseCount2 = 0;
float flowRate1 = 0;
float flowRate2 = 0;
float calibrationFactor = 4.5;  
unsigned long lastFlowCalcTime = 0;
unsigned long lastServerUpdateTime = 0;
unsigned long flowCalcInterval = 1000;  
unsigned long serverUpdateInterval = 5000;  
float leakThreshold = 20.0;  
void setup() {
  Serial.begin(115200);
  ESP11.begin(115200);
  lcd.begin(16, 2);
  lcd.print("Water Leakage");
  lcd.setCursor(0, 1);
  lcd.print("Monitoring...");
  pinMode(buz, OUTPUT);
  digitalWrite(buz, LOW);
  attachInterrupt(digitalPinToInterrupt(flowMeter1Pin), pulseCounter1, FALLING);  // ISR for flowmeter 1
  attachInterrupt(digitalPinToInterrupt(flowMeter2Pin), pulseCounter2, FALLING);  // ISR for flowmeter 2
  delay(2000);
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("  Loading!....");
  sendData("AT+CWMODE=3\r\n", 2000, DEBUG); // Configure as access point and client
  lcd.setCursor(0, 1);
  lcd.print("      20%   ");
  sendData("AT+RST\r\n", 2000, DEBUG); // Reset module
  lcd.setCursor(0, 1);
  lcd.print("      40%   ");
  sendData("AT+GMR\r\n", 2000, DEBUG); // View version Info
  lcd.setCursor(0, 1);
  lcd.print("      60%  ");
  sendData("AT+CIFSR\r\n", 2000, DEBUG); // Get IP address
  lcd.setCursor(0, 1);
  lcd.print("      90%   ");
  lcd.clear();
}

void loop() {
  unsigned long currentTime = millis();

  
  if (currentTime - lastFlowCalcTime >= flowCalcInterval) {
    
    noInterrupts();
    flowRate1 = (pulseCount1 / calibrationFactor);
    flowRate2 = (pulseCount2 / calibrationFactor);
    pulseCount1 = 0;  
    pulseCount2 = 0;
    interrupts();  

    lastFlowCalcTime = currentTime;

    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Flow1: ");
    lcd.print(flowRate1);
    lcd.print(" L/min");
    lcd.setCursor(0, 1);
    lcd.print("Flow2: ");
    lcd.print(flowRate2);
    lcd.print(" L/min");

       if (flowRate1 >= flowRate2 + leakThreshold) {
      delay(2000);
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("    Water     ");
      lcd.setCursor(0, 1);
      lcd.print("Leak Detected!");
      digitalWrite(buz, HIGH);
      delay(2000);  // Short delay for alarm
      digitalWrite(buz, LOW);  // Stop buzzer
    }
  }

 
  if (currentTime - lastServerUpdateTime >= serverUpdateInterval) {
    sent();
    lastServerUpdateTime = currentTime;
  }
  delay(2000);
}


void pulseCounter1() {
  pulseCount1++;
}


void pulseCounter2() {
  pulseCount2++;
}

String sendData(String command, const int timeout, boolean debug) {
  String response = "";
  Serial.print(command);

  long int time = millis();
  while ((time + timeout) > millis()) {
    if (Serial.available()) {
      char c = Serial.read();
      ESP11.write(c);
    }
  }

  if (debug) {
    
  }
  return response;
}

void sent() {
  sendData("AT+CIPSTART=" + SSL + "," + remoteip + "," + portnum + "\r\n", 1000, DEBUG);

  String getStr = "GET /water_leakage/update.php?f1=";
  getStr += flowRate1;
  getStr += "&f2=";
  getStr += flowRate2;
  String cmd = "AT+CIPSEND=";
  cmd += String(getStr.length());
  sendData(cmd + "\r\n", 1000, DEBUG);
  sendData(getStr, 1000, DEBUG);

  String closeCommand = "AT+CIPCLOSE\r\n";
  sendData(closeCommand, 2000, DEBUG);
}
