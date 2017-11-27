# Robo-mussel
Arduino code for a simple one-wire Type-K thermocouple temperature logger

#include <OneWire.h>
#include <DallasTemperature.h>
#include <SPI.h>
#include <SD.h>
#include <Wire.h>
#include "RTClib.h"

// Data wire is plugged into port 2 on the Arduino
#define ONE_WIRE_BUS 2

// Setup a oneWire instance to communicate with any OneWire devices (not just Maxim/Dallas temperature ICs)
OneWire oneWire(ONE_WIRE_BUS);

// Pass our oneWire reference to Dallas Temperature. 
DallasTemperature sensors(&oneWire);
DeviceAddress addr;

//SD card stuff
// A simple data logger for the Arduino digital pins

// how many milliseconds between grabbing data and logging it. 1000 ms is once a second
#define LOG_INTERVAL  30000 // mills between entries (reduce to take more/faster data)

// how many milliseconds before writing the logged data permanently to disk
// set it to the LOG_INTERVAL to write each time (safest)
// set it to 10*LOG_INTERVAL to write all data every 10 datareads, you could lose up to 
// the last 10 reads if power is lost but it uses less power and is much faster!
#define SYNC_INTERVAL 300000 // mills between calls to flush() - to write data to the card
uint32_t syncTime = 0; // time of last sync()

#define ECHO_TO_SERIAL   1 // echo data to serial port
#define WAIT_TO_START    0 // Wait for serial input in setup()

// The digital pin that connects (via 1-wire) to the sensors
#define tempPin 2                // digital pin 2

float temperatureC;

RTC_PCF8523 RTC; // define the Real Time Clock object

// for the data logging shield, we use digital pin 10 for the SD cs line
const int chipSelect = 10;

// the logging file
File logfile;
// the name of the logging file
char *filename;

void error(char *str)
{
  Serial.print("error: ");
  Serial.println(str);
}
  
void setup() {
  // put your setup code here, to run once:
  // start serial port

  pinMode(2, INPUT)
  
 ; Serial.begin(9600);
  Serial.println("Dallas Temperature IC Control Library Demo");

  // Start up the library
  sensors.begin();
  Serial.begin(9600);
  Serial.println();
  Serial.print("Number of Devices found on the bus = ");
  Serial.println(sensors.getDeviceCount());               //telling us how many sensors we have connected to the Arduino
  
  
#if WAIT_TO_START
  Serial.println("Type any character to start");
  while (!Serial.available());
#endif //WAIT_TO_START

  // initialize the SD card
  Serial.print("Initializing SD card...");
  // make sure that the default chip select pin is set to
  // output, even if you don't use it:
  pinMode(10, OUTPUT);
  
  // see if the card is present and can be initialized:
  if (!SD.begin(chipSelect)) {
    error("Card failed, or not present");
  }
  Serial.println("card initialized.");
  
  // create a new file
  filename = "LOGGER00.CSV";
  for (uint8_t i = 0; i < 100; i++) {
    filename[6] = i/10 + '0';
    filename[7] = i%10 + '0';
    if (! SD.exists(filename)) {
      // only open a new file if it doesn't exist
      logfile = SD.open(filename, FILE_WRITE); 
      break;  // leave the loop!
    }
  }
  
  if (! logfile) {
    error("couldnt create file");
  }
  
  Serial.print("Logging to: ");
  Serial.println(filename);

  // connect to RTC
  Wire.begin();  
  if (!RTC.begin()) {
    logfile.println("RTC failed");
#if ECHO_TO_SERIAL
    Serial.println("RTC failed");
#endif  //ECHO_TO_SERIAL
  }
  RTC.adjust(DateTime(F(__DATE__), F(__TIME__)));

  logfile.println("millis,date,starttime,tempDevice0,tempDevice1");    
#if ECHO_TO_SERIAL
  Serial.println("millis,date,starttime,tempDevice0,tempDevice1");
#endif //ECHO_TO_SERIAL
logfile.close();

Serial.println("done.");
}

void loop() {
// put your main code here, to run repeatedly:
  //SD card code
  // re-open the file for reading:
  logfile = SD.open(filename, FILE_WRITE);
  if (logfile) {
  // log milliseconds since starting
  uint32_t m = millis();
  logfile.print(m);           // milliseconds since start
  logfile.print(", ");    
#if ECHO_TO_SERIAL
  Serial.print(m);         // milliseconds since start
  Serial.print(", "); 
#endif

  // following line sets the RTC to the date & time this sketch was compiled
 DateTime now = RTC.now();
 
  // log time
  logfile.print(now.month(), DEC);
  logfile.print("/");
  logfile.print(now.day(), DEC);
  logfile.print("/");
  logfile.print(now.year(), DEC);
  logfile.print(",");
  logfile.print(now.hour(), DEC);
  logfile.print(":");
  logfile.print(now.minute(), DEC);
  logfile.print(":");
  logfile.print(now.second(), DEC);
  logfile.print('"');
#if ECHO_TO_SERIAL
  Serial.print(now.month(), DEC);
  Serial.print("/");
  Serial.print(now.day(), DEC);
  Serial.print("/");
  Serial.print(now.year(), DEC);
  Serial.print(",");
  Serial.print(now.hour(), DEC);
  Serial.print(":");
  Serial.print(now.minute(), DEC);
  Serial.print(":");
  Serial.print(now.second(), DEC);
  Serial.print('"');
#endif //ECHO_TO_SERIAL 
// call sensors.requestTemperatures() to issue a global temperature 
  // request to all devices on the bus
  Serial.print("Requesting temperatures...");
  sensors.requestTemperatures(); // Send the command to get temperatures
  Serial.println("DONE");

  for (uint8_t s=0; s < sensors.getDeviceCount(); s++) {
    // get the unique address 
    sensors.getAddress(addr, s);
    // just look at bottom two bytes, which is pretty likely to be unique
    int smalladdr = (addr[6] << 8) | addr[7];

    temperatureC = sensors.getTempCByIndex(s);
    Serial.print("Temperature for the device #"); Serial.print(s); 
    Serial.print(" with ID #"); Serial.print(smalladdr);
    Serial.print(" is: ");
    Serial.println(temperatureC);
    logfile.print(", ");    
    logfile.print(temperatureC);
  }
 
  logfile.println();
#if ECHO_TO_SERIAL
  Serial.println();
#endif // ECHO_TO_SERIAL

      // close the file:
    logfile.close();
  }  
  else {
    // if the file didn't open, print an error:
    Serial.print("error opening ");
    Serial.println(filename);
  }
  // delay for the amount of time we want between readings
  delay((LOG_INTERVAL -1) - (millis() % LOG_INTERVAL)); //logs data to the SD card every 10 data reads
}
