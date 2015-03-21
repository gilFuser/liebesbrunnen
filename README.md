# liebesbrunnen
The technical details and resources of the Liebesbrunnen

The artistic concept and a video of the Liebesbrunnen in action can be seen in the link https://vimeo.com/122400494.

The Liebesbrunnen is essentially a plastic (PVC maybe)tube with one meter high and ten centimeters of diameter. It have wires LEDs and rubbish inside. It produces sound and video while it is manipulated.

There's a web-cam attached in one of the extremities of it to record what goes on inside it. The video is processed by Processing, which adds delays and layers in a similar way as the sound does.
The sound is captured by a contact microphone made from a piezo disc and processed with Supercollider. Then it have the pitch altered and layers from delays are added to it.
Every time the Liebesbrunnen changes from an horizontal position to a vertical one or vice-versa, the tilt sensor starts some synths with random parameters, so the Liebesbrunnen sounds differently, in a more or less unpredictable way. The tilt sensor state also changes the LEDs controlled by an Arduino program. The sound amplitude controls the brightness of the white LED.
