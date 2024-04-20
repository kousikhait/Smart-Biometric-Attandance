# Smart-Biometric-Attandance
//libraries*******************************
//ESP32----------------------------
#include <WiFi.h>
#include <NTPClient.h>
#include <WiFiUdp.h>
#include <FirebaseESP32.h>
#include <SimpleTimer.h>  //https://github.com/jfturcot/SimpleTimer
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27,20,4); 
//OLED-----------------------------
//#include <SPI.h>
//#include <Wire.h>
//#include <Adafruit_GFX.h>          //https://github.com/adafruit/Adafruit-GFX-Library
//#include <Adafruit_SSD1306.h>      //https://github.com/adafruit/Adafruit_SSD1306
#include <Adafruit_Fingerprint.h>  //https://github.com/adafruit/Adafruit-Fingerprint-Sensor-Library
//************************************************************************
//Fingerprint scanner Pins (Serial2 pins Rx2 & Tx2)
#define Finger_Rx 16  //Rx2
#define Finger_Tx 17  //Tx2

//************************************************************************
SimpleTimer timer;
HardwareSerial mySerial(2);  //ESP32 Hardware Serial 2
Adafruit_Fingerprint finger = Adafruit_Fingerprint(&mySerial);
//************************************************************************
/* Set these to your desired credentials. */
const char *ssid = "LAPTOP-7TANPE82 4015";
const char *password = "12345678";
//************************************************************************
String getData, Link;
//String stat = true;
//************************************************************************
int FingerID = 0;  // The Fingerprint ID from the scanner
uint8_t id;
//Biometric Icons********************************


//FIREBASE AUTH//

//1. Change the following info
#define FIREBASE_HOST "biometric-attandance-default-rtdb.asia-southeast1.firebasedatabase.app/"
#define FIREBASE_AUTH "vs3cODd4v6vCQUHYxFF25MyrGRxLU80swUchxFrD"

//2. Define FirebaseESP32 data object for data sending and receiving
FirebaseData firebaseData, firebaseData1, firebaseData2, firebaseData3, firebaseData4;
FirebaseJson json,json1,json2;



WiFiUDP ntpUDP;
#define offset 19800                    //UTC -1 = -3600,UTC +1 = 3600, UTC +2 = 7200 (FOR EVERY +1 HOUR ADD 3600,SUBTRACT 3600 FOR -1 HOUR)
NTPClient timeClient(ntpUDP, "pool.ntp.org");

////
void setup() {

   lcd.init();                      // initialize the lcd 
  // Print a message to the LCD.
  lcd.backlight();
  // lcd.setCursor(2,0);
  // lcd.print("Group Mem G-6  ");
  // lcd.setCursor(2,1);
  // lcd.print(" Biometric Attandance");
  //  lcd.setCursor(0,2);
  // lcd.print("Arduino LCM IIC 2004");
  //  lcd.setCursor(2,3);
  // lcd.print("Power By Ec-yuan!");

  pinMode(23, OUTPUT);
  pinMode(2, OUTPUT);

  Serial.begin(9600);
   timeClient.begin();
   timeClient.setTimeOffset(offset);
  delay(100);
  //-----------initiate OLED display-------------

  //---------------------------------------------
  connectToWiFi();
  //---------------------------------------------

  Firebase.begin(FIREBASE_HOST, FIREBASE_AUTH);
  //4. Enable auto reconnect the WiFi when connection lost
  Firebase.reconnectWiFi(true);



  // Set the data rate for the sensor serial port
  finger.begin(57600);
  Serial.println("\n\nAdafruit finger detect test");

  if (finger.verifyPassword()) {
    Serial.println("Found fingerprint sensor!");
     lcd.setCursor(0,0);
  lcd.print("Found fingerprint sensor!");
  digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(23, LOW);

  } else {
    Serial.println("Did not find fingerprint sensor :(");
    lcd.setCursor(2,0);
  lcd.print("Did not find fingerprint sensor :(");
  delay(5000);
   lcd.clear();
  digitalWrite(2, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(2, LOW);
    while (1) { delay(1); }
  }
  //---------------------------------------------
  finger.getTemplateCount();
  Serial.print("Sensor contains ");
  Serial.print(finger.templateCount);
  Serial.println(" templates");
  Serial.println("Waiting for valid finger...");
   lcd.setCursor(0,0);
  lcd.print("Waiting for valid finger...");
  delay(5000);
   lcd.clear();
  digitalWrite(2, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(2, LOW);

  //---------------------------------------------
  timer.setInterval(10000L, ChecktoAddID);     //Set an internal timer every 10sec to check if there a new fingerprint in the website to add it.
  timer.setInterval(15000L, ChecktoDeleteID);  //Set an internal timer every 15sec to check wheater there an ID to delete in the website.
}
//************************************************************************
void loop() {

//  digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
//   delay(1000);                      // wait for a second
//   digitalWrite(23, LOW);   // turn the LED off by making the voltage LOW
//   delay(1000); 


  timer.run();  //Keep the timer in the loop function in order to update the time as soon as possible
  //check if there's a connection to Wi-Fi or not
  if (!WiFi.isConnected()) {
    connectToWiFi();  //Retry to connect to Wi-Fi
  }
  CheckFingerprint();  //Check the sensor if the there a finger.
  delay(500);
}
//************************************************************************
void CheckFingerprint() {
  //  unsigned long previousMillisM = millis();
  //  Serial.println(previousMillisM);
  // If there no fingerprint has been scanned return -1 or -2 if there an error or 0 if there nothing, The ID start form 1 to 127
  // Get the Fingerprint ID from the Scanner
  FingerID = getFingerprintID();
  DisplayFingerprintID();
  //  Serial.println(millis() - previousMillisM);
}
//*Display the fingerprint ID state on the OLED************
void DisplayFingerprintID() {
  //Fingerprint has been detected
  if (FingerID > 0) {
    Serial.println("Yourd finferprint id is" + String(FingerID));
     digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(23, LOW);
    lcd.setCursor(0,0);
    lcd.print("Yourd finferprint id is" + String(FingerID));
    delay(1000);
   lcd.clear();
    time_update();
    delay(1000);
  //    digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
  // delay(1000);                      // wait for a second
  // digitalWrite(23, LOW);   // turn the LED off by making the voltage LOW
  // delay(1000);
    SendFingerprintID(FingerID);  // Send the Fingerprint ID to the website.
    delay(1000);
  }
  //---------------------------------------------
  //No finger detected
  else if (FingerID == 0) {
    Serial.println("No finger detected");
    lcd.setCursor(0,0);
  lcd.print("No finger detected");
  delay(1000);
   lcd.clear();

  //  lcd.setCursor(2,1);
  //  lcd.print(" detected");
    digitalWrite(2, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(2, LOW);
  }
  //---------------------------------------------
  //Didn't find a match
  else if (FingerID == -1) {
    Serial.println("Didn't find a match");
     lcd.setCursor(0,0);
  lcd.print("sorry!:Didn't");
  lcd.setCursor(0,1);
  lcd.print("find a match");
  delay(1000);
   lcd.clear();

    digitalWrite(2, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(2, LOW);
  }
  //---------------------------------------------
  //Didn't find the scanner or there an error
  else if (FingerID == -2) {
    Serial.println(" sorry!:Didn't find the scanner or there an error");
    lcd.setCursor(0,0);
  lcd.print("Didn't find the");
  lcd.setCursor(0,1);
  lcd.print("scanner or there an error");
  delay(5000);
   lcd.clear();

  digitalWrite(2, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(2, LOW);
  }
}

void time_update() {
  String path;
  path = "/Student Info/Data/" + String(FingerID) + "/Intime";

  /*
    if(stat == "1"){
        path = "/Student Info/Data/"+String(FingerID)+"/Intime";
        stat = "0";
    }
    else
    {
        path = "/Student Info/Data/"+String(FingerID)+"/Outtime";
        stat = "1";
    }
    */

  /*
    String td = ntp_time();

    if(Firebase.setString(firebaseData, path,td))
    {
      Serial.println("Time updated");
    }
    else{
        Serial.println("Time not updated");
    }

    */
}


//*send the fingerprint ID to the website************
void SendFingerprintID(int finger) {
  Serial.println("Sending the Fingerprint ID");

  //getData = "?FingerID=" + String(finger); // Add the Fingerprint ID to the Post array in order to send it
  //GET methode
  //    Link = URL + getData;

  String path = "/Student Info/Data/" + String(finger) + "/name";
  String path1 = "/Student Info/Data/" + String(finger) + "/email";

  if (Firebase.getString(firebaseData, path)) {
    //Success
    Serial.print("welcome!! :");
     lcd.setCursor(0,0);
  lcd.print("welcome!! :");
    // Serial.print("your email is :"); 
  
    Serial.println(firebaseData.stringData());
    digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(23, LOW);
    lcd.setCursor(0,1);
  lcd.print(firebaseData.stringData());
  delay(5000);
   lcd.clear();
  //  delay(1000);
    // Firebase.getString(firebaseData, path1);
    // Serial.print("your email is :")
    // Serial.println(firebaseData.stringData());
//  digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
//   delay(1000);                      // wait for a second
//   digitalWrite(23, LOW);
  // delay(5000);
  //  lcd.clear();   // turn the LED off by making the voltage LOW
  // delay(1000);

  } 
  // else if (Firebase.getString(firebaseData, path1)) {
  //   //Success
  //   Serial.print("your email is :");
  //   Serial.println(firebaseData.stringData());


  // } 
  else {
    //Failed?, get the error reason from firebaseData

    Serial.print("Error in setInt, ");
    Serial.println(firebaseData.errorReason());
    digitalWrite(2, HIGH);  // turn the LED on (HIGH is the voltage level)
  delay(1000);                      // wait for a second
  digitalWrite(2, LOW);
  }

  // print user data or not found
}
//*Get the Fingerprint ID*****************
int getFingerprintID() {
  uint8_t p = finger.getImage();
  switch (p) {
    case FINGERPRINT_OK:
      //Serial.println("Image taken");
      break;
    case FINGERPRINT_NOFINGER:
      //Serial.println("No finger detected");
      return 0;
    case FINGERPRINT_PACKETRECIEVEERR:
      //Serial.println("Communication error");
      return -2;
    case FINGERPRINT_IMAGEFAIL:
      //Serial.println("Imaging error");
      return -2;
    default:
      //Serial.println("Unknown error");
      return -2;
  }
  // OK success!
  p = finger.image2Tz();
  switch (p) {
    case FINGERPRINT_OK:
      //Serial.println("Image converted");
      break;
    case FINGERPRINT_IMAGEMESS:
      //Serial.println("Image too messy");
      return -1;
    case FINGERPRINT_PACKETRECIEVEERR:
      //Serial.println("Communication error");
      return -2;
    case FINGERPRINT_FEATUREFAIL:
      //Serial.println("Could not find fingerprint features");
      return -2;
    case FINGERPRINT_INVALIDIMAGE:
      //Serial.println("Could not find fingerprint features");
      return -2;
    default:
      //Serial.println("Unknown error");
      return -2;
  }
  // OK converted!
  p = finger.fingerFastSearch();
  if (p == FINGERPRINT_OK) {
    //Serial.println("Found a print match!");
  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    //Serial.println("Communication error");
    return -2;
  } else if (p == FINGERPRINT_NOTFOUND) {
    //Serial.println("Did not find a match");
    return -1;
  } else {
    //Serial.println("Unknown error");
    return -2;
  }
  // found a match!
  Serial.print("Found ID #");
  Serial.print(finger.fingerID);
  Serial.print(" with confidence of ");
  Serial.println(finger.confidence);

  return finger.fingerID;
}
//*Check if there a Fingerprint ID to delete*****************
void ChecktoDeleteID() {
  Serial.println("Check to Delete ID");
  // digitalWrite(23, HIGH);  // turn the LED on (HIGH is the voltage level)
  // delay(1000);                      // wait for a second
  // digitalWrite(23, LOW);

  String path = "/Student Data/del";
  String path1 = "/Student Data/delval";
  int val, val1;

  if (Firebase.getInt(firebaseData, path)) {
    //Success
    Serial.print("Get int data success, int = ");
    val = firebaseData.intData();
    Serial.println(firebaseData.intData());


  } else {
    //Failed?, get the error reason from firebaseData

    Serial.print("Error in getInt, ");
    Serial.println(firebaseData.errorReason());
  }

  if (val == 1) {
    if (Firebase.getInt(firebaseData, path1)) {
      //Success
      Serial.print("Get delete value = ");
      val1 = firebaseData.intData();
      Serial.println(firebaseData.intData());
      deleteFingerprint(val1);
    }
  }
}
//*Delete Finpgerprint ID****************
uint8_t deleteFingerprint(int id) {
  uint8_t p = -1;

  p = finger.deleteModel(id);

  if (p == FINGERPRINT_OK) {
    Serial.println("Deleted!");

  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    Serial.println("Communication error");

    return p;
  } else if (p == FINGERPRINT_BADLOCATION) {
    Serial.println("Could not delete in that location");

    return p;
  } else if (p == FINGERPRINT_FLASHERR) {
    Serial.println("Error writing to flash");

    return p;
  } else {
    Serial.print("Unknown error: 0x");
    Serial.println(p, HEX);

    return p;
  }
}
//*Check if there a Fingerprint ID to add*****************
void ChecktoAddID() {
  Serial.println("Check to Add ID");
  //      id = add_id.toInt();

  int d;

  String path = "/Student Data/stat";
  String path2 = "/Student Data/id";

  if (Firebase.getInt(firebaseData, path)) {
    //Success


    d = firebaseData.intData();
    Serial.print("Data ..");
    Serial.println(firebaseData.intData());


    if (d == 1) {
      Firebase.getInt(firebaseData1, path2);
      id = firebaseData1.intData();
      Serial.println("Fetched id is: " + String(id));
      getFingerprintEnroll();
    }



  } else {
    //Failed?, get the error reason from firebaseData

    Serial.print("Error in setInt, ");
    Serial.println(firebaseData.errorReason());
  }
}
//*Enroll a Finpgerprint ID****************
uint8_t getFingerprintEnroll() {

  int p = -1;
  Serial.println("Enter your finger");
  while (p != FINGERPRINT_OK) {
    p = finger.getImage();
    switch (p) {
      case FINGERPRINT_OK:
        Serial.println("Image taken");

        break;
      case FINGERPRINT_NOFINGER:
        Serial.println(".");

        break;
      case FINGERPRINT_PACKETRECIEVEERR:
        Serial.println("I...");
        break;
      case FINGERPRINT_IMAGEFAIL:
        Serial.println("Imaging error");
        break;
      default:
        Serial.println("Unknown error");
        break;
    }
  }

  // OK success!

  p = finger.image2Tz(1);
  switch (p) {
    case FINGERPRINT_OK:

      break;
    case FINGERPRINT_IMAGEMESS:

      return p;
    case FINGERPRINT_PACKETRECIEVEERR:
      Serial.println("Communication error");
      return p;
    case FINGERPRINT_FEATUREFAIL:
      Serial.println("Could not find fingerprint features");
      return p;
    case FINGERPRINT_INVALIDIMAGE:
      Serial.println("Could not find fingerprint features");
      return p;
    default:
      Serial.println("Unknown error");
      return p;
  }

  Serial.println("Remove finger");
  delay(2000);
  p = 0;
  while (p != FINGERPRINT_NOFINGER) {
    p = finger.getImage();
  }
  Serial.print("ID ");
  Serial.println(id);
  p = -1;

  while (p != FINGERPRINT_OK) {
    p = finger.getImage();
    switch (p) {
      case FINGERPRINT_OK:
        Serial.println("Image taken");

        break;
      case FINGERPRINT_NOFINGER:
        Serial.println("Scanning....");

        break;
      case FINGERPRINT_PACKETRECIEVEERR:
        Serial.println("Communication error");
        break;
      case FINGERPRINT_IMAGEFAIL:
        Serial.println("Imaging error");
        break;
      default:
        Serial.println("Unknown error");
        break;
    }
  }

  // OK success!

  p = finger.image2Tz(2);
  switch (p) {
    case FINGERPRINT_OK:
      Serial.println("Image converted");

      break;
    case FINGERPRINT_IMAGEMESS:
      Serial.println("Image too messy");
      return p;
    case FINGERPRINT_PACKETRECIEVEERR:
      Serial.println("Communication error");
      return p;
    case FINGERPRINT_FEATUREFAIL:
      Serial.println("Could not find fingerprint features");
      return p;
    case FINGERPRINT_INVALIDIMAGE:
      Serial.println("Could not find fingerprint features");
      return p;
    default:
      Serial.println("Unknown error");
      return p;
  }

  // OK converted!
  Serial.print("Creating model for #");
  Serial.println(id);

  p = finger.createModel();
  if (p == FINGERPRINT_OK) {
    Serial.println("Prints matched!");

  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    Serial.println("Communication error");
    return p;
  } else if (p == FINGERPRINT_ENROLLMISMATCH) {
    Serial.println("Fingerprints did not match");
    return p;
  } else {
    Serial.println("Unknown error");
    return p;
  }

  Serial.print("ID ");
  Serial.println(id);
  p = finger.storeModel(id);
  if (p == FINGERPRINT_OK) {
    Serial.println("Stored!-->>Sending to Firebase");


    confirmAdding(id);
  } else if (p == FINGERPRINT_PACKETRECIEVEERR) {
    Serial.println("Communication error");
    return p;
  } else if (p == FINGERPRINT_BADLOCATION) {
    Serial.println("Could not store in that location");
    return p;
  } else if (p == FINGERPRINT_FLASHERR) {
    Serial.println("Error writing to flash");
    return p;
  } else {
    Serial.println("Unknown error");
    return p;
  }
}
//*Check if there a Fingerprint ID to add*****************
void confirmAdding(int id) {
  String td = ntp_time();

  Serial.println("Adding data to firebase....");
  String path = "/Student Data/" + String(id);
  String path2 = "/Student Data/stat";
  String path3 = "/Student Info/Data/" + String(id);
  String path4 = "/Student Info/Data/" + String(id)+"/inout";

  json.set("name", "0");
  json.set("ph", "0");
  json.set("email", "0");
  json.set("enrolltime", "0");

  json1.set("Intime",td);
  


  if (Firebase.updateNode(firebaseData, path3, json)) {
    Serial.println("PASSED");
    Serial.println("PATH: " + firebaseData.dataPath());
    Serial.print("PUSH NAME: ");
    Serial.println(firebaseData.pushName());
    Serial.println("ETag: " + firebaseData.ETag());
    Serial.println("------------------------------------");
    Serial.println();

    delay(300);

    if (Firebase.setInt(firebaseData2, path2, 0)) {
      Serial.println("Stat is updated");
    } else {
      Serial.println("Stat is not updated");
    }
    
    delay(300);
   
   if (Firebase.pushJSON(firebaseData2, path4, json1)) {
    Serial.println("PASSED");
    Serial.println("PATH: " + firebaseData.dataPath());
    Serial.print("PUSH NAME: ");
    Serial.println(firebaseData.pushName());
    Serial.println("ETag: " + firebaseData.ETag());
    Serial.println("------------------------------------");
    Serial.println();
   }
   else {
    Serial.println("FAILED");
    Serial.println("REASON: " + firebaseData2.errorReason());
  }





  } else {
    Serial.println("FAILED");
    Serial.println("REASON: " + firebaseData.errorReason());
  }
}


String ntp_time()
{
    timeClient.update();
    time_t epochTime = timeClient.getEpochTime();
    String formattedTime = timeClient.getFormattedTime();
    struct tm *ptm = gmtime ((time_t *)&epochTime);

    int monthDay = ptm->tm_mday;
    int currentMonth = ptm->tm_mon+1;
    int currentYear = ptm->tm_year+1900;
    String currentDate = String(monthDay)+"/"+String(currentMonth)+"/"+String(currentYear); 
    
    //Serial.println("Time: "+formattedTime);
    //Serial.println("Date: "+currentDate);

    String td = "D"+currentDate+"T"+formattedTime;
    return td;
}



//*connect to the WiFi*****************
void connectToWiFi() {
  WiFi.mode(WIFI_OFF);  //Prevents reconnection issue (taking too long to connect)
  delay(1000);
  WiFi.mode(WIFI_STA);
  Serial.print("Connecting to ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);


  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("Connected");



  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());  //IP address assigned to your ESP

  delay(1000);
}

