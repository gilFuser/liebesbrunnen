# liebesbrunnen
The technical details and resources of the Liebesbrunnen

The artistic concept and a video of the Liebesbrunnen in action can be seen in the link https://vimeo.com/122400494.

The Liebesbrunnen is essentially a plastic (PVC maybe) tube with one meter high and ten centimeters of diameter. It have wires LEDs and rubbish inside. It produces sound and video while it is manipulated.

There's a web-cam attached in one of the extremities of it to record what goes on inside it. The video is processed by Processing, which adds delays and layers in a similar way as the sound does.
The sound is captured by a contact microphone made from a piezo disc and processed with Supercollider. Delays with altered pitches are added to the original sound.
Every time the Liebesbrunnen changes from the horizontal position to vertical or vice-versa the tilt sensor re-starts some synths with random parameters in order to the Liebesbrunnen to sounds differently, in a more or less unpredictable way. The tilt sensor state also changes the LEDs controlled by an Arduino program. The sound amplitude controls the brightness of the white LED.


## Here it goes the Arduino Programm:

```c_cpp
//  For more informations about this stetch, see Arduino/File/Examples/Basics/Digital/Debounce
//  and Arduino/File/Examples/Basics/Communication/Serial... one of those. I need to check wich one
// give names to the pins, regarding the LEDs colors
const int tiltPin = 2;
const int blue2 = 3;
const int yellow = 5;
const int red3 = 6;
const int red = 9;
const int white = 10;
const int green = 11;
const int red2 = 12;
const int ledPin = 13;

//initial values for some LEDs brightness
int briG = 0;
int briW = 255;
int briR = 127;
int briR3 = 100;
int value;
int valueR = 127;
int valueR3 = 100;

long time=0;  //for the timer. In order not to use 'wait'
// for fading exponentially - much better than linearly
int periode = 3000;
int displace = 1515;

int tiltState = 0;  // current state of the tilt sensor
int lastTiltState = LOW;  // previous state of the tilt sensor
int ledState = HIGH;  // visual feedback of the tilt sensor state

//to debouncing - discard fast changings on the tilt sensor state
long lastDebounceTime = 0;  // the last time the output pin was toggled
long debounceDelay = 50;    // the debounce time; increase if the output flickers

void setup() {
  // this is very specific related to the layout from the pins in the Arduino FIO  
  pinMode(red, OUTPUT);
  pinMode(green, OUTPUT);
  pinMode(white, OUTPUT);
  pinMode(red2, OUTPUT);
  pinMode(yellow, OUTPUT);
  pinMode(red3, OUTPUT);
  pinMode(blue2, OUTPUT);
  pinMode(tiltPin, INPUT_PULLUP);
  pinMode(ledPin, OUTPUT);
  digitalWrite(ledPin, ledState);

  Serial.begin(115200);  // this is maybe too fast
}

void loop() {
  time = millis();

  int reading = digitalRead(tiltPin);
  if (reading != lastTiltState) {  // if a changing in the tilt sensor occurs
    lastDebounceTime = millis();  // get the instant of the changing
  }

  if ((millis() - lastDebounceTime) > debounceDelay) {  /* if the period in between two
   changes in the tilt sensor is bigger than the given 'time threshold'*/
    if (reading != tiltState) {  // and if the tilt sensor's state is indeed changing
      tiltState = reading;  // mark the actual state
      Serial.write(reading);  // print the actual state
      if (tiltState == HIGH) {
        ledState = !ledState;
      }
    }
  }
  // set the LED:
  digitalWrite(ledPin, ledState);

  // save the reading.  Next time through the loop,
  // it'll be the lastTiltState:
  lastTiltState = reading;

  // read the numbers that come from Supercollider and use it to modulate the brightness
  if(Serial.available()) {
    byte val= Serial.read();
    analogWrite(white, val);
  }
  //  fades in and outs, in loops. Maybe there are some gambiarras on these pieces of code, 
  //  but it's working ok.
  if (tiltState == LOW) {  // if the Liebesbrunnen is in a more horizontal position:...
    value = 16*sin(PI/periode*(time/24));
    if(value < 0 || value > 20){
      value = value * -1;
    }
    briG = value;

    if (valueR3 > 30) {
      valueR3 = 128+127*cos(2*PI/periode*(displace-time)/6);
    } 
    else if (valueR3 <= 1) { 
      valueR3 = 30; 
    };
    briR3 = valueR3;

    valueR = map(128+127*sin(PI/periode*(time/1.6)), 0, 255, 63, 192);
    if(valueR <= 0){
      value = value * -1;
    };
    briR = valueR;

    analogWrite(red, briR);  
    analogWrite(green, briG);
    digitalWrite(red2, HIGH);  //roxo
    analogWrite(yellow, 255);
    analogWrite(red3, briR3);  // laranja
    analogWrite(blue2, 8);
    ledState = !ledState;
  }
  else {  // if the Liebesbrunnen is positioned more vertically:...
    if (valueR < 122) {
      valueR = 128-127*cos(2*PI/periode*(displace-time)/6);
    }  
    else if (valueR >= 122) {
      valueR = 122;
    };
    briR3 = valueR;
    analogWrite(red3, briR3);

    briG = 16*sin(2*PI/periode*(time)/8);

    if(briG < 0){
      briG = briG * -1;
    }
    // Serial.println(briG);  // this is for debugging and check how the number are behaving, and
    // MUST be commented while running Supercollider, otherwise every time a variable is printed
    // a Synth will start.
    analogWrite(green, briG);
    analogWrite(red, briR);

  }
}
```

## The Processing part:

