# Liebesbrunnen
#####Technical details and resources of the Liebesbrunnen

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
  float delayTime = constrain(map(noise(yoff)*10, 1, 7, 1, 96), 1, 96);
  yoff = (yoff+0.01) % nDelayFrames;
  nDelayFrames = int(delayTime);
  if (video.available()) {
    video.read();
    video.loadPixels();

    currentFrame = (currentFrame-1 + nDelayFrames) % nDelayFrames;
    // +30= delay time. must be less than nDelayFrames
    currentFrame2 = (currentFrame +30)%nDelayFrames;
    // +60= delay time. must be less than nDelayFrames
    currentFrame3 = (currentFrame +60)%nDelayFrames;
    for (int x = 0; x < video.width; x++) {
      for (int y = 0; y < video.height; y++) {
        // flip the image horizontally
        framesH[currentFrame].pixels[y*video.width + x] = 
        video.pixels[y*video.width+(video.width-(x+1))];
        // flip the image both horizontally and vertically
        framesV[currentFrame].pixels[y*(video.width) + x] = 
        video.pixels[(video.height - 1 - y)*(video.width) + x];
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
    //try with ADD, DARKEST etc here. see blend help
    blend(framesV[currentFrame3], 0, 0, width, height, 0, 0, width, height, OVERLAY);
    // if I use this next, the framerate drops dramatically. I would like to know why!
    //blend(framesH[currentFrame2], 0, 0, width, height, 0, 0, width, height, OVERLAY);

    updatePixels();
  }
    // println(nDelayFrames);
  //println(int(frameRate));
}
```

## ...and the Supercollider

```SuperCollider
s.freeAll;
s.quit;
s.options.memSize = 262144;
s = FreqScope.server.boot;
s.meter;
s.plotTree;
p = SerialPort("/dev/ttyUSB0", 115200, crtscts: true);
p.close;

(
~sourceG = Group.before;
~fxG = Group.tail;
~ampG =Group.after;

SynthDef(\dryS,	{ | /*in=[0,1],*/ out, gate = 1, release = 0.3, pitchDry |
	var  dry, pitchRate, grainSize, pitchDisp, timeDisp, env, sig, tilt=0;
	grainSize = 0.5;
	dry = AudioIn.ar([1,2]);
	env = Env.asr(1, 1, 1, \sin);

	pitchDry = PitchShift.ar( Compander.ar(dry, dry, 0.2, 1.5, 0.3, 0.002, 0.06, 0.75), 
	grainSize, ExpRand(5, 40).round*0.1, LFDNoise3.kr(0.054, 0.5, 2), LFNoise2.kr(0.0313, 
	grainSize-0.05, grainSize).lag3(0.24));

	sig = EnvGen.kr(env, gate, doneAction:2) * pitchDry;
	Out.ar(out, sig);
	};
).add;

SynthDef(\wetS,	{ |in, out, gate = 1|
	var wet, wetEnv, sig, env, winSize = {ExpRand(0.3, 3)}, pitchDisp = {ExpRand(0.2, 2.0)};

	env = Env.asr(1, 1, 2, \sin);
	sig = HPF.ar(In.ar(in, 2), 192);

	wet = Splay.ar(PitchShift.ar(LocalIn.ar(2) + sig, ExpRand(0.3, 3), 
	LFNoise2.kr(0.0334, 0.2, 2).lag3(0.321), ExpRand(0.2, 2.0), ExpRand(0.5, 2.0) ));
	wet = OnePole.ar(wet, 0.47);
	wet = OnePole.ar(wet, -0.079);

	LocalOut.ar(wet * LFNoise2.ar(0.0187, 0.1, 1).lag3(0.587));
	wet = BPeakEQ.ar(wet, 20000, 1, 12);
	wetEnv = EnvGen.kr(env, gate, doneAction:2) * wet;
	Out.ar(out, wetEnv);
};).add;

SynthDef(\revS,	{ |in, out|
	var del, sig, delTime, decTime, ampR;
	sig = In.ar(in, 4);
	del = LocalIn.ar(2)+sig;
	del = Splay.ar(del.reverse*0.3);
	4.do {
		del = AllpassC.ar(del, 12, ExpRand(0.2, 6.0), ExpRand(3, 24));
	};


	ampR = Amplitude.kr(Mix.ar(sig));
	del = Compander.ar(del, Pan2.ar(In.ar(in, 4)), 0.25, 1, 1/4, 0.002, 0.6);

	LocalOut.ar(del * LFNoise2.ar(0.0107, 0.2, 1).lag2(0.3));

	Out.ar(out, del);
};).add;

SynthDef(\amp,	{
	arg in, in2, in3, out;
	var inNum=3, att=0.01, rel=0.2, rate=48,sig,trig,amp;
	in = In.ar(in,2);
	in2 = In.ar(in2,2);
	in3 = In.ar(in3,2);
	sig = Median.ar(11, 
		(BHiPass.ar(Mix.ar(in+(in2/2)+(in3*2)), 
		100, 1).linlin(0.0001, 0.08, 5.0, 255.0).lag3(0.48)));
	trig = Impulse.kr(rate);
	amp = Amplitude.kr(sig, att, rel, inNum.reciprocal);
	SendReply.kr(trig, '/amp', [amp]);  // Get ampl. on the server and send to the language
	Out.ar(out, in+(in2/2)+in3);
	};
).add;
//s.prepareForRecord;
)
(
//s.record;
x = Synth(\revS, [\in, [10, 11, 12, 13], \out, 14], ~fxG, \addToTail);
z = Synth(\amp, [\in, [10, 11], \in2, [12, 13],\in3, [14, 15],\out, 0], ~ampG, \addToTail);

p = SerialPort("/dev/ttyUSB0", 115200, crtscts: true);

OSCdef(\receiveAmp, {|msg|
	var amp = (msg[3]).asInteger;
	var tilt = (msg[0]);
	//p.read(msg[0]).postln;
	p.put(msg[3].linlin(0, 1, 0, 3, 255).asInteger);
}, '/amp' );

Routine.run({
	var byte;
	inf.do{
		while({byte= p.read; byte.notNil}, {
			if (byte == 1)
			{
				if(t.notNil){
					t.set(\gate, 0);
				};
				if(w.notNil){
					w.set(\gate, 0);
				};
				u = Synth(\dryS, [\in, [1,2], \out, 10], ~sourceG);
				v = Synth(\wetS, [\in, [10, 11], \out, 12], ~fxG);
			};
			if(byte == 0)
			{
				if(u.notNil){
					u.set(\gate, 0);
				};
				if(v.notNil){
					v.set(\gate, 0);
				};
				t = Synth(\dryS, [\in, [1,2], \out, 10], ~sourceG);
				w = Synth(\wetS, [\in, [10, 11], \out, 12], ~fxG);
			};
		});
		0.01.wait;
	};
});
CmdPeriod.doOnce({p.close});
)
```

