 //прерываний не будет. прерывания поддерживают только два порта :(  
  
  #include <LiquidCrystal_I2C.h>
  #include <Wire.h>
  
  const int buttonPinUp = 3;       // Пин кнопки увеличения температуры
  const int buttonPinDown = 8;     // Пин кнопки уменьшения температуры
  const int buttonPinStart = 2;    // Пин кнопки запуска нагрева

  const int heaterPin1 = 5;        // Пин первого нагревательного элемента
  const int heaterPin2 = 6; 

  const int tempSensor35DzOldPin = A3;   
  const int tempSensor35DzNewPin = A0;   

  const unsigned long errorTemperatureDelta = 5;
  const unsigned long ETD = errorTemperatureDelta * 1023 / 500;
  const unsigned long errorTemperatureValue = 105;
  const unsigned long ETV = errorTemperatureValue * 1023 / 500;

  bool heatingActive = false; 
  bool keepConstantTemperature = false;

  unsigned int targetTemp = 30;              
  unsigned int currentTemp = 0;
  unsigned int hysteresis = 1;               

  unsigned long lastUpdate = 0; 
  const int updateInterval = 1000; 

  LiquidCrystal_I2C LCD(0x27, 16, 2);

void setup() {
    pinMode(13, OUTPUT);

    pinMode(buttonPinUp, INPUT);
    pinMode(buttonPinDown, INPUT);
    pinMode(buttonPinStart, INPUT);
    pinMode(heaterPin1, OUTPUT);
    pinMode(heaterPin2, OUTPUT);
    digitalWrite(heaterPin1, HIGH);
    digitalWrite(heaterPin2, HIGH);

    Serial.begin(9600);
    Serial.println("setup ready");

    LCD.init(); 
    LCD.backlight();
   
    LCD.setCursor(1, 0);
    LCD.print("Welcome");
  
    LCD.setCursor(1, 1); 
    LCD.print("back"); 
    LCD.display();
    delay(3000);
    LCD.clear();
}

void loop() {
  int upStat = digitalRead(buttonPinUp);
  int downStat = digitalRead(buttonPinDown);
  int startStat = digitalRead(buttonPinStart);

  unsigned int sensor35OldValue = analogRead(tempSensor35DzOldPin);
  unsigned int sensor35NewValue = analogRead(tempSensor35DzNewPin);

  validation(sensor35OldValue, sensor35NewValue);

  
  int temp35Old = sensor35OldValue * 500 / 1024;
  int temp35New = sensor35NewValue * 500 / 1024;
  currentTemp = (temp35Old + temp35New) / 2;

  if (upStat == LOW && targetTemp < 125) {
    targetTemp += 5;
    Serial.println("Target temp: ");
    Serial.println(targetTemp);
    Serial.println();
    Serial.println("Current temp: ");
    Serial.println(currentTemp);
    Serial.println();
    delay(500);
  } 

  if (downStat == LOW && targetTemp > 25) {
    targetTemp -= 5;
    Serial.println("Target temp: ");
    Serial.println(targetTemp);
    Serial.println();
    Serial.println("Current temp: ");
    Serial.println(currentTemp);
    Serial.println();
    delay(500);
  }

  if (startStat == LOW){
    keepConstantTemperature = !keepConstantTemperature;
    Serial.println("Temperature keeping is ");
    Serial.println((keepConstantTemperature) ? "true\n" : "false\n");

    delay(500);
  }

  if (millis() - lastUpdate >= updateInterval)
  {
    lastUpdate = millis();
    updateScreen(sensor35OldValue, temp35Old, sensor35NewValue, temp35New, currentTemp);    
  }

  if (keepConstantTemperature)
  {
    if (currentTemp < targetTemp - hysteresis) {
      heatingActive = true;
      digitalWrite(heaterPin1, LOW);
      digitalWrite(heaterPin2, LOW);
      delay(500);        
    } else if (currentTemp > targetTemp + hysteresis) {
      heatingActive = false;
      digitalWrite(heaterPin1, HIGH);
      digitalWrite(heaterPin2, HIGH);
      delay(500);
    }  
  } 

}

void updateScreen(int sensor35OldValue, int temp35Old, int sensor35NewValue, int temp35New, int currentTemp)
{
  // Serial.println("35DZ old pin input value");
  // Serial.println(sensor35OldValue);

  // Serial.println("35DZ old temperature");
  // Serial.println(temp35Old);
  // Serial.println();
  // Serial.println("35DZ new pin input value");
  // Serial.println(sensor35NewValue);

  // Serial.println("35DZ new temperature");
  // Serial.println(temp35New);
  // Serial.println();
  // Serial.println();

  LCD.clear();
  LCD.setCursor(0, 0); 
  LCD.print("Target temp ");
  LCD.print(targetTemp); 

  LCD.setCursor(0, 1); 
  LCD.print("Current ");

  LCD.print(currentTemp);
  LCD.print(" ");
  LCD.print(temp35Old);
  LCD.print(" ");
  LCD.print(temp35New);
}        

void validation(int sensor35OldValue, int sensor35NewValue)
{
  bool readingIisValid = true;
  if (abs(sensor35NewValue - sensor35OldValue) > ETD)
  {
    LCD.clear();
    LCD.setCursor(0, 0); 
    LCD.print("Some sensor err!");
    LCD.setCursor(0, 1);
    LCD.print("Fix and restart");

    readingIisValid = false;

    if (heatingActive)
    {
      digitalWrite(heaterPin1, HIGH);
      digitalWrite(heaterPin2, HIGH);      
    }
  }

  if (sensor35OldValue > ETV)
  {
    LCD.clear();
    LCD.setCursor(0, 0); 
    LCD.print("Left sensor err!");
    LCD.setCursor(0, 1);
    LCD.print("Fix and restart");

    digitalWrite(heaterPin1, HIGH);
    digitalWrite(heaterPin2, HIGH);

    readingIisValid = false;

    if (heatingActive)
    {
      digitalWrite(heaterPin1, HIGH);
      digitalWrite(heaterPin2, HIGH);
    }
  }

  if (sensor35NewValue > ETV)
  {
    LCD.clear();
    LCD.setCursor(0, 0); 
    LCD.print("Right sensor err!");
    LCD.setCursor(0, 1);
    LCD.print("Fix and restart");

    readingIisValid = false;

    if (heatingActive)
    {
      digitalWrite(heaterPin1, false);
      digitalWrite(heaterPin2, false);
    }
  }

  if(!readingIisValid)
    abort();
}