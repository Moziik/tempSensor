# tempSensor
# Dallas ds18x20 temperature sensor
# MicroPython Esp32 with Dallas ds18B20 tempreature sensor

import machine, onewire, ds18x20, time
from machine import Pin, ADC,SoftI2C
from ssd1306 import SSD1306_I2C
from time import sleep

class DS18X20:
    def __init__(self, onewire):
        self.ow = onewire
        self.buf = bytearray(9)
        self.config = bytearray(3)
        self.power = 1 # strong power supply by default
        self.powerpin = None

    def powermode(self, powerpin=None):
        if self.powerpin is not None: # deassert strong pull-up
           self.powerpin(PULLUP_OFF)
           self.ow.writebyte(self.ow.CMD_SKIPROM)
           self.ow.writebyte(CMD_RDPOWER)
           self.power = self.ow.readbit()
        if powerpin is not None:
            assert type(powerpin) is Pin, "Parameter must be a Pin object"
            self.powerpin = powerpin
            self.powerpin.init(mode=Pin.OUT, value=0)
        return self.power

    def scan(self):
        if self.powerpin is not None: # deassert strong pull-up
            self.powerpin(PULLUP_OFF)
        return [rom for rom in self.ow.scan() if rom[0] in (0x10, 0x22, 0x28)]

    def convert_temp(self, rom=None):
        if self.powerpin is not None: # deassert strong pull-up
            self.powerpin(PULLUP_OFF)
        self.ow.reset()
        if rom is None:
            self.ow.writebyte(self.ow.CMD_SKIPROM)
        else:
            self.ow.select_rom(rom)
        self.ow.writebyte(CMD_CONVERT, self.powerpin)

    def read_scratch(self, rom):
        if self.powerpin is not None: # deassert strong pull-up
            self.powerpin(PULLUP_OFF)
        self.ow.reset()
        self.ow.select_rom(rom)
        self.ow.writebyte(CMD_RDSCRATCH)
        self.ow.readinto(self.buf)
        assert self.ow.crc8(self.buf) == 0, 'CRC error'
        return self.buf

    def write_scratch(self, rom, buf):
        if self.powerpin is not None: # deassert strong pull-up
            self.powerpin(PULLUP_OFF)
        self.ow.reset()
        self.ow.select_rom(rom)
        self.ow.writebyte(CMD_WRSCRATCH)
        self.ow.write(buf)

    def read_temp(self, rom):
        try:
            buf = self.read_scratch(rom)
            if rom[0] == 0x10:
                if buf[1]:
                    t = buf[0] >> 1 | 0x80
                    t = -((~t + 1) & 0xff)
                else:
                    t = buf[0] >> 1
                return t - 0.25 + (buf[7] - buf[6]) / buf[7]
            elif rom[0] in (0x22, 0x28):
                t = buf[1] << 8 | buf[0]
                if t & 0x8000: # sign bit set
                    t = -((t ^ 0xffff) + 1)
                return t / 16
            else:
                return None
        except AssertionError:
            return None

    def resolution(self, rom, bits=None):
        if bits is not None and 9 <= bits <= 12:
            self.config[2] = ((bits - 9) << 5) | 0x1f
            self.write_scratch(rom, self.config)
            return bits
        else:
            data = self.read_scratch(rom)
            return ((data[4] >> 5) & 0x03) + 9

    def fahrenheit(self, celsius):
        return celsius * 1.8 + 32 if celsius is not None else None

    def kelvin(self, celsius):
        return celsius + 273.15 if celsius is not None else None

i2c = SoftI2C(scl=Pin(4), sda=Pin(5))    #Init i2c
oled_width = 128
oled_height = 64
oled = SSD1306_I2C(128, 64, i2c)
pot = machine.Pin(2)
ds_sensor = ds18x20.DS18X20(onewire.OneWire(pot)) # create the onewire object
roms = ds_sensor.scan()# scan for devices on the bus
print('Found DS devices: ', roms) 
try:
  while True:
      ds_sensor.convert_temp()
      time.sleep_ms(750)    #The reading temperature needs at least 750ms
      for rom in roms:
          oled.fill(0)
          oled.text("Temperature:",10,16)
          oled.text(str(ds_sensor.read_temp(rom)),24,40)
          oled.show()  
except KeyboardInterrupt:
          pass

