# Screvle
--****************************************************************************
--**********************    Hand-held CANbus analyser   **********************
--****************************************************************************
--**********************   Gemaakt door: Wout Geysels   **********************
--****************************************************************************
--**********************   In opdracht van: De Lijn     **********************
--****************************************************************************
--**********************           Versie 1.1           **********************
--****************************************************************************
--****************************************************************************

--****************************   INITIALITATIE   *****************************
system.enable5V()		      	--zet de 5v aan, voor CAN module

can = peripheral.open( "can1" )     	--open peripheral
can.bitrate = 250000                	--bitrate instellen op 250kb/s
can.rxbuffersize = 10		      	--buffersize (10 CANberichten)

uart = peripheral.open( "uart1" )   	--open uart peripheral met naam 'uart1'
uart.baudrate = 9600 

minPntDefault = math.tofloat("5.04")  	--5.04km/h/s = 1,4 m/s² is waarde volgens de norm, onder de waarde is goede acceleratie
maxPntDefault = math.tofloat("5.7")   	--5.7km/h/s = 1,583 m/s² is waarde volgens de norm, boven de waarde is slechte acceleratie
minPnt = minPntDefault			--bij init is de onderste grens gelijk aan de waarde volgens de norm
maxPnt = maxPntDefault 			--bij init is de bovenste grens gelijk aan de waarde volgens de norm

--****************************************************************************
--******************************   HOOFDMENU   *******************************
--****************************************************************************
function hoofdMenu()  						--startscherm, keuze overzicht tussen de verschillende toepassingen
   can:registerCallback( 0, nil )  				--ophouden met het inlezen van de J1587 data
   uart:registerCallback( 0, nil )  				--ophouden met het inlezen van de J1587 data

   local J1939Button = mgf.createButton("1. J1939")		--button aanmaken: naam is 'J1939Button' met label "1. J1939"
   local J1587Button = mgf.createButton("2. J1587")		--button aanmaken: naam is 'J1587Button' met label '2. J1587"
   local accButton = mgf.createButton("3. Accelerometer")	--button aanmaken: naam is 'accButton' met label "3. Accelerometer"
   local setupButton = mgf.createButton("4. Setup")		--button aanmaken: naam is 'setupButton' met label "4. J1939"
   local turnoffButton = mgf.createButton( "-- Shut down --" ) 	--button aanmaken: naam is 'turnoff' met label "-- shut down --"
   
   local panel = mgf.createPanel()				--panel aanmaken, een panel kunnen we laten zien op het scherm
  
   panel:addWidget(J1939Button, 0, 0, 240, 72)			--wigdet 'J1939Button' toevoegen aan panel
   panel:addWidget(J1587Button, 0, 72, 240, 72)			--wigdet 'J1587Button' toevoegen aan panel 
   panel:addWidget(accButton, 0, 144, 240, 72)			--wigdet 'accButton' toevoegen aan panel 
   panel:addWidget(setupButton, 0, 216, 240, 72)		--wigdet 'setupButton' toevoegen aan panel 
   panel:addWidget(turnoffButton, 0, 288, 240, 32 )		--wigdet 'turnoffButton' toevoegen aan panel 
   mgf.setWidget(panel)						--het panel op scherm afdrukken

   J1939Button.actionListener = J1939Functie			--op de button 'J1939Button' gedrukt? ga naar J1939Functie
   J1587Button.actionListener = J1587Functie			--op de button 'J1587Button' gedrukt? ga naar J1939Functie
   accButton.actionListener = accFunctie			--op de button 'accButton' gedrukt? ga naar J1939Functie
   setupButton.actionListener = setupFunctie			--op de button 'setupButton' gedrukt? ga naar J1939Functie
   turnoffButton.actionListener = turnoffListener		--op de button 'turnoffButton' gedrukt? ga naar J1939Functie
end

--****************************************************************************
--*****************************   CAN-BUS   **********************************
--****************************************************************************
function J1939Functie()  --1.1				--start functie voor het inlezen van J1939 berichten
  led.green = 0						--groenwaarde van RGBled op '0' zetten
  led.red = 0 						--roodwaarde van RGBled op '0' zetten
  logStatus = 0                   			--'0' is NIET bezig met loggen, '1' is bezig met loggen

  lfs.mkdir( "J1939logfiles" ) 				--aanmaken van een map met naam "J1939logfiles"

							--ID-waardes toekennen aan hun variabele, voor het filteren
  local toerentalID = 0xCF00400  --217056256dec         	
  local snelheidID  = 0x18FEF100 --419361024dec
  local verbruikID  = 0x18FEF200 --419361280dec
  local afstandID   = 0x18FEC1EE --419348974dec
  local tempID      = 0x18FEF500 --419362048dec
  local fuelID      = 0x18FE563D --419321405dec
  local pedalPosID  = 0xCF00300  --217056000dec
  
  local titelLabel = mgf.createLabel( "J1939" )		--label aanmaken: naam is 'titelLabel' en tekstvak "J1939"
  local rpmLabel = mgf.createLabel("...rpm")		--label aanmaken: naam is 'rpmLabel' en tekstvak "...rpm"
  local speedLabel = mgf.createLabel("...km/h")		--label aanmaken: naam is 'speedLabel' en tekstvak "...km/h"
  local verbrLabelAct = mgf.createLabel("...l/100km")	--label aanmaken: naam is 'verbrLabelAct' en tekstvak "...l/100km"
  local verbrLabelGem = mgf.createLabel("...l/h")	--label aanmaken: naam is 'verbrLabelgem' en tekstvak "...l/h"
  local afstandLabel = mgf.createLabel("...km")		--label aanmaken: naam is 'afstandLabel' en tekstvak "...km"
  local tempLabel = mgf.createLabel("...C")		--label aanmaken: naam is 'tempLabel' en tekstvak "...C"
  local fuelLvlLabel = mgf.createLabel("lvl = ...")	--label aanmaken: naam is 'fuelLvlLabel' en tekstvak "lvl = ..."
  local positieLabel = mgf.createLabel("pedal = ...")	--label aanmaken: naam is 'positieLabel' en tekstvak "pedal = ..."
  local startButton = mgf.createButton("start")		--button aanmaken: naam is 'startButton' en tekstvak "start"
  local stopButton = mgf.createButton("stop")		--button aanmaken: naam is 'stopButton' en tekstvak "stop"
  local backButton = mgf.createButton("back")		--button aanmaken: naam is 'backButton' en tekstvak "back"
  local panel = mgf.createPanel()			--panel aanmaken, een panel kunnen we laten zien op het scherm

  panel:addWidget( titelLabel, 0, 0, 240, 30 )
  panel:addWidget( rpmLabel, 0, 30, 150, 30 )
  panel:addWidget( speedLabel, 0, 60, 150, 30 )
  panel:addWidget( verbrLabelAct, 0, 90, 150, 30 )
  panel:addWidget( verbrLabelGem, 0, 120, 150, 30 )
  panel:addWidget( afstandLabel, 0, 150, 150, 30 )
  panel:addWidget( tempLabel, 0, 180, 150, 30 )
  panel:addWidget( fuelLvlLabel, 0, 210, 150, 30 )
  panel:addWidget( positieLabel, 0, 240, 150, 30 )
  panel:addWidget( startButton, 150, 30, 90, 120 )
  panel:addWidget( stopButton, 150, 150, 90, 120 )
  panel:addWidget( backButton, 0, 270, 240, 50 )
  --mgf.setWidget( panel )

  function callback( peripheral, id, buf )		--telkens we een CANbericht ontvangen komen we naar deze functie 
							--waar we 3 parameters meegeven
    local message = canmessage.decode( buf )		--in de variabele 'message' schrijven we de data van de buffer  
							--(gedecodeerd volgens CAN protocol)
    if( message.id == toerentalID ) then                --wanneer het ID van dit bericht gelijk is aan dat van 'toerentalID'
      local rpm = ((message[ 4 ] + message[ 5 ]*256))/8 --resolutie toerental is 0.125rpm/bit, byte 4 is LSB, byte 5 is MSB
      rpmLabel.label = rpm.."rpm"			--waarde van 'rpm'schrijven naar zijn label 'rpmLabel'
      name = "toerental"				--string "toerental" in variabele zetten om te gebruiken bij het loggen
      value = rpm					--de waarde van 'rpm' in variabele zetten om te gebruiken bij het loggen
      eenheid = "rpm"					--string "rpm" in variabele zetten om te gebruiken bij het loggen
            
    elseif( message.id == snelheidID ) then             --wanneer het ID van dit bericht gelijk is aan dat van 'snelheidID'
      local snelheid = message[ 3 ] 			--resolutie is 1km/h/bit, dus ik lees enkel de MSB
      speedLabel.label = snelheid.."km/h"		--waarde van 'snelheid' schrijven naar zijn label
      name = "snelheid"
      value = snelheid
      eenheid = "km/h"
      
    elseif( message.id == verbruikID ) then             --wanneer het ID van dit bericht gelijk is aan dat van 'verbruikID'
      local verbruikAct = math.tostring((message[ 4 ]/2) + (message[ 3 ]/512),1) --resolutie is 
      local verbruikGem = math.tostring((message[ 2 ]*(math.tofloat("12.8"))) + (message[ 1 ]/20),1)
      verbruikLabelAct.label = verbruikAct.."l/100km"
      verbruikLabelGem.label = verbruikGem.."l/h"
      name = "verbruik act."
      value = verbruikAct
      eenheid = "l/100km"
      name2 = "verbruik gem."
      value2 = verbruikGem
      eenheid2 = "l/h"
       
    elseif( message.id == afstandID ) then                   --totaal aantal km
      local afstand = math.tostring(( message[ 1 ] + (message[ 2 ]*256) + (message [ 3 ]*256*256) + (message [ 4 ]*256*256*256))/200,0)
      afstandLabel.label = afstand.."km"
      name = "totale afstand"
      value = afstand
      eenheid = "km"
      
    elseif( message.id == tempID ) then                      --temperatuur
      local temp = math.tostring((( message[ 4 ] + (message[ 5 ]*256))*math.tofloat("0.03125"))-273,0)
      print("temperatuur = "..temp)
      temperatuurLabel.label = temp.." C"
      name = "temperatuur"
      value = temp
      eenheid = "Celsius"
      
    elseif( message.id == fuelID ) then                      --fuel level
      local level = math.tostring( message[ 1 ] * (math.tofloat("0.4")),1)
      fuelLvlLabel.label = "fuel ="..level.."%"
      name = "brandstofniveau"
      value = level
      eenheid = "%"
      
    elseif( message.id == pedalPosID ) then                  --positie gaspedaal
      local positie = ( message[ 2 ] * (math.tofloat("0.4")))
      positieLabel.label = "gas ="..math.tostring(positie,0).."%"
      name = "gaspedaal pos."
      value = positie
      eenheid = "%"
    end
    
    if (logStatus == 1 and file ~= nil and system.getTicks() >= timeout) then
      time = system.getTime()
      file:write(name..","..value..","..eenheid..","..time.weekday..","..time.day..","..time.month..","..time.year..","..time.hours..","..time.minutes..","..time.seconds.."\r\n")
      --file:write("toerental,"..rpm..",rpm,"..time.weekday..","..time.day..","..time.month..","..time.year..","..time.hours..","..time.minutes..","..time.seconds.."\r\n")
      if (message.id == verbruikID) then
        file:write(name2..","..value2..","..eenheid2..","..time.weekday..","..time.day..","..time.month..","..time.year..","..time.hours..","..time.minutes..","..time.seconds.."\r\n")
      end
       timeout = system.getTicks()+100  --of moet dit voor vorige end?
    end

  mgf.repaint()
end --einde callbackfunctie


--------------------------------------------  
function startButton.actionListener()
    if logStatus == 0 then
      logStatus = 1 
      time = system.getTime()
    file = io.open("J1939logfiles/parameters " ..time.day.."-"..time.month.."-"..time.year.."_"..time.hours.."u"..time.minutes.."m"..time.seconds.."s.csv", "w" )
      --file = io.open("eindwLOG/toerental.csv", "w" )
    file:write("Parameter,Waarde,Eenheid,weekdag,dag,maand,jaar,uur,minuut,second\r\n")
    led.green = 20
    led.red = 0
    end
  end  
  
  function stopButton.actionListener() 
    if logStatus == 1 then
    logStatus = 0    
    file:close()
    led.green = 0
    led.red = 20
    end
  end 
  
  function backButton.actionListener()
    if logStatus == 1 then
      file:close()
      logStatus = 0
    end
    led.green = 0
    led.red = 0
    hoofdMenu()
  end
mgf.setWidget(panel)
can:registerCallback( 0, callback )
end

--****************************************************************************
--******************************   UART   ************************************
--****************************************************************************
function J1587Functie()
  
i=0
  logStatus = 0
a = buffer.create( 100 )
check_reset = 512 +256
check = check_reset
  
  lfs.mkdir( "J1587logfiles" )

  local titelLabel = mgf.createLabel( "J1587" )
  local rpmLabel = mgf.createLabel("... rpm")
  local snelheidLabel = mgf.createLabel("... km/h")
  local verbruikLabel = mgf.createLabel("... l/h")
  local startButton = mgf.createButton("start")
  local stopButton = mgf.createButton("stop")
  local backButton = mgf.createButton("back")
   panel = mgf.createPanel()

  panel:addWidget( titelLabel, 0, 0, 240, 30 )
  panel:addWidget( rpmLabel, 0, 30, 240, 50 )
  panel:addWidget( snelheidLabel, 0, 80, 240, 50 )
  panel:addWidget( verbruikLabel, 0, 130, 240, 50 )
  panel:addWidget( startButton, 0, 180, 120, 90 )
  panel:addWidget( stopButton, 120, 180, 120, 90 )
  panel:addWidget( backButton, 0, 270, 240, 50 )

function callback( peripheral, id, buf )
  for counter=1,buf.n do
    i=i+1
    --zoek MID: 128 = engine ECU, 144 = vehicle ECU
    if i==1 then
      if buf[ counter ] == 128 or buf[ counter ] == 144 then
        a[i] = buf[ counter ]
        check = check-a[i]
        --print("MID ="..a[i].." and check ="..check)
      else
        i=0
        check = check_reset
      end

    --zoek PID: 190 = toerental, 183 = verbruik, 84 = snelheid
    elseif i==2 then  
      if (a[1] == 128 and (buf[ counter ] == 190 or buf[ counter ] == 183)) or (a[1] == 144 and buf[ counter ] == 84) then 
        a[i] = buf[ counter ]
        check = check-a[i]
        --print("PID ="..a[i].." and check ="..check)
      else
        i=0
        check = check_reset
      end
 
    --data na MID en PID opslaan tot checksum
    elseif i >= 3 and check > 0 then 
      a[i] = buf[ counter ]
      check = check-a[i]
     
      --heel het bericht is ontvangen
    end
    
    if check == 0 then
      if a[1] == 128 then
        if a[2] == 190 then --toerental
          toerental = (a[3]+(a[4]*256))*math.tofloat("0.25")
          rpmLabel.label = "toerental ="..toerental.."rpm"
          print("toerental ="..math.tostring(toerental,0).."rpm")
        elseif a[2] == 183 then --verbruik
          fuel = (a[3]+(a[4]*256))*math.tofloat("0.06")
          verbruikLabel.label = "verbruik ="..fuel.."l/h"
          print("fuel rate ="..math.tostring(fuel,2).."l/h")
        end
      elseif a[1] == 144 then
        if a[2] == 84 then --snelheid
          snelheid = a[3]*math.tofloat("0.8")
          snelheidLabel.label = "snelheid ="..snelheid.."km/h"
          print("snelheid ="..math.tostring(snelheid,2).."km/h")
        end
      end
      i=0
      check = check_reset
  
      --dit bericht was fout, begin opnieuw
    elseif check<0 then 
      if( i < 21 ) then
        check = check + 256
      else
        --print("checksum kleiner dan nul!", check, i) 
        i=0
        check = check_reset
      end
    end
    if (logStatus == 1 and file ~= nil and system.getTicks() >= timeout) then
      time = system.getTime()
      file:write(name..","..value..","..eenheid..","..time.weekday..","..time.day..","..time.month..","..time.year..","..time.hours..","..time.minutes..","..time.seconds.."\r\n")
      --file:write("toerental,"..rpm..",rpm,"..time.weekday..","..time.day..","..time.month..","..time.year..","..time.hours..","..time.minutes..","..time.seconds.."\r\n")
      if (message.id == verbruikID) then
        file:write(name2..","..value2..","..eenheid2..","..time.weekday..","..time.day..","..time.month..","..time.year..","..time.hours..","..time.minutes..","..time.seconds.."\r\n")
      end
    timeout = system.getTicks()+100  --of moet dit voor vorige end?
    end

  end  
end --einde callbackfunctie J1587
  
  
--------------------------------------------  
function startButton.actionListener()
    print(logStatus)
    if logStatus == 0 then
      logStatus = 1 
      time = system.getTime()
    file = io.open("J1587logfiles/parameters " ..time.day.."-"..time.month.."-"..time.year.."_"..time.hours.."u"..time.minutes.."m"..time.seconds.."s.csv", "w" )
      --file = io.open("eindwLOG/toerental.csv", "w" )
    file:write("Parameter,Waarde,Eenheid,weekdag,dag,maand,jaar,uur,minuut,second\r\n")
    led.green = 20
    led.red = 0
    end
  end  
  
  function stopButton.actionListener() 
    if logStatus == 1 then
    logStatus = 0    
    file:close()
    led.green = 0
    led.red = 20
    end
  end 
  
  function backButton.actionListener()
    if logStatus == 1 then
      file:close()
      logStatus = 0
    end
    led.green = 0
    led.red = 0
    hoofdMenu()
  end
  
  mgf.setWidget(panel)
  print("wachten op data")
  uart:registerCallback( 0, callback )
end

--****************************************************************************
--**************************   ACCELEROMETER   *******************************
--****************************************************************************
function accFunctie()
  
gpio.setPins( gpio.PORTC, 0x000C )
gpio.setMode( gpio.PORTC, 0x000C, gpio.ModeOutputPP )
spi = peripheral.open( "spi1" )
spi.port = gpio.PORTC
spi.pin = 0x0008
spi.polarity = 1
spi.phase = 1
spi.speed = 100000
buf = buffer.create( 12 )

--CONFIG =          0x1A 
--ACCEL_CONFIG =    0x1C 
ACCEL_XOUT_H =    0x3B + 128 
ACCEL_XOUT_L =    0x3C + 128 
ACCEL_YOUT_H =    0x3D + 128 
ACCEL_YOUT_L =    0x3E + 128 
ACCEL_ZOUT_H =    0x3F + 128 
ACCEL_ZOUT_L =    0x40 + 128 
--ACC_FULL_SCALE_2_G = 0x00

buf[1] = 0x6B
buf[2] = 0x00
spi:write( buf, 2 ) --reset

--buf[1] = ACCEL_CONFIG
--buf[2] = ACC_FULL_SCALE_2_G
--spi:write( buf, 2 ) --schaal accelero

local titelLabel = mgf.createLabel("versnelling")
local accGLabel = mgf.createLabel("...")
local accMps2Label = mgf.createLabel("...")
local accKmPHourPSecLabel = mgf.createLabel("...")
local startButton = mgf.createButton("start")
local resetButton = mgf.createButton("reset")
local stopButton = mgf.createButton("stop")
local backButton = mgf.createButton("Back")
panel=mgf.createPanel()
panel:addWidget(titelLabel, 0, 0, 240, 30)
panel:addWidget(accGLabel, 0, 30, 240, 50)
panel:addWidget(accMps2Label, 0, 80, 240, 50)
panel:addWidget(accKmPHourPSecLabel, 0, 130, 240, 50)
panel:addWidget(startButton, 0, 180, 80, 90)
panel:addWidget(resetButton, 80, 180, 80, 90)
panel:addWidget(stopButton, 160, 180, 80, 90)
panel:addWidget(backButton, 0, 270, 240, 50)

theme = mgf.createTheme()--thema aanmaken voor achtergrondkleur te kunnen aanpassen
accGLabel.theme = theme
accMps2Label.theme = theme
accKmPHourPSecLabel.theme = theme

status = "reset"

function callback()
buf[1] = ACCEL_XOUT_H
spi:writeRead( buf, 1, buf, 2 )
  valueH = buf[2]
  if valueH >= 128 then valueH = valueH - 256 end
    buf[1] = ACCEL_XOUT_L
    spi:writeRead( buf, 1, buf, 2 ) 
    valueL = math.tofloat(buf[2],2)  
    totaalX = (valueL + valueH*256)/16384 --scale factor: 16384 LSB/G
-------------------------------------------------------------- 
  buf[1] = ACCEL_YOUT_H
  spi:writeRead( buf, 1, buf, 2 )
  valueH = buf[2]
  if valueH >= 128 then valueH = valueH - 256 end
    buf[1] = ACCEL_YOUT_L
    spi:writeRead( buf, 1, buf, 2 ) 
    valueL = math.tofloat(buf[2],2)  
    totaalY = (valueL + valueH*256)/16384 --scale factor: 16384 LSB/G
--------------------------------------------------------------  
  buf[1] = ACCEL_ZOUT_H
  spi:writeRead( buf, 1, buf, 2 )
  valueH = buf[2]
  if valueH >= 128 then valueH = valueH - 256 end
    buf[1] = ACCEL_ZOUT_L
    spi:writeRead( buf, 1, buf, 2 ) 
    valueL = math.tofloat(buf[2],2)  
    totaalZ = (valueL + valueH*256)/16384 --scale factor: 16384 LSB/G
--print("X:"..math.tostring(totaalX).."   Y:"..math.tostring(totaalY).."   Z:"..math.tostring(totaalZ))

  Gtotaal = math.tofloat(math.sqrt(totaalX*totaalX + totaalY*totaalY + totaalZ*totaalZ)-1) --total g-kracht
  print(math.tostring(Gtotaal,2).." G")

  totalConv = Gtotaal*math.tofloat("9.81") --total g-kracht in m/s²
  print(math.tostring(totalConv,2).." m/s^2")
  
  accKmPHourPSec = Gtotaal*math.tofloat("35.3")
  print(math.tostring(accKmPHourPSec,2).." km/h/s")
------------------------------------------------------------------------
-----------------------------------------------------------------------
print(thema)
print(status)
if accKmPHourPSec <= minPnt and system.getTicks() >= timeoutRed then --groen 
 thema = "groen"
 print("groen") 
 theme.background = graphics.COLOR_GREEN

elseif accKmPHourPSec > minPnt and accKmPHourPSec < maxPnt and thema ~= "oranje" then --oranje
  thema = "oranje"
  print("oranje")
  theme.background = 0x00FF8C00 --oranje
    
elseif accKmPHourPSec >= maxPnt and thema ~= "rood" then --rood
    thema = "rood"
    theme.background = graphics.COLOR_RED
end
if status == "reset" and system.getTicks() >= delayLabels then
   if thema == "rood" and totalConv > highestNumber then
       highestNumber = totalConv
       accGLabel.label = math.tostring(Gtotaal,3).." G"
       accMps2Label.label = math.tostring(highestNumber,3).." m/s^2"
       accKmPHourPSecLabel.label = math.tostring(accKmPHourPSec,3).." km/h/s"
   timeoutRed = system.getTicks()+5000
elseif system.getTicks() >= timeoutRed then
  accGLabel.label = math.tostring(Gtotaal,3).." G"
  accMps2Label.label = math.tostring(totalConv,3).." m/s^2"
  accKmPHourPSecLabel.label = math.tostring(accKmPHourPSec,3).." km/h/s"
  highestNumber = totalConv
end  
      delayLabels = system.getTicks()+100  --min tijd voor schrijven van waarde naar labels (ms)
end
  
if status == "start" then
  if Gtotaal > math.tofloat(highestNumber) then
    highestNumber = Gtotaal
    accGLabel.label = math.tostring(Gtotaal,3).." G"
    totalConv = Gtotaal*math.tofloat("9.81")
    accMps2Label.label = math.tostring(totalConv,3).." m/s^2"
    accKmPHourPSec = Gtotaal*math.tofloat("35.3")
    accKmPHourPSecLabel.label = math.tostring(accKmPHourPSec,3).." km/h/s"
  end
end

function startButton.actionListener()
  if status == "reset" then
    status = "start"
    highestNumber = 0
    led.red = 0
    led.green = 20
  end 
end

function stopButton.actionListener()
  if status == "start" then
    print("stopped")
    status = "stop"
    led.green = 0
    led.red = 20
    reset.actionListener = reset
  end
end

function resetButton.actionListener()
  if status == "stop" then
    status = "reset"
    led.green = 0
    led.red = 0
  end
end
    
function backButton.actionListener()
    status = "reset"
    led.green = 0
    led.red = 0
    highestNumber = 0
    hoofdMenu()
end
mgf.repaint()
end
delayLabels = system.getTicks()
timeoutRed = system.getTicks()
mgf.setWidget(panel) 
t = timer.create( callback, 50)
t:start()

end

--****************************************************************************
--******************************   SETUP   ***********************************
--****************************************************************************
function setupFunctie()  --4    1km/h/s == 0,27777 m/s^2
  local titelLabel = mgf.createLabel( "settings" )
  local minPntLabel = mgf.createLabel("acc. min =" ..math.tostring(minPnt,2))
  local maxPntLabel = mgf.createLabel("acc. max =" ..math.tostring(maxPnt,2))
  local editButton = mgf.createButton("edit")
  local defaultButton = mgf.createButton("default")      
  local backButton = mgf.createButton("back")  
  local panel = mgf.createPanel()
  
  panel:addWidget(titelLabel, 0, 0, 240, 30)
  panel:addWidget(minPntLabel, 0, 30, 240, 60)
  panel:addWidget(maxPntLabel, 0, 90, 240, 60)
  panel:addWidget(editButton, 0, 150, 120, 130)
  panel:addWidget(defaultButton, 120, 150, 120, 130)  
  panel:addWidget(backButton, 0, 270, 240, 50)
  mgf.setWidget( panel ) 
  editButton.actionListener = editFunctie
  defaultButton.actionListener = defaultFunctie  
  backButton.actionListener = hoofdMenu
end
-------------------------------------------------------------------------------
function editFunctie()  --4.1
  nummerEditor = mgf.createNumberEditor()
  dialog = mgf.createDialog( nummerEditor, "groen eindpunt", mgf.ModeOKCancel )
  dialog.actionListener = dialogAction1
  mgf.setWidget(dialog, 0xFFFF, 0xFFFF, dialog.width, dialog.height)
end
function dialogAction1(widget, action)
  if(action == mgf.ActionDialogOK) then
    minPnt = nummerEditor.value
    editFunctie2()
  elseif(action == mgf.ActionDialogCancel) then
   setupFunctie()
  end
end

function editFunctie2()  --4.1
  nummerEditor.value = 0
  dialog = mgf.createDialog( nummerEditor, "rood beginpunt", mgf.ModeOKCancel )
  dialog.actionListener = dialogAction2
  mgf.setWidget(dialog, 0xFFFF, 0xFFFF, dialog.width, dialog.height)
end
function dialogAction2(widget, action)
  if(action == mgf.ActionDialogOK) then
    if nummerEditor.value < minPnt then  --de maximum waarde moet groter zijn dan de minimum waarde
      editFunctie2()                     --als dit niet zo is moet de waarde aangepast worden
    else
      maxPnt = nummerEditor.value        --de grenswaarde tussen rode en oranje zone is de nieuw ingegeven waarde
      setupFunctie()                     --terug naar de functie 'setupFunctie'
    end
  elseif(action == mgf.ActionDialogCancel) then  --indien er op cancel gedrukt wordt
    setupFunctie()								 --terug naar de functie 'setupFunctie'
  end
end
--------------------------------------------------------------------------
function defaultFunctie()  --zet waardes voor accelerometer terug in defaultwaarde
  minPnt = minPntDefault
  maxPnt = maxPntDefault
  setupFunctie()
end
-------------------------------------------------------------------------------
-------------------------------------------------------------------------------
function turnoffListener() --power off-button in het hoofdmenu, schakelt de screvle volledig uit
   gpio.setPins( gpio.PORTD, 0x0008 )
   gpio.setMode( gpio.PORTD, 0x0008, gpio.ModeOutputOD )
   gpio.resetPins( gpio.PORTD, 0x0008 )
end
-------------------------------------------------------------------------------------------

hoofdMenu()
