# TeMPtaTion


**********************************************************
*   This code is done for Educational Purposes Only      *
*   Coded by: Karl Tapawan                               *
*   For: Lyceum of the Philippines University - Cavite   *
*   Subject: Temperature Monitoring System               *
**********************************************************


**********************************************************
*   Gizduino ATMega328 , CC3000 , LM35                   *
**********************************************************


#include <Adafruit_CC3000.h>
#include <ccspi.h>
#include <SPI.h>
#include <string.h>
#include "utility/debug.h"

// Default Pin GizDuino CC3000
#define ADAFRUIT_CC3000_IRQ   3 
#define ADAFRUIT_CC3000_VBAT  5
#define ADAFRUIT_CC3000_CS    10
Adafruit_CC3000 cc3000 = Adafruit_CC3000(ADAFRUIT_CC3000_CS, ADAFRUIT_CC3000_IRQ, ADAFRUIT_CC3000_VBAT,
                                         SPI_CLOCK_DIV2); 
//Router Specifics
#define WLAN_SSID       "SSID"       
#define WLAN_PASS       "PASS"
//Router Security
#define WLAN_SECURITY   WLAN_SEC_WPA2

//Web transactions on port 80
char server[] = "tmm.site50.net";
Adafruit_CC3000_Client client;


void setup()
{

  Serial.begin(115200);
  Serial.println(F("Hello, Patient 1!\n")); 

  displayDriverMode();
  Serial.print("Free RAM: "); Serial.println(getFreeRam(), DEC);

  /* Initialise CC3000 */
  Serial.println(F("\nInitialising the CC3000 ..."));
  if (!cc3000.begin())
  {
    Serial.println(F("Unable to initialise the CC3000! Check your wiring?"));
    while(1);
  }

  /* Setting MAC Address Spoof */

  uint8_t macAddress[6] = { 0x08, 0x00, 0x28, 0x01, 0x79, 0xB7 };
   if (!cc3000.setMacAddress(macAddress))
   {
     Serial.println(F("Failed trying to update the MAC address"));
     while(1);
   }
  /* End MAC Spoof */

  uint16_t firmware = checkFirmwareVersion();
  if ((firmware != 0x113) && (firmware != 0x118)) {
    Serial.println(F("Wrong firmware version!"));
    for(;;);
  }

  displayMACAddress();

  /* Cleanup ng Past Connection */
  Serial.println(F("\nDeleting old connection profiles"));
  if (!cc3000.deleteProfiles()) {
    Serial.println(F("Failed!"));
    while(1);
  }

  /* Connecting to Wi-Fi Router */
  char *ssid = WLAN_SSID;            
  Serial.print(F("\nAttempting to connect to ")); Serial.println(ssid);

  /* NOTE: Secure connections are not available in 'Tiny' mode! */
  if (!cc3000.connectToAP(WLAN_SSID, WLAN_PASS, WLAN_SECURITY)) {
    Serial.println(F("Failed!"));
    while(1);
  }

  Serial.println(F("Connected!"));

  /* Wait for DHCP to complete */
  Serial.println(F("Request DHCP"));
  while (!cc3000.checkDHCP())
  {
    delay(100);
  }  

  /* Display the IP address DNS, Gateway, etc. */  
  while (! displayConnectionDetails()) {
    delay(1000);
  }

    }


void loop()
{
        uint32_t ip = 0;
        Serial.print(F("tmm.site50.net -> "));
        while  (ip  ==  0)  
        {
            if  (!  cc3000.getHostByName("tmm.site50.net", &ip))  
            {
                Serial.println(F("Couldn't resolve!"));
                while(1){}
            }
        }  
                cc3000.printIPdotsRev(ip);
                Serial.println(F(""));
                Serial.println(F("Posting!"));
                    Adafruit_CC3000_Client client = cc3000.connectTCP(ip, 80);
        if (client.connected()) 
        {
            {
              int x = analogRead(A1);
              float w = x * 0.48828125; 
              float tempPin= w + 1.5; 
                client.println("POST /sample1.php HTTP/1.1");
                Serial.println("POST /sample1.php HTTP/1.1");
                client.println("Host: tmm.site50.net");
                Serial.println("Host: tmm.site50.net");
                client.println("From: your@email.tld"); 
                Serial.println("From: your@email.tld");
                client.println("User-Agent: gizDuinodev1/1.0");
                Serial.println("User-Agent: gizDuinodev1/1.0");
                client.println("Content-Type: Application/x-www-form-urlencoded");
                Serial.println("Content-Type: Application/x-www-form-urlencoded");
                client.println("Content-Length: 50"); // notice not println to not get a linebreak
                Serial.println("Content-Length: 50");
                client.println("");               // to get the empty line
                Serial.println("");
                client.println("temp="); 
                Serial.println("temp=");
                client.print(tempPin);        
                Serial.print(tempPin);
                Serial.println("POSTED!");
                delay (1000);
                // While we're connected, print out anything the server sends:
                while (client.connected())
                    {
                    if (client.available())
                        {
                        char c = client.read();
                        Serial.print(c);
                        }
                    }
                        Serial.println();
            } 
        }
                    else // If the connection failed, print a message:
                    {
                        Serial.println(F("Connection failed"));
                    }
        delay(10000);
}


/**************************************************************************/
/*!
    @brief  Displays the driver mode (tiny of normal), and the buffer
            size if tiny mode is not being used

    @note   The buffer size and driver mode are defined in cc3000_common.h
*/
/**************************************************************************/
void displayDriverMode(void)
{
  #ifdef CC3000_TINY_DRIVER
    Serial.println(F("CC3000 is configure in 'Tiny' mode"));
  #else
    Serial.print(F("RX Buffer : "));
    Serial.print(CC3000_RX_BUFFER_SIZE);
    Serial.println(F(" bytes"));
    Serial.print(F("TX Buffer : "));
    Serial.print(CC3000_TX_BUFFER_SIZE);
    Serial.println(F(" bytes"));
  #endif
}

/**************************************************************************/
/*!
    @brief  Tries to read the CC3000's internal firmware patch ID
*/
/**************************************************************************/
uint16_t checkFirmwareVersion(void)
{
  uint8_t major, minor;
  uint16_t version;

#ifndef CC3000_TINY_DRIVER  
  if(!cc3000.getFirmwareVersion(&major, &minor))
  {
    Serial.println(F("Unable to retrieve the firmware version!\r\n"));
    version = 0;
  }
  else
  {
    Serial.print(F("Firmware V. : "));
    Serial.print(major); Serial.print(F(".")); Serial.println(minor);
    version = major; version <<= 8; version |= minor;
  }
#endif
  return version;
}

/**************************************************************************/
/*!
    @brief  Tries to read the 6-byte MAC address of the CC3000 module
*/
/**************************************************************************/
void displayMACAddress(void)
{
  uint8_t macAddress[6];

  if(!cc3000.getMacAddress(macAddress))
  {
    Serial.println(F("Unable to retrieve MAC Address!\r\n"));
  }
  else
  {
    Serial.print(F("MAC Address : "));
    cc3000.printHex((byte*)&macAddress, 6);
  }
}


/**************************************************************************/
/*!
    @brief  Tries to read the IP address and other connection details
*/
/**************************************************************************/
bool displayConnectionDetails(void)
{
  uint32_t ipAddress, netmask, gateway, dhcpserv, dnsserv;

  if(!cc3000.getIPAddress(&ipAddress, &netmask, &gateway, &dhcpserv, &dnsserv))
  {
    Serial.println(F("Unable to retrieve the IP Address!\r\n"));
    return false;
  }
  else
  {
    Serial.print(F("\nIP Addr: ")); cc3000.printIPdotsRev(ipAddress);
    Serial.print(F("\nNetmask: ")); cc3000.printIPdotsRev(netmask);
    Serial.print(F("\nGateway: ")); cc3000.printIPdotsRev(gateway);
    Serial.print(F("\nDHCPsrv: ")); cc3000.printIPdotsRev(dhcpserv);
    Serial.print(F("\nDNSserv: ")); cc3000.printIPdotsRev(dnsserv);
    Serial.println();
    return true;
  }
}
