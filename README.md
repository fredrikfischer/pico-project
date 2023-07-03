import network
from machine import Pin
import time
import utime
import dht
import ujson
import urequests
import umqtt.robust as mqtt

# Fill in your network name (SSID) and password here:
ssid = "MYWIFI"
password = "MYPASSWORD"

# Adafruit MQTT configuration
adafruit_mqtt_host = "io.adafruit.com"
adafruit_mqtt_port = 1883
adafruit_username = "firreb"
adafruit_key = "MYKEY"


def connect_wifi():
    # Connect to WLAN
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(ssid, password)
    while not wlan.isconnected():
        print('Waiting for connection...')
        time.sleep(1)
    print(wlan.ifconfig())

try:
    connect_wifi()
except KeyboardInterrupt:
    machine.reset()

relay_pin = Pin(16, Pin.OUT)
sensor = dht.DHT11(Pin(15))

def send_adafruit_mqtt_data(feed_name, value):
    client = mqtt.MQTTClient("pico", adafruit_mqtt_host, port=adafruit_mqtt_port,
                            user=adafruit_username, password=adafruit_key, ssl=False)
    client.connect()

    topic = "{}/feeds/{}/json".format(adafruit_username, feed_name)
    payload = ujson.dumps({"value": value})
    client.publish(topic, payload)

    client.disconnect()
    print("Data sent to Adafruit MQTT")
       
def get_temperature_threshold():
    api_url = "https://io.adafruit.com/api/v2/firreb/feeds/temperature-threshold"
    headers = {"X-AIO-Key": adafruit_key}
    response = urequests.get(api_url, headers=headers)
    data = ujson.loads(response.text)
    return int(data["last_value"])
    
while True:    
    sensor.measure()
    temp = sensor.temperature()
    humidity = sensor.humidity()

    temperature_threshold_value = get_temperature_threshold()
    print("Temperature threshold value: " + str(temperature_threshold_value))
    # Control relay based on Adafruit value
    if temperature_threshold_value < temp:
        relay_pin.value(0)  # Switch on the relay
        print("Fan on")
    else:
        relay_pin.value(1)  # Switch off the relay
        print("Fan off")
    
    send_adafruit_mqtt_data("humidity-feed", humidity)
    send_adafruit_mqtt_data("temperature-feed", temp)

    time.sleep(10)  # Delay for 1 second before the next iteration

