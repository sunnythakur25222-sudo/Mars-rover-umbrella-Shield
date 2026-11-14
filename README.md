import time
import random
from enum import Enum, auto

# ============================================================
# CONFIGURATION & THRESHOLDS
# ============================================================

DUST_HIGH_THRESHOLD  = 0.50      
DUST_CLEAR_THRESHOLD = 0.20      
LIGHT_NORMAL_LUX     = 900       

MOTOR_RUN_TIME       = 2.8       
POLL_DELAY           = 1.0       

# ============================================================
# HARDWARE CLASSES
# ============================================================

class DustSensor:
    def __init__(self):
        self.voltage = 0.15

    def read(self):
        return max(0.05, self.voltage + random.uniform(-0.03, 0.03))

    def set_dusty(self, dusty: bool):
        self.voltage = 0.75 if dusty else 0.15


class LightSensor:
    def __init__(self):
        self.lux = 1300

    def read(self):
        return self.lux + random.uniform(-40, 40)

    def set_storm(self, storm: bool):
        self.lux = 150 if storm else 1300


class Camera:
    def __init__(self):
        self.storm_detected = False

    def is_storm(self):
        return self.storm_detected

    def set_storm(self, active: bool):
        self.storm_detected = active


class Actuators:
    def __init__(self):
        self.position = "RETRACTED"

    def extend(self):
        if self.position == "EXTENDED":
            return
        time.sleep(MOTOR_RUN_TIME)
        self.position = "EXTENDED"

    def retract(self):
        if self.position == "RETRACTED":
            return
        time.sleep(MOTOR_RUN_TIME)
        self.position = "RETRACTED"

# ============================================================
# MAIN SYSTEM LOGIC
# ============================================================

class State(Enum):
    MONITOR    = auto()
    DEPLOYED   = auto()

class UmbrellaSystem:
    def __init__(self):
        self.dust      = DustSensor()
        self.light     = LightSensor()
        self.camera    = Camera()
        self.actuators = Actuators()
        self.state     = State.MONITOR
        self.last_dust = 0.0
        self.last_lux  = 0.0

    def print_status(self, cycle):
        storm = "YES" if self.camera.is_storm() else "no "
        print(f"[{cycle:02d}] {self.state.name:8} | Dust={self.last_dust:.3f}V | Light={self.last_lux:5.0f} lux | Storm={storm}")

    def run(self):
        for cycle in range(1, 17):
            if cycle == 5:
                self.camera.set_storm(True)
                self.dust.set_dusty(True)
                self.light.set_storm(True)

            if cycle == 11:
                self.camera.set_storm(False)
                self.dust.set_dusty(False)
                self.light.set_storm(False)

            self.last_dust = self.dust.read()
            self.last_lux  = self.light.read()
            self.print_status(cycle)

            if self.state == State.MONITOR:
                if self.camera.is_storm() and self.last_dust >= DUST_HIGH_THRESHOLD:
                    self.actuators.extend()
                    self.state = State.DEPLOYED

            elif self.state == State.DEPLOYED:
                if self.last_dust <= DUST_CLEAR_THRESHOLD and self.last_lux >= LIGHT_NORMAL_LUX:
                    self.actuators.retract()
                    self.state = State.MONITOR

            time.sleep(POLL_DELAY)

if __name__ == "__main__":
    system = UmbrellaSystem()
    system.run()
