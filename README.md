# liebesbrunnen
The technical details and resources of the Liebesbrunnen

The artistic concept and a video of the Liebesbrunnen in action can be seen in the link https://vimeo.com/122400494.

The Liebesbrunnen is essentially a plastic (PVC maybe) tube with one meter high and ten centimeters of diameter. It have wires LEDs and rubbish inside. It produces sound and video while it is manipulated.

There's a web-cam attached in one of the extremities of it to record what goes on inside it. The video is processed by Processing, which adds delays and layers in a similar way as the sound does.
The sound is captured by a contact microphone made from a piezo disc and processed with Supercollider. Delays with altered pitches are added to the original sound.
Every time the Liebesbrunnen changes from the horizontal position to vertical or vice-versa the tilt sensor re-starts some synths with random parameters in order to the Liebesbrunnen to sounds differently, in a more or less unpredictable way. The tilt sensor state also changes the LEDs controlled by an Arduino program. The sound amplitude controls the brightness of the white LED.


## Here it goes the Arduino Programm:

```Arduino
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

```Processing
import processing.video.*;
Capture video;
int capW = 960;  //match camera resolution here
int capH = 720;
float yoff = 58.0;  // 2nd dimension of perlin noise
float delayTime;
int nDelayFrames = 96; // about 3 seconds
int currentFrame = nDelayFrames-1;
int currentFrame2;
int currentFrame3;
int numPixels;
int[] previousFrame;
PImage frames[];
//PImage framesHV[];
PImage framesV[];
//PImage videoFlipH;
PImage framesH[];
void setup() {
  size(capW, capH);  //set monitor size here
  frameRate(200);
  video = new Capture(this, capW, capH, "/dev/video0", 24);
  video.start();
  frames = new PImage [nDelayFrames];
  //framesHV = new PImage[nDelayFrames];
  framesV = new PImage[nDelayFrames];
  //videoFlipH = new PImage(video.width, video.height);
  framesH = new PImage[nDelayFrames];
  for (int i= 0; i<nDelayFrames; i++) {
    frames[i] = createImage(capW, capH, ARGB);
    //framesHV[i] = createImage(capW, capH, ARGB);
    framesV[i] = createImage(capW, capH, ARGB);
    framesH[i] = createImage(capW, capH, ARGB);
    numPixels = video.width * video.height;
    // Create an array to store the previously captured frame
    previousFrame = new int[numPixels];
    loadPixels();
  }
}
void draw() {
  float xoff = yoff; // Option #2: 1D Noise
  float delayTime = constrain(map(noise(yoff)*10, 1, 7, 1, 96), 1, 96);    // Option #2: 1D Noise
  yoff = (yoff+0.01) % nDelayFrames;
  nDelayFrames = int(delayTime);
  if (video.available()) {
    video.read();
    video.loadPixels();

    currentFrame = (currentFrame-1 + nDelayFrames) % nDelayFrames;
    currentFrame2 = (currentFrame +30)%nDelayFrames;  //+30= delay time. must be less than nDelayFrames
    currentFrame3 = (currentFrame +60)%nDelayFrames;  //+60= delay time. must be less than nDelayFrames
    for (int x = 0; x < video.width; x++) {
      for (int y = 0; y < video.height; y++) {
        // flip the image horizontally
        framesH[currentFrame].pixels[y*video.width + x] = video.pixels[y*video.width+(video.width-(x+1))];
        // flip the image both horizontally and vertically
        framesV[currentFrame].pixels[y*(video.width) + x] = video.pixels[(video.height - 1 - y)*(video.width) + x];
      }
    }
// desaturate the image
    for (int loc = 0; loc < width*height; loc++) {
      color currColor = framesH[currentFrame].pixels[loc];
      int currR = (currColor >> 16) & 0xFF;
      int currG = (currColor >> 8) & 0xFF;
      int currB = currColor & 0xFF;
      int newR = abs(int(currR+(currG+currB)/2)/2);
      int newG = abs(int(currG+(currR+currB)/2)/2);
      int newB = abs(int(currB+(currG+currR)/2)/2);
      framesH[currentFrame].pixels[loc] = 0xff000000 | (newR << 16) | (newG << 8) | newB;
    }
    updatePixels();
    framesH[currentFrame].updatePixels();
    framesV[currentFrame3].updatePixels();
    //framesH[currentFrame2].updatePixels();
    image(framesH[currentFrame], 0, 0, width, height);
    blend(framesV[currentFrame3], 0, 0, width, height, 0, 0, width, height, OVERLAY);  //try with ADD, DARKEST etc here. see blend help
    //blend(framesH[currentFrame2], 0, 0, width, height, 0, 0, width, height, OVERLAY);  // if I use this, the framerate drops dramatically. Why?

    updatePixels();
  }
    // println(nDelayFrames);
  //println(int(frameRate));
}```

