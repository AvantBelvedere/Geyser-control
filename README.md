Geyser-control
==============

 /*
 A Web server that displays ON and OFF buttons in html page with Temperature to activate a Geyser,
 which is controlled by a DS18B20 temperature sensor. When target temperature is reached,
 the relay will disconect power. The LCD submits the current status of the relay and by which means
 the relay was activated or deactivated. Works with Arduino Uno and W5100 Arduino Wiznet Ethernet shield.

Circuit:
 * Ethernet shield attached to pins 10, 11, 12, 13
 * Button pins attached to A5 and GND
 * LCD 16 X 2 connected as follows:
   LCD Pin 14 to Digital Pin 2
   LCD     13                3
   LCD     12                4
   LCD     11                5
   LCD      6                6
   LCD      5                GND
   LCD      4                7
   LCD      3                10K Potentiometer Wiper / Centre
   LCD      2                10K Potentiometer side pin to +5V
   LCD      1                10K Potentiometer opposite side pin to GND
   
   *DS18B20 connected to Digital Pin 8

 created 16 July 2014
 by Rainbowfish
 
 */

#include <SPI.h>
#include <Ethernet.h>
#include <LiquidCrystal.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#define ONE_WIRE_BUS 8

byte mac[] = { 0xDE, 0xAD, 0xBE, 0xEF, 0xFE, 0xED };
IPAddress ip(10, 0, 0, 8);
EthernetServer server(80);

int relay1 = 9;
String state = "OFF";

LiquidCrystal lcd(7, 6, 5, 4, 3, 2);

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

int Button1 = A5;
int val = 0;

void setup()
{
  Serial.begin(9600);
  Ethernet.begin(mac, ip);
  server.begin();
  Serial.print("server IP is ");
  Serial.println(Ethernet.localIP());

  pinMode(relay1, OUTPUT);
  pinMode(Button1, INPUT);
  sensors.begin();
  lcd.begin(16, 2);
}

void loop()
{
  sensors.requestTemperatures();
  delay (1000);
  Serial.print("Temperature is: ");
  lcd.setCursor(0, 0);
  lcd.print(sensors.getTempCByIndex(0));
  lcd.print((char)223);
  lcd.print("C");
  Serial.print(sensors.getTempCByIndex(0));

  if (analogRead(Button1) == 0)

  {
    digitalWrite(relay1, HIGH);

    lcd.setCursor(0, 1);
    lcd.print("ON by Switch               ");
  }
  if (sensors.getTempCByIndex(0) > 17)

  {
    digitalWrite(relay1, LOW);
    lcd.setCursor(0, 1);
    lcd.print("OFF Temp Control               ");
  }

  EthernetClient client = server.available();

  if (client) {
    Serial.println("new client");
    boolean currentLineIsBlank = true;
    String string = "";
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        Serial.write(c);
        string.concat(c);
        
        int position = string.indexOf("LED=");

        if (string.substring(position) == "LED=ON")
        {
          digitalWrite(relay1, HIGH);
          state = "ON";
        }
        if (string.substring(position) == "LED=OFF")
        {
          digitalWrite(relay1, LOW);
          state = "OFF";
        }
        if (c == '\n' && currentLineIsBlank) {

          client.println("HTTP/1.1 200 OK");
          client.println("Content-Type: text/html");
          client.println("Connection: close");
          client.println();
          client.println("<html>");
          client.println("<head>");
          client.println("<title>Hot or Not</title>");
          client.println("</head>");
          client.println("<body>");
          client.println("<h1 align='center'>Home Control</h1><h3 align='center'>Geyser Temp</h3>");
          client.println("<div style='text-align:center;'><h2>");
          client.println(sensors.getTempCByIndex(0));
          client.println("&deg");
          client.println("C");
          client.println("<p>");
          client.println("<button onClick=location.href='./?LED=ON\' style='FONT-SIZE: 50px; HEIGHT: 200px; FONT-FAMILY: Arial; WIDTH: 325px; 126px; Z-INDEX: 0; TOP: 236px;'>");
          client.println("ON");
          client.println("</button>");
          client.println("<button onClick=location.href='./?LED=OFF\' style='FONT-SIZE: 50px; HEIGHT: 200px; FONT-FAMILY: Arial; WIDTH: 325px; 126px; Z-INDEX: 0; TOP: 236px;'>");
          client.println("OFF");
          client.println("</button>");
          client.println("<br /><br />");
          client.println("<b>Switch is ");
          client.print(state);
          client.println("</b><br />");
          client.println("</b></body>");
          client.println("</html>");

          {
            lcd.clear();
            lcd.setCursor(0, 1);
            lcd.print(state);
            lcd.print(" by WebSurfer");
          }
          break;
        }
        if (c == '\n') {
          currentLineIsBlank = true;
        }
        else if (c != '\r') {
          currentLineIsBlank = false;
        }
      }
    }
    delay(1);
    client.stop();
  }
}

