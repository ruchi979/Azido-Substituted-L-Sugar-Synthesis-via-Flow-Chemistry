from os import listdir
from os.path import isfile, join
import serial 

import numpy as np
import pandas as pd 
import time 
from scipy.integrate import trapz

#step1: be sure to the address of the files that the ftir data is exported is matching to line 11 (mypath)
mypath = r"C:\\Users\\Admin\\Desktop\\ruchi\\Exp 2024-09-21 10-04"
onlyfiles = [f for f in listdir(mypath) if isfile(join(mypath, f))]
#step2: make sure that pump and the potentiostat is correctly addressed in the line 16 and 17
pump_1=serial.Serial("COM30",9600) #HPLC Pump
#pump_2 = serial.Serial("COM29",9600) #HPLC Pump
temp_sensor = serial.Serial('COM19', 9600)  # Replace 'COM10' with the correct port
printer = serial.Serial("COM6", 115200, timeout=1)
arduino_port = 'COM52'
baud_rate = 9600

# Establish a serial connection
ser = serial.Serial(arduino_port, baud_rate, timeout=1)
time.sleep(2)  # Wait for the connection to initialize

#step 3: grab the lines from 22 to 90 and presss f9 
def area_under(data,start,end):
    x = np.flip(data.iloc[start:end,0].to_numpy())
    y = np.flip(data.iloc[start:end,1].to_numpy())
    area = trapz(y,x)
    return np.abs(area)

def file_namer(num):
    str1 = str(num)
    length = int(len((str1)))
    empt = ''
    for i in range(5-length):
        empt = empt+'0'
        
    return empt+str1

        
def ftir_extract(filename,init,end):

    filename = filename

    temp_df = pd.read_csv(filename)
    # nump_df = temp_df.to_numpy()
    area = area_under(temp_df, init, end)
    # max_peak = np.max(nump_df[90:120,1])
    print(area)

    return area


#--------------------------------------------------------------------------------
#Ardunio and temp related functions

def send_command(command):
    temp_sensor.write(command.encode() + b'\n')  # Send the command to Arduino with a newline character
    temp_sensor.flush()  # Flush the serial buffer

def read_response_temp():
    timeout = 5  # Maximum time to wait for data in seconds
    interval = 0.5  # Interval to check for data in seconds
    start_time = time.time()

    while time.time() - start_time < timeout:
        if temp_sensor.in_waiting > 0:
            line = temp_sensor.readline().decode().strip()
            if line.startswith("Temp:"):
                data = line.split(":")[1].split(",")
                if len(data) == 4:
                    temperatureC = float(data[0])
                    cold = float(data[1])
                    hot = float(data[2])
                    return temperatureC, cold, hot
        time.sleep(interval)  # Wait for the specified interval

    return "No response received"

def print_temperature_data():
    send_command("GetData")
    temperatureC, cold, hot = read_response_temp()
    if temperatureC is not None:
        print(f"Current Temperature (Celsius): {temperatureC} °C")
        print(f"Cold Setpoint: {cold}")
        print(f"Hot Setpoint: {hot}")
    else:
        print("Failed to read temperature data.")
        
#--------------------------------------------------------------------------------


def function(flowrate_1,flowrate_2,temp,pressure_1):
    
     #set the pumps with the flowrate as the desired flowrate for the function
     
   #set the pumps with the flowrate as the desired flowrate for the function
     
    time.sleep(0.1)
    fr_1=flowrate_1*1000 #converting ml/min to ul/min as pump takes this as input val
    pump_1.write(('flow:'+ str(fr_1) + '\r').encode())
    
    #time.sleep(0.1)
    #fr_2=flowrate_2*1000 #converting ml/min to ul/min as pump takes this as input val
   # pump_2.write(('flow:'+ str(fr_2) + '\r').encode())
        
    # Set initial values for temperature to start heating at. 
    time.sleep(2)
    cold_setpoint = temp #26.4
    hot_setpoint = 1000 #dummy val since no cooling mechanism is there. High value means cooling code is never triggered
    send_command(f'C{cold_setpoint}')
    send_command(f'H{hot_setpoint}')
    
    set_max_pressure(pressure_1)
    #pumps run
    pump_1.write(b'on\r')
    time.sleep(0.1)
   # pump_2.write(b'on\r')
   
    time.sleep(50)

 
def function2():
    time.sleep(70)
    
    files = [f for f in listdir(mypath) if isfile(join(mypath, f))]

    val = 0
    
    #change wavelengths as per product here.
    f_row=15 #first row of range for wavelength as per IR CSV
    l_row=21 #last row of range for wavelength as per IR CSV
    
    val += ftir_extract(files[-1],f_row,l_row)
    val += ftir_extract(files[-2],f_row,l_row)
    val += ftir_extract(files[-3],f_row,l_row)

    avg_val = val/3
    
    return avg_val
def set_max_pressure(value):
    command = f"setMaxPressure {value}\n"
    ser.write(command.encode())
    time.sleep(1)  # Wait for Arduino to process the command
    response = ser.readline().decode('utf-8').strip()
    print(response)

def set_min_pressure(value):
    command = f"setMinPressure {value}\n"
    ser.write(command.encode())
    time.sleep(1)  # Wait for Arduino to process the command
    response = ser.readline().decode('utf-8').strip()
    print(response)


#step 4:grab the line 93 and f9
from skopt.optimizer import Optimizer


#step5:in line 96 we have to define the range that (flowrate_1,Flowrate_2,temperature,pressure) (from,to) and after (anytime) appllying changes you need to grab the line 96 and f9
#flowrates are in ml/min, temperature is in celsius
bounds = [(1.0,10.0),(25.0,50.0),(3.0,7.0)]
#step 6: grab the line 100 and f9
opter =Optimizer(bounds,base_estimator='gp',n_initial_points=3,acq_func="EI",random_state= 42)


#step7: to selecte number of the cycles that you have to do the experiment and then grab the line 104 to 121 and f9: the closed loop experimentation is initiated
number_of_cycles = 22
results = []
flowrates_1 = []
flowrates_2 = []
temp = []
Pressure_1 = []

product_wavelength=True #set to true if product wavelengths being monitored

if product_wavelength == True:
    val=1
else:
    val=-1
    
# Step 8: Test Tubes on Printer
USE_PRINTER = True
REST_HEIGHT = 200
X_HOME = -5
Y_HOME = 20
Z_HOME = 175
DEFAULT_PUMP_TIME="1"
# Distance between test tubes
X_SPACING=20
Y_SPACING=20
# Number of test tubes
X_ROWS = 11
Y_COLUMNS = 4

def send_cmd(cmd):
    print(cmd)
    printer.write(f"{cmd}\n".encode("ASCII"))

def move(x=None, y=None, z=None):
    s = "G0"
    if x is not None:
        s += f"X{x}"
    if y is not None:
        s += f"Y{y}"
    if z is not None:
        s += f"Z{z}"
    
    s+= "F5000"
    send_cmd(s)

def printer_positions():
    for j in range(Y_COLUMNS):
        for i in range(X_ROWS):
            if j%2==1:
                yield (X_HOME + (X_ROWS - 1 - i) * X_SPACING, Y_HOME + j * Y_SPACING, Z_HOME)
            else:
                yield (X_HOME + i * X_SPACING, Y_HOME + j * Y_SPACING, Z_HOME)

# Run this.
tube_location = list(printer_positions())

try:

    for i in range(number_of_cycles):
        move(*tube_location[2*i])
        asked = opter.ask()
        print(asked[0])
        print(asked[0])
        print(asked[1])
        print(asked[2])
        function(asked[0],asked[0],asked[1],asked[2])
        
        move(*tube_location[2*i+1])
        told= function2()
        
        print(f"area under the curve in the round {i:.2f} = {told:.2f}")
        opter.tell(asked,-told*val)

        results.append(told)
        flowrates_1.append(asked[0])
        flowrates_2.append(asked[0])
        temp.append(asked[1])
        Pressure_1.append(asked[2])

        dict1 = {"flowrate_1":flowrates_1,"flowrate_2":flowrates_2,"temp":temp,"Pressure_1":Pressure_1,"area-results":results}
        df2 = pd.DataFrame(dict1)
        df2.to_csv("output round "+str(i)+".csv")
finally:

    cold_setpoint = -1000 #set very low that the current temp is above cold set. This means heater will switch off.
    send_command(f'C{cold_setpoint}') 
    temp_sensor.close()
   
    pump_1.write(b'off\r')
    pump_1.close()

    #pump_2.write(b'off\r')
    #pump_2.close()

   

