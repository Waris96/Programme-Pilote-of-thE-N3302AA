# Programme-Pilote-of-thE-N33A01A
Code to pilote the N33A01A load for controlling the current in the circuit
#---------------------------------------------------------------------
# NOTE: the default pyvisa import works well for Python 3.6+
# if you are working with python version lower than 3.6, use 'import visa' instead of import pyvisa as visa
# install: pyvisa, pyvisa-py and PySerial
# example (in MS-DOS window): pip install -U pyvisa pyvisa-py PySerial

# V1.00 JUN-2022 T. ISAMBERT/ T.ISAMBERT_A.W DJIBRIL Creation
# V2.00 JUL-2022 T. ISAMBERT/ A.W DJIBRIL Addin JSON table and different controling of load moduleDevices

import pyvisa as visa
import time
import json
import math

#---------------------------------------------------------------------
#                  S E T U P   V A R I A B L E S
#---------------------------------------------------------------------
# General
ON = 1
OFF = 0
verbose = False
commPort = 6
moduleDevices = [{"name": "N3302A", "amp": 30, "watt": 150},{"name": "N3304A", "amp": 60, "watt": 300}, {"name": "HP60504B", "voltMaxi": 60, "currentMaxi": 120, "wattMaxi": 600}]

class Module:
        def __init__(self, channel, name, currentMaxi, wattMaxi, currentMaxiPerLoad, currentActual, status):
                self.channel = channel
                self.name = name
                self.currentMaxi = currentMaxi
                self.wattMaxi = wattMaxi
                self.currentMaxiPerLoad = currentMaxiPerLoad
                self.currentActual = currentActual
                self.status = status

# Parametrable variables
#-----------------------
# All current are in amperes
currentStart = 2.5
currentStop = 10.0
currentStandby = 1.5
currentStep = 0.1

# Take into account the watt of output (150W or 300W)
inputVoltage = 20.0

# All time are in seconde
timeOn = 2.0
timeOff = 0.0

#---------------------------------------------------------------------
#                  S U B R O U T I N E S
#---------------------------------------------------------------------
# Replace range(start, stop, step) with decimal values and different ways!
def drange(start, stop, step):
        r = start
        while r < stop+step:
                yield r
                if start<stop:
                        r += step
                else:
                        r -= step

# Read JSON array
def getModule(name):
        for obj in moduleDevices:
                if name==obj["name"]:
                        return [name, obj["currentMaxi"], obj["wattMaxi"]]

def sendCommand(Mat, Command):
        if verbose:
                print('DEBUG: command sent = %s' % (Command))
        Mat.write(Command)

# Print module information
def printModuleInfo(channel=-1):
        for i in range(0, len(mod)):
                txt="OFF"
                if mod[i]["status"]==1:
                        txt="ON"
                if channel==-1:
                        print('Load %s: %s, Imaxi=%dA, Wmaxi=%dW, Imaxi/load=%dA, Iactual=%.3fA, Status=%s\n' % (mod[i]["channel"], mod[i]["name"], mod[i]["currentMaxi"], mod[i]["wattMaxi"], mod[i]["currentMaxiPerLoad"], mod[i]["currentActual"], txt))
                elif mod[i]["channel"]==f"{channel}":    
                        print('Load %s: %s, Imaxi=%dA, Wmaxi=%dW, Imaxi/load=%dA, Iactual=%.3fA, Status=%s\n' % (mod[i]["channel"], mod[i]["name"], mod[i]["currentMaxi"], mod[i]["wattMaxi"], mod[i]["currentMaxiPerLoad"], mod[i]["currentActual"], txt))

# Setup the module of electronic load
def setupLoad(channel, current, onOff=-1):
        global eLoad, mod
        txt="OFF"
        if onOff==1:
                txt="ON"
        # Select load
        sendCommand(eLoad, ':CHANnel:LOAD %G' % (channel))
        # Set-up current of load
        sendCommand(eLoad, ':SOURce:TRANsient:MODE CONTinuous')
        sendCommand(eLoad, ':SOURce:CURRent:LEVel:IMMediate:AMPLitude %G' % (current)) 
        # Memorise current of active load
        mod[channel-1]["currentActual"]=current
        if onOff!=-1 and onOff!=mod[load-1]["status"]:
                sendCommand(eLoad, ':SOURce:OUTPut:STATe %d' % (onOff))
                # Memorise status of load
                mod[channel-1]["status"]=onOff
        if verbose:
                if onOff!=-1:
                        print('Load(%d) current = %.3fA, status = %s' % (channel, current, txt))

# Set the current using maxi current/load
def setCurrent(current):
        global mod

        def getCurrentLoads():
                current_load = 0.0
                for i in range(0, len(mod)):
                        if mod[i]["status"]==ON:
                                current_load+=mod[i]["currentActual"]
                return current_load

        a = 0.0
        # set up loads only for growing current
        for i in range(0, len(mod)):
                if mod[i]["status"]==ON:
                        if mod[i]["currentActual"]<=mod[i]["currentMaxiPerLoad"]:
                                a += current
                                setupLoad(mod[i]["channel"], current, ON)
                                break
                else:
                        if (current-a)<=mod[i]["currentMaxiPerLoad"]:
                                a += current
                                setupLoad(mod[i]["channel"], current-a, ON)
                        else:
                                a += mod[i]["currentMaxiPerLoad"]
                                setupLoad(mod[i]["channel"], mod[i]["currentMaxiPerLoad"], ON)

#---------------------------------------------------------------------
#                  M A I N    P R O G R A M
#---------------------------------------------------------------------
# Detection of connected resources
rm = visa.ResourceManager()
eLoad = rm.open_resource("ASRL" + str(commPort) + "::INSTR", write_termination = '\r\n', read_termination = '\r\n')

# timeout in ms
eLoad.timeout = 10000
print('Connected to: %s' % (eLoad))

# Reset
sendCommand(eLoad, '*RST') 

# Get information from material
sendCommand(eLoad, '*IDN?')
print('Material: %s' % (eLoad.read()))

sendCommand(eLoad, ':SYSTem:VERSion?')
print('Version: %s' % (eLoad.read()))
sendCommand(eLoad, '*RDT?')

# Reading all module and setup JSON array
jsondata = []
for x in eLoad.read().split(";"):
        a=x.split(":")
        channel = a[0][-1:] 
        name = a[1]
        b=getModule(name)
        currMaxi= math.floor(b[2]/voltageInput)
        if currMaxi>b[1]:
                currMaxi=b[1]
        jsondata=Module(channel, name, b[1], b[2], currMaxi, 0.0, OFF)
        mod.append(jsondata.__dict__)
        printModuleInfo(channel)

iMax = 0
for i in range(0, len(mod)):
        iMax += math.floor(mod[i]["wattMaxi"]/voltageInput)

# Print information before starting test
printMessage('----------------------------------\n')
printMessage('Voltage input: %.3fV\n' % (voltageInput))
printMessage('Current maxi available: %.3fV\n' % (iMax))
printMessage('Current start: %.3fA\n' % (currentStart))
printMessage('Current stop: %.3fA\n' % (currentStop))
printMessage('Current step: %.3fA\n' % (currentStep))
printMessage('Current standby: %.3fA\n' % (currentStandby))
printMessage('Time ON: %.3fs\n' % (timeOn))
printMessage('Time OFF: %.3fs\n' % (timeOff))
printMessage('----------------------------------\n')

# Enable load with standby current
setCurrent(currentStandby)
input('Press [ENTER] to start the program:')
print('')

# Loop to increase the output current value
for a in drange(currentStart, currentStop, currentStep):
        setCurrent(a)
        print('Current: %.3fA during: %.3fs' % (a, timeOn))
        time.sleep(timeOn)
        if timeOff>0:
                setCurrent(currentStandby)
                print('Current: %.3fA during: %.3fs' % (currentStandby, timeOff))
                time.sleep(timeOff)

# Disable load
for i in range(0, len(mod)):
        if mod[i]["status"]==ON:
            sendCommand(eLoad, ':CHANnel:LOAD %G' % (i))
            sendCommand(eLoad, ':SOURce:OUTPut:STATe %d' % (OFF))

# Reset
sendCommand(eLoad, '*RST') 

# Stop communication
eLoad.close()
rm.close()

# end of program
