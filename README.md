# Adafruit NeoPixel Library for Pi PICO C/C++ SDK

Forked from [https://github.com/martinkooij/pi-pico-adafruit-neopixels](martinkooij/pi-pico-adafruit-neopixels)

Port of the Adafruit Library to PI PICO in a C/C++ SDK environment.
Build according to SDK standards as an INTERFACE library

Friendly usage of pio statemachine implementations: Any neopixels object seizes sm and pio program space in a friendly cooperative way 

This fork has been changed so it can be cloned or submoduled directly into a subdirectory of your pico project.

## Installation

1. Clone into a subdirectory (or submodule) this into your project
2. After your `add_executable() declaration` add `add_subdirectory(<where you cloned this repo to>)` (so if you cloned into `pico_neopixels` that line would be `add_subdirectory(pico_neopixels)`)
3. Add `pico_neopixel` to your `target_link_libraries()` declaration (it will be called this regardless of where you submodule into)
4. There is no 4

If the PICO SDK is installed according to the guidelines on the raspberry pi pico site the following should work:

````
cd pi-pico-adafruit-nexopixels/example
mkdir build && cd build
cmake ..
make
````
or on windows
````
cd pi-adafruit-neopixels/example
mkdir build && cd build
cmake .. -G "NMake Makefiles"
nmake
````

## Library documentation

See the Adafruit Neopixel library documentation on the various resources. Note that the <code>begin()</code> method is only kept for compatibility reasons but has no effect. On the pico a dedicated pio statemachine is seized and started at the very first <code>show()</code> command of the object. 

## Resources (specific to the pico port)
Each object seizes a pio/sm pair at the moment of the first call to the "show" method. It uses the SDK calls <code>pio_claim_unused_sm</code>, <code>pio_can_add_program</code> to find unused statemachines and unused instruction space on the pio's. 

In the destructor the library releases the claimed resources (the sm, and if no other statemachine are using it anymore also removes the program instruction space on the designated pio by using <code>pio_remove_program</code>, if there are no other sm serving neopixel objects on that pio). 

The destructor is called if called explicitly when the object was constructed with "new" or implicitly when it goes out of scope in a function. These are all quite special cases. When defined before main it will not go out of scope in the programming lifetime. This would be the normal use case.

There is a maximum of 8 neopixel strands, if pio and sm's are not used for any other purposes in your project. Strands can be assigned to any GPIO pin of the pico. 

## Advanced Brightness Control (specific to the pico port)
A method <code>myPixels.setBrightnessFunctions(fp_r, fp_g, fp_b, fp_w)</code> has been added that can be called at all times. The method takes 4 pointers functions that translate 8-bit pixel values to dimmed 8-bit pixel values. From that moment on extra memory is allocated to remember the (exact) original color set values of the pixels, and the dimmed values are stored in the memory that is used for display. A myPixels.show() will push the values out to the physical string. All actions are now done relative to the brightnessfunctions installed. 

To make things easier a function <code>neopixels_gamma8(uint8_t val)</code> has been added outside the object header that can be used before a Neo_Pixel object is not in scope. 
 
Extra memory is *only* created if a call to setBrightnessFunctions is made. Otherwise *no* extra memory is allocated to keep the original values. For compatibility purposes the original setBrightness with the original somewhat broken behavior is kept unchanged, but it is ill advised to use both at the same time.  
 
A typical usage would be:
````
uint8_t halfdimmed(uint8_t val) {
  return ((val * neopixels_gamma8(126))>>8) ;
}

uint8_t notdimmed(uint8_t val) {
  return val;
}

Adafruit_NeoPixel myPixels(60, 7, NEO_GRB + NEO_KHZ800);

int main() {
  int original_value ;
  for (int i = 0 ; i < 60 , i++) myPixels.setPixelColor(i,266,49,99) ; //set nice colors
  while (1) {
    myPixels.setBrightnessFunctions(notdimmed,notdimmed,notdimmed,notdimmed) // do not dim any colors
	myPixels.show() ;
	sleep_ms(5000) ;
    myPixels.setBrightnessFunctions(halfdimmed,halfdimmed,halfdimmed,halfdimmed) // dim all colors on the string to half brightness
	myPixels.show() ;
    original_value = myPixels.getColor(0)	; // this would give the value corresponding with 266,49,99 even though the string is dimmed
	sleep_ms(5000) ;
	}
}
````
