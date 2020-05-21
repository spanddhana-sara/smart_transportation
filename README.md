# smart_transportation
Code


#include <SPI.h>
#include <MFRC522.h>
#include <ESP8266WiFi.h>
#include <TinyGPS++.h>
#include <SoftwareSerial.h>
#include <FirebaseArduino.h>
#include <BlynkSimpleEsp8266.h>

#define DDEBUG true 

#define LED_G 16 //define green LED
#define LED_R 15 //define red LED
#define RST_PIN         0 
// Configurable, see typical pin layout above
#define SS_PIN          2           // Configurable, see typical pin layout above
#define GPS_TX_PIN      4
#define GPS_RX_PIN      5
#define FIREBASE_HOST "tejaswini-93a41.firebaseio.com"
#define FIREBASE_AUTH "Q8d5gtyg8wbdDziL3EZFOM8s976eEEAEudPRr82k"

const char* ssid    = "OnePlus 7 pro";
const char* password = "1pluswifi";
const String swipe_path = "/swipe/";
const String tracking_path = "/tracking/bus_1";

MFRC522 mfrc522(SS_PIN, RST_PIN);   // Create MFRC522 instance.
MFRC522::MIFARE_Key key;

SoftwareSerial ss(4, 5); // The serial connection to the GPS device

BlynkTimer timer;
TinyGPSPlus gps;

float latitude , longitude;

String flatX = "1000.0";
String flonY = "1000.0";
String fspd = "0.0";
String fsats = "0.0";
String fbear = "0.0";

static unsigned long lastRefreshTime = 0;
static const unsigned long REFRESH_INTERVAL = 10000; // ms
unsigned int move_index = 1;         // fixed index,

/*
 * 
 */
void send_to_firebase(String i_path, String i_data, bool i_set){
  FirebaseObject jsonObj((const char *)i_data.c_str());
  if(i_set == true){
    Firebase.set((const String&)i_path, (const JsonVariant&)jsonObj.getJsonVariant());
    if (Firebase.failed()) 
    {
        if (DDEBUG){
          Serial.println(Firebase.error());
        }
    }
  }
  else{
    if (Firebase.push((const String&)i_path, (const JsonVariant&)jsonObj.getJsonVariant()))
    {
        digitalWrite(LED_R, HIGH);
        delay(500);
        digitalWrite(LED_R, LOW);
        if (DDEBUG){
          Serial.println("Info sent");
        }
    }
    else
    {
        if (DDEBUG){
        Serial.println("Unable to upload.");
        }
    }
  }
}

/*
 * 
 */
 void send_data(String i_id, String i_lat, String i_long) { 
    
    
    
}

/*
 * 
 */
void wifi_setup(){
    if (DDEBUG){
      Serial.println();
      Serial.println();
      Serial.print("Connecting to ");
      Serial.println(ssid);
    }

    WiFi.begin(ssid, password);
    

    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        if (DDEBUG){
          Serial.print(".");
        }
    }
    
    if (DDEBUG){
      Serial.println("");
      Serial.println("WiFi connected");
      Serial.println("IP address: ");
      Serial.println(WiFi.localIP());
    }
}

/*
 * 
 */
void printHex(byte *buffer, byte bufferSize) {
  for (byte i = 0; i < bufferSize; i++) {
    Serial.print(buffer[i] < 0x10 ? " 0" : " ");
    Serial.print(buffer[i], HEX);
  }
}

/*
 * 
 */
void rfidInit()
{
  if (DDEBUG){
    Serial.println(F("Start RFID setup."));
  }
  SPI.begin(); 
  mfrc522.PCD_Init(); 
  for (byte i = 0; i < 6; i++) {
    key.keyByte[i] = 0xFF;
  }

  if (DDEBUG){
    Serial.println(F("This code scan the MIFARE Classsic NUID."));
    Serial.print(F("Using the following key:"));
    printHex(key.keyByte, MFRC522::MF_KEY_SIZE);
    Serial.println(F("End RFID setup."));
  }
}

/*
 * 
 */
String readRFID()
{
  String content= "";
  // Look for new cards
  if ( ! mfrc522.PICC_IsNewCardPresent()) 
  { 
    return content;
  }
  // Select one of the cards
  if ( ! mfrc522.PICC_ReadCardSerial()) 
  {
    return content;
  }

  MFRC522::PICC_Type piccType = mfrc522.PICC_GetType(mfrc522.uid.sak);
  if (DDEBUG){
    Serial.print(F("PICC type: "));
    Serial.println(mfrc522.PICC_GetTypeName(piccType));
  }
  
  // Check is the PICC of Classic MIFARE type
  if (piccType != MFRC522::PICC_TYPE_MIFARE_MINI &&  
    piccType != MFRC522::PICC_TYPE_MIFARE_1K &&
    piccType != MFRC522::PICC_TYPE_MIFARE_4K) {
    if (DDEBUG){
      Serial.println(F("Your tag is not of type MIFARE Classic."));
    }
    return content;
  }
  
  //Show UID on serial monitor
  if (DDEBUG){
    Serial.print("UID tag :");
  }
  byte letter;
  for (byte i = 0; i < mfrc522.uid.size; i++) 
  {
     if (DDEBUG){
     Serial.print(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " ");
     Serial.print(mfrc522.uid.uidByte[i], HEX);
     }
     content.concat(String(mfrc522.uid.uidByte[i] < 0x10 ? " 0" : " "));
     content.concat(String(mfrc522.uid.uidByte[i], HEX));
     digitalWrite(LED_G, HIGH);
     delay(500);
     digitalWrite(LED_G, LOW);
  } 
  // Halt PICC
  mfrc522.PICC_HaltA();
 
  // Stop encryption on PCD
  mfrc522.PCD_StopCrypto1();
  return content;
}
  
/*
 * 
 */
void cardReader(){
  String card_id = readRFID(); 
  if(card_id.length() != 0){
    if (DDEBUG){
    Serial.print("Sending card details ");
    Serial.print(flatX);
    Serial.print(", ");
    Serial.println(flonY);
    }
    String epoch = String(gps.date.year()) + "-" + String(gps.date.month()) +\
                  "-" + String(gps.date.day()) + " " + String(gps.time.hour()) +\
                  ":" + String(gps.time.minute()) + ":" + String(gps.time.second()) +\
                  ":" + String(gps.time.centisecond());
    String temp_data = String("{\"rfid\":\"")+ card_id + "\",\"lat\":\"" + flatX + "\",\"long\":\"" +\
                  flonY  +"\",\"ts\":\"" + epoch + "\"}";

    if (DDEBUG){
      Serial.println(temp_data); 
    }
    send_to_firebase(swipe_path, temp_data, false);
  }
}

/**
 * Initialize.
 */
void setup() {
  if (DDEBUG){
  Serial.begin(115200); 
  while (!Serial);     
  } 
  wifi_setup();
  ss.begin(9600);
  pinMode(LED_G, OUTPUT);
  pinMode(LED_R, OUTPUT);
  delay(1000);
  Firebase.begin(FIREBASE_HOST, FIREBASE_AUTH);
  rfidInit();
  timer.setInterval(500L, cardReader);
}

/**
 * Main loop.
 */
void loop() {
    while (ss.available() > 0)
    if (gps.encode(ss.read()))
    {
      if (gps.location.isValid())
      {
        latitude = gps.location.lat();
        flatX = String(latitude , 6);
        longitude = gps.location.lng();
        flonY = String(longitude , 6);
      }
  
      if(gps.speed.isValid())
      {
        fspd = String(gps.speed.kmph(), 2);
      }
  
      if(gps.satellites.isValid())
      {
        fsats = String(gps.satellites.value());
      }
  
      if(gps.course.isValid())
      {
        fbear = String(gps.course.deg(),6);
      }
    }

  if(millis() - lastRefreshTime >= REFRESH_INTERVAL)
  {
    if (DDEBUG){
      Serial.print(flatX);
      Serial.print(", ");
      Serial.print(flonY);
      Serial.print(", ");
      Serial.print(fspd);
      Serial.print(", ");
      Serial.println(fsats);
      Serial.println(String(gps.date.year()) + ":" + String(gps.date.month()) + ":" +\
                    String(gps.date.day()) + ":" + String(gps.time.hour()) + ":" + \
                    String(gps.time.minute()) + ":" + String(gps.time.second()) + ":" + \
                    String(gps.time.centisecond()));
     Serial.print("HEAP:: ");
     Serial.println(ESP.getFreeHeap());
    }

    if(flatX!="1000.0")
    {
      String temp_data = String("{\"lat\":\"")+ flatX + "\",\"long\":\"" + flonY + "\",\"speed\":\"" +\
                  fspd  + "\"}";
      send_to_firebase(tracking_path, temp_data, true);
    }
    lastRefreshTime += REFRESH_INTERVAL;
  }
  timer.run();
} 
