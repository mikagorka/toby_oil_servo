# **BambulKab: Smart Oil Stove Control**

## **🔹 Why I Created This Project**
I built **BambulKab** to automatically control my **Koppe oil stove**, as it tends to **overheat during transitional seasons**. Instead of manually adjusting the oil regulator all the time, this project automates the process using a **Raspberry Pi, MQTT, and a servo motor**.

## **🔥 How My Modified Oil Stove Works**
My oil stove is **not a standard model**, as I have already made several modifications to it:
✅ **Ignition Rod (Glühstab):** Activates as soon as the oil regulator is turned above **0**. (It’s the black part in the image.)  
✅ **Thermostat-Controlled Heating:** The **thermostat** regulates whether the stove should stay in **low-flame mode** or heat up.  
❌ However, the thermostat **cannot turn the stove off** – that’s why I built this project!  

## **📌 My Solution**
This system **automates the oil regulator** using a **servo motor** and a **Raspberry Pi Zero 2 W**. The Raspberry Pi:
1. Controls the **servo motor** to turn the regulator on/off.
2. Uses **MQTT** for wireless control.
3. Works with **HomeKit** via **Homebridge** for easy automation.

---

## **🛠 Setting Up the Raspberry Pi**
Follow these steps to install everything required for **BambulKab**.

### **1️⃣ Flash Raspberry Pi OS & Update**
1. Flash **Raspberry Pi OS Lite** onto an SD card.
2. Boot up and run:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

### **2️⃣ Install Required Packages**
```bash
sudo apt install -y mosquitto mosquitto-clients python3-pip
pip3 install paho-mqtt pigpio
```
Start the **MQTT broker**:
```bash
sudo systemctl enable mosquitto
sudo systemctl restart mosquitto
```

### **3️⃣ Configure MQTT**
Edit the config file:
```bash
sudo nano /etc/mosquitto/mosquitto.conf
```
Add:
```ini
listener 1883
allow_anonymous true
log_dest syslog
log_dest stdout
persistence false
```
Restart Mosquitto:
```bash
sudo systemctl restart mosquitto
```

### **4️⃣ Install Servo Control with MQTT**
Create the script:
```bash
nano ~/oelregler/mqtt_servo.py
```
Paste this:
```python
import pigpio
import time
import paho.mqtt.client as mqtt

SERVO_PIN = 18
SPEED = 0.01  # Adjust this for smooth movement

pi = pigpio.pi()
if not pi.connected:
    print("Error: pigpio not connected!")
    exit()

def set_angle(angle):
    pulse_width = 500 + (angle / 180.0) * 2000
    pi.set_servo_pulsewidth(SERVO_PIN, pulse_width)

def on_message(client, userdata, msg):
    payload = msg.payload.decode()
    if payload == "ON":
        print("Turning oil regulator ON (0°)")
        for angle in range(180, -1, -1):
            set_angle(angle)
            time.sleep(SPEED)
    elif payload == "OFF":
        print("Turning oil regulator OFF (180°)")
        for angle in range(0, 181, 1):
            set_angle(angle)
            time.sleep(SPEED)
    
    pi.set_servo_pulsewidth(SERVO_PIN, 0)

client = mqtt.Client()
client.on_message = on_message
client.connect("localhost", 1883, 60)
client.subscribe("oelregler/set")

print("Listening for MQTT messages...")
client.loop_forever()
```
Save & exit (`CTRL + X`, `Y`, `Enter`).

### **5️⃣ Set Up MQTT as a Service**
```bash
sudo nano /etc/systemd/system/mqtt_servo.service
```
Paste this:
```ini
[Unit]
Description=MQTT Servo Control
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/pi/oelregler/mqtt_servo.py
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```
Save & enable:
```bash
sudo systemctl daemon-reload
sudo systemctl enable mqtt_servo
sudo systemctl start mqtt_servo
```

---

## **🏠 HomeKit Integration via Homebridge**
If you have **Homebridge running** on another Raspberry Pi (e.g., a Pi 5), install the **homebridge-mqttthing** plugin:
```bash
sudo npm install -g homebridge-mqttthing
```
Modify your **Homebridge `config.json`**:
```json
{
  "accessories": [
    {
      "accessory": "mqttthing",
      "type": "switch",
      "name": "Oil Stove Control",
      "url": "mqtt://<YOUR-RASPBERRY-IP>",
      "topics": {
        "getOn": "oelregler/status",
        "setOn": "oelregler/set"
      },
      "onValue": "ON",
      "offValue": "OFF"
    }
  ]
}
```
Restart Homebridge:
```bash
sudo systemctl restart homebridge
```
Now you can **control your oil stove via HomeKit & Siri**! 🔥  

---

## **🖨️ 3D-Printed Parts**
These **3D-printed components** fit perfectly for my **Koppe oil stove**.  
- I used **these magnets** for better attachment:  
  👉 [Amazon Link](https://www.amazon.de/dp/B088FN6YN6?ref_=ppx_hzsearch_conn_dt_b_fed_asin_title_1&th=1)  
- **M3 inserts for servo mounting** (pressed in with a soldering iron).  
- Servo motor:  
  👉 [Amazon Link](https://www.amazon.de/dp/B07H87592P?ref=ppx_yo2ov_dt_b_fed_asin_title)  

📌 **I will upload the STL files on MakerWorld soon.**  

---

## **🔗 Links & Future Plans**
✅ **GitHub Repository:** _(Coming soon)_  
✅ **MakerWorld STL Files:** _(Coming soon)_  
✅ **YouTube Demo:** 👉 [Watch the video](https://youtube.com/shorts/KCvpp14In0I?feature=share)  

🚀 **Enjoy automating your stove – because manual control is outdated!** 😎🔥
