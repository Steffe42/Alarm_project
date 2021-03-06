**Koden**
Här ska koden som används i projektet presenteras och beskrivas kort. 
**Biblotek**
Vi använder oss av en map där vi förvarar vår main-fil och MQTT-mappen. 

**Main**
Inleder koden med Import och deklarationer
```py
from mqtt import MQTTClient
import network
from network import WLAN
import machine
from machine import Pin
import time
import pycom
from machine import PWM

pir_pin = Pin("P20", mode=Pin.IN, pull = Pin.PULL_DOWN)
button_pin = Pin('P6', mode=Pin.IN, pull=None)
buzzer = Pin("P7", mode=Pin.OUT)
adc = machine.ADC()   
temp_pin = adc.channel(pin="P16")
tim = PWM(0, frequency=300)
ch = tim.channel(2, duty_cycle=0, pin=buzzer)
prev = time.ticks_ms()
```
Denna delen av koden innehåller nödvändiga imports och deklarationer. Import är viktigt för att t.ex MQTT ska kunna sammanspela med main-filen, medans deklarationerna ger oss verktyget att programmera komponenterna att göra vad vi vill. 

Nästa steg av main är alla funktioner

```py
def sub_cb(topic, msg):
    message = str(msg)
    if message == "b'ON'":
        print(message)

def pin_handler():
    print("Motion has been detected")

def alarm():
    for i in range(6):
        ch = tim.channel(2, duty_cycle=0.8, pin=buzzer)
        time.sleep(0.25)
        ch = tim.channel(2, duty_cycle=0, pin=buzzer)
        time.sleep(0.25)

def buttonEventCallback(argument):
    global alarm_deactivate
    global prev

    now = time.ticks_ms()
    since = now - prev

    if since > 1000:
        alarm_deactivate = True
        prev = now
        print("Alarm is deactivated")
    else:
        print("Button press was ignored try agian")
```
def alarm - Denna funktionen används för att framkalla ljud från "buzzern". Den anropas längre ner i main-filen. 
def buttonEventCallback - Knappfunktionens syfte är att förhindra felaktiga "tryckvärden" och för att meddela när knappen är tryckt.

Denna del av koden börjar med att få internet uppkoppling och kopplar upp sig till MQTT-servern. 

```py
wlan = network.WLAN(mode=network.WLAN.STA)
wlan.connect('', auth=(network.WLAN.WPA2, ''))
while not wlan.isconnected():
    time.sleep_ms(50)
print(wlan.ifconfig())

client = MQTTClient("Oscar_p", "io.adafruit.com", user="Steffe42", password="aio_Tfiu38NjW3axyWwezxNWc2JYNmHF", port=1883)
client.set_callback(sub_cb)
client.connect()
print("connected to mqtt")

button_pin.callback(Pin.IRQ_FALLING, buttonEventCallback)
alarm_deactivate = False
```

Sista delen av main är en while sats som är "hjärnan" i koden. 
```py
while True:
    millivolts = temp_pin.voltage()
    temp_celcius = (millivolts - 500.0) / 10.0

    pir_move = pir_pin()

    if alarm_deactivate:
        print("idle")
        client.publish(topic="Steffe42/feeds/Alarm-status", msg="Button have been pressed and alarm is now idle!")
        machine.idle()
        print("idle")
        time.sleep(20)
        alarm_deactivate = False
        print("alarm is active")

    elif pir_move or temp_celcius > 40:
        alarm()
        client.publish(topic="Steffe42/feeds/Alarm-activated", msg="Your alarm have been activated call 112!!!")
        print("ALARM")
        time.sleep(20)

    else:
        client.publish(topic="Steffe42/feeds/Alarm-status", msg="Alarm is ok and waiting!")
        #print("Alarm is ok")
        client.publish(topic="Steffe42/feeds/Alarm-activated", msg="Alarm is ok and waiting!")
        print("Alarm is ok")
        time.sleep(20)

pir.callback(Pin.IRQ_FALLING, pin_handler)
```
Inuti while-loopen finns det tre nyckelsteg:
Första If-satsen behandlar knapptryck, skickar till servern att larmet är avstängt. 
I elif-satsen kollas det om Pir sensorn eller temperatur sensorn har aktiverats, och skickar ett meddelande till MQTT.
I else väntar den på någon sorts input, och printar att allt är okej tills något händer.
