# Strike-Detector-IoT-Midterm
Strike Detector Midterm Codes
Python and Arduino IDE



```python
import socket
import math
import time

PORT = 5005

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(("0.0.0.0", PORT))

print("Listening", PORT)

ACC_THRESHOLD_G = 2.5
GYRO_THRESHOLD_DPS = 325.0
COOLDOWN_S = 0.7

RECORD_DURATION = 30
last_strike = 0.0

filename = f"punch_log_{int(time.time())}.text"
log = open(filename, "w")
print(f"RECORDING PUNCHES for {RECORD_DURATION} seconds")
print(f"Logging to {filename}")

start_time = time.time()
end_time = start_time + RECORD_DURATION
jab_count = 0
hook_count = 0

while time.time() < end_time:
    try:
        data, addr = sock.recvfrom(4096)
    except socket.timeout:
        continue

    parts = data.decode(errors="ignore").strip().split(",")
    if len(parts) != 8:
        continue

    seq = int(parts[0])
    t_ms = int(parts[1])
    ax = int(parts[2]); ay = int(parts[3]); az = int(parts[4])
    gx = int(parts[5]); gy = int(parts[6]); gz = int(parts[7])

    ax_g = ax / 16384.0
    ay_g = ay / 16384.0
    az_g = az / 16384.0

    gx_dps = gx / 131.0
    gy_dps = gy / 131.0
    gz_dps = gz / 131.0

    a_mag = math.sqrt(ax_g*ax_g + ay_g*ay_g + az_g*az_g)
    g_mag = math.sqrt(gx_dps*gx_dps + gy_dps*gy_dps + gz_dps*gz_dps)

    impulse = abs(a_mag - 1.0)

    now = time.time()
    if ((now - last_strike) >= COOLDOWN_S) and (impulse >= ACC_THRESHOLD_G or g_mag >= GYRO_THRESHOLD_DPS):
        last_strike = now

        abs_ax = abs(ax_g)
        abs_ay = abs(ay_g)
        abs_az = abs(az_g)

        p1, p2, p3 = sorted([abs_ax, abs_ay, abs_az], reverse=True)
        if (p2/p1) >= 0.7 and p2 >= 1.0:
            strike_type = "HOOK"
        else:
            strike_type = "JAB"

        rel_time = now - start_time
        line = f"{rel_time:.3f}s {strike_type}"
        print(line.strip())
        log.write(line)

log.close()
print("Log Complete")
print(f"Saved to:{filename}")

```arduino
#include <WiFi.h>
#include <WiFiUdp.h>
#include <Wire.h>
#include <MPU6050.h>

const char* ssid = "myphone";
const char* password = "mypassword";

// Put your Raspberry Pi hotspot IP here:
IPAddress piIP(172, 20, 10, 10);
const uint16_t piPort = 5005;

// Confirmed I2C pins
#define SDA_PIN 18
#define SCL_PIN 9

WiFiUDP udp;
MPU6050 mpu;

uint32_t seq = 0;

void setup() {
  Serial.begin(115200);
  delay(1000);

  // Wi-Fi
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected");
  Serial.print("ESP32 IP: ");
  Serial.println(WiFi.localIP());

  // I2C + MPU
  Wire.begin(SDA_PIN, SCL_PIN);
  Wire.setClock(100000);  // safe start

  Serial.println("Initializing MPU6050...");
  mpu.initialize();

  if (!mpu.testConnection()) {
    Serial.println("MPU6050 connection failed. Check wiring/pins/power.");
    while (true) delay(1000);
  }
  Serial.println("MPU6050 connected!");
}

void loop() {
  int16_t ax, ay, az, gx, gy, gz;
  mpu.getMotion6(&ax, &ay, &az, &gx, &gy, &gz);

  uint32_t t = millis();

  // CSV: seq,t,ax,ay,az,gx,gy,gz
  char msg[160];
  snprintf(msg, sizeof(msg), "%lu,%lu,%d,%d,%d,%d,%d,%d",
           (unsigned long)seq++,
           (unsigned long)t,
           ax, ay, az, gx, gy, gz);

  udp.beginPacket(piIP, piPort);
  udp.print(msg);
  udp.endPacket();

  delay(10); // ~100 Hz
}
