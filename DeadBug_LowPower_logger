//-------------------------------------------------------------------------------
/* This is a sketch for a One-Wire 2-sensor Type K thermocouple low-power temperature logger using the Arduino UNO R3 and 
DeadBug 2.0 shield developed by Dead Bug Prototypes: https://www.tindie.com/products/Dead_Bug_Prototypes/extreme-low
-power-data-logging-shield-for-arduino/*/
/*The temperature sampling interval is set to 30 seconds and the logger completely powers off in between samples. This logger
runs on a 9V Energizer Ultimate Lithium battery but a battery from 6-10V is compatible with the shield*/
// 2/21/2018
//-------------------------------------------------------------------------------
#include <Wire.h>
#include <SD.h>
#include <SPI.h>
#include <OneWire.h>
#include <DallasTemperature.h>

#define DS1337_ADDRESS      0x68
#define ONE_WIRE_BUS        2
#define SD_CHIPSELECT       8
#define POWEROFF            7

String strLogline = "";
float strTemp;

void SetAlarm() {
  Wire.beginTransmission(DS1337_ADDRESS);
  Wire.write((byte) 7);
  Wire.write(bin2bcd(29));     // Alarm 1 second
  Wire.write(0b10000000);      // Alarm 1 minute
  Wire.write(0b10000000);      // Alarm 1 timer (flag set)
  Wire.write(0b10000000);      // Alarm 1 date (flag set)
  Wire.write(0b10000000);      // Alarm 2 minute
  Wire.write(0b10000000);      // Alarm 2 timer (flag set)
  Wire.write(0b10000000);      // Alarm 2 date (flag set)
  Wire.write(0b00011111);      // turn off the oscillator and enable alarms 1 & 2
  Wire.write((byte) 0);        // Reset everything
  Wire.endTransmission();
}
	void GetDate() {
  Wire.begin(DS1337_ADDRESS);
  Wire.beginTransmission(DS1337_ADDRESS);
  Wire.write((byte) 0);	
  Wire.endTransmission();

  Wire.requestFrom(DS1337_ADDRESS, 7);
  uint8_t ss = bcd2bin(Wire.read());
  uint8_t mm = bcd2bin(Wire.read());
  uint8_t hh = bcd2bin(Wire.read());
  Wire.read();
  uint8_t d = bcd2bin(Wire.read());
  uint8_t m = bcd2bin(Wire.read());
  uint16_t y = bcd2bin(Wire.read()) + 2000;

  if (m < 10) strLogline += '0';
  strLogline += m;
  strLogline += '/';
  if (d < 10) strLogline += '0';
  strLogline += d;
  strLogline += '/';
  strLogline += y;
  strLogline += ',';
  if (hh < 10) strLogline += '0';
  strLogline += hh;
  strLogline += ':';
  if (mm < 10) strLogline += '0';
  strLogline += mm;
  strLogline += ':';
  if (ss < 10) strLogline += '0';
  strLogline += ss;
  strLogline += ',';
}
// utility functions
static uint8_t bcd2bin (uint8_t val) { 
  return val - 6 * (val >> 4); 
}
static uint8_t bin2bcd (uint8_t val) { 
  return val + 6 * (val / 10); 
}
       
void WriteSD() 
{
  if (!SD.begin(SD_CHIPSELECT)) {
    Serial.println("initialization failed! (No SD card, or not correctly mounted?)");
    return;
  }

// the logging file
File myFile;
 
  myFile = SD.open("DATALOG.CSV", FILE_WRITE);
  
  if (myFile) {
    if (!strLogline) {           				//This may be omitted because I was trying to create a header
    myFile.println("date,time,NucellaTemp,MusselTemp");
    }
    else {
    myFile.println(strLogline);
    myFile.close();
    }
    Serial.print("Writing: ");
    Serial.println(strLogline); 
  }
  else {
    Serial.println("Error writing to DATALOG.CSV");
  }
}
void getTemp() {
  // Setup a oneWire instance to communicate with any OneWire devices (not just Maxim/Dallas temperature ICs)
  OneWire oneWire(ONE_WIRE_BUS);
  // Pass our oneWire reference to Dallas Temperature. 
  DallasTemperature sensors(&oneWire);
  DeviceAddress addr;
  pinMode(ONE_WIRE_BUS, OUTPUT);
  sensors.begin();
  Serial.println();
  Serial.print("Number of Devices found on the bus = ");
  Serial.println(sensors.getDeviceCount()); 
  
  Serial.print("Requesting temperatures...");
  sensors.requestTemperatures(); // Send the command to get temperatures
  Serial.println("DONE");
  
  for (uint8_t s=0; s < sensors.getDeviceCount(); s++) {
  // get the unique address 
  sensors.getAddress(addr, s);
  // just look at bottom two bytes, which is pretty likely to be unique
  int smalladdr = (addr[6] << 8) | addr[7]; 
  
  strTemp = sensors.getTempCByIndex(s);
  Serial.print("Temperature for the device #"); Serial.print(s); 
  Serial.print(" with ID #"); Serial.print(smalladdr);
  Serial.print(" is: ");
  Serial.println(strTemp);
  
  strLogline += strTemp;
  strLogline += ',';
  }
}

void setup()
{
	Serial.begin(9600);
        pinMode(POWEROFF, OUTPUT);
        digitalWrite(POWEROFF, HIGH);
	delay(1000);
	GetDate();
	getTemp();
	WriteSD();
}

void loop()
{
	while (true)
	{
          SetAlarm();
          digitalWrite(POWEROFF, LOW);
	  delay(500);
	}
}
