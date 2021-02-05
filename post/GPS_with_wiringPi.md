# How to use GPS <small>(feat. wiringPi)</small> #

### Environment ###
* Odroid XU4
* Goouuu Tech GT-U7 GPS
* `wiringPi` python wrapper -> [odroid xu4 wiringPi](https://wiki.odroid.com/odroid-xu4/application_note/gpio/wiringpi)

<br>

### Interface ###
* Use UART pins in GPIOs

<br>

### Getting data ###
##### code example #####
```python
import odroid_wiringpi as wpi

UART_DEV = "/dev/ttySAC0"
BAUDRATE = 9600

fd = wpi.serialOpen(UART_DEV, BAUDRATE)
wpi.wiringPiSetup()

try:
    while(1):
        if wpi.serialDataAvail(fd) :
            data = wpi.serialGetchar(fd)
            print(chr(data), end='')

except Exception as e:
    print(e)
```

##### The result may look like below, #####
```
$GNGGA,080831.000,3724.73827,N,12705.70917,E,1,06,1.3,121.4,M,0.0,M,,*70
$GNRMC,080831.000,A,3724.73827,N,12705.70917,E,0.94,150.53,050221,,,A*7B
```

<br>

### Parsing ###
##### In `GGA` #####
```python
arr = "$GNGGA,080731.000,3724.73827,N,12705.70917,E,1,06,1.3,121.4,M,0.0,M,,*70".split(',')
arr[0]  # Sentence ID
arr[1]  # UTC, 080831.000 means HHMMSS.sss, 08시 07분 31.000 초(UTC)
arr[2]  # Latitude, 3724.73827 means ddmm.mmmmm
arr[3]  # N=North, S=South
arr[4]  # Longitude, 12705.70917 means dddmm.mmmmm
arr[5]  # E=East, W=West
arr[6]  # 0=Invalid, 1=Valid SPS, 2=Valid DGPS, 3=Valid PPS
arr[7]  # Satelites being used(0~12)
arr[8]  # HDOP, Horizontal Dilution of Precision
arr[9]  # Altitude(MSL: Mean Sea Level)
arr[10] # Altitude Units, M=Meters
arr[11] # Geoid seperation, MSL-Geoid
arr[12] # Geoid Units, M=Meters
arr[13] # DGPS Age
arr[14] # DGPS Station ID, where is it???
arr[15] # Checksum
arr[16] # terminator, CR/LF
```

##### In `RMC` #####
`RMC` includes current date.
```python
arr = "$GNRMC,080831.000,A,3724.73827,N,12705.70917,E,0.94,150.53,050221,,,A*7B".split(',');
arr[9] # 050221 means ddmmyy, 05일 02월 21년
```

<br>

### Convert GPS data to degree/minute ###
* Latitude : `3724.73827` -> `37` and `24.73827`, `37` is degree and `24.73827` is minute.  
* Minute to Degree : `24.73827 / 60 = 0.4123045`  
* Degrees Latitude : `37 + 0.4123045 = 37.4123045`
* Search that latitude in google map!