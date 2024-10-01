#include <FastLED.h>

#define NUM_LEDS 12
#define DATA_PIN 6
CRGB leds[NUM_LEDS];

const int buttonPin = 2;

enum State { ATTRACTION_MODE, SPINNING, SLOWDOWN, BLINKING, WIN_EFFECT };
State currentState = ATTRACTION_MODE;

unsigned long lastButtonPressTime = 0;
unsigned long lastAttractionModeTime = 0;
bool buttonPressed = false;
bool blinking = false;
int stopLED = -1;
unsigned long spinStartTime;
int spinSpeed = 20;  // Adjusted spin speed

void setup() {
  FastLED.addLeds<WS2811, DATA_PIN, GRB>(leds, NUM_LEDS);
  pinMode(buttonPin, INPUT_PULLUP);
  randomSeed(millis());
  lastAttractionModeTime = millis();
}

void loop() {
  switch (currentState) {
    case ATTRACTION_MODE:
      updateAttractionMode();
      if (digitalRead(buttonPin) == LOW && millis() - lastButtonPressTime > 200) {
        lastButtonPressTime = millis();
        currentState = SPINNING;
      }
      break;
    case SPINNING:
      updateSpinningLEDs();
      if (digitalRead(buttonPin) == LOW || millis() - spinStartTime > 5000) {
        currentState = SLOWDOWN;
      }
      break;
    case SLOWDOWN:
      startSlowdown();
      currentState = BLINKING;
      break;
    case BLINKING:
      blinkStoppedLED();
      currentState = WIN_EFFECT;
      break;
    case WIN_EFFECT:
      if (digitalRead(buttonPin) == LOW) {
        // Immediately transition to spinning if button is pressed
        currentState = SPINNING;
        break;
      }
      winEffect();
      currentState = ATTRACTION_MODE;
      break;
  }
}

void updateAttractionMode() {
  static unsigned long lastUpdate = 0;
  unsigned long currentTime = millis();
  if (currentTime - lastUpdate >= 100) {
    lastUpdate = currentTime;
    for (int i = 0; i < NUM_LEDS; i++) {
      leds[i] = CHSV(random(255), 255, 255);
    }
    FastLED.show();
  }
}

void startSpinning() {
  FastLED.clear();
  FastLED.show();
  spinStartTime = millis();
}

void updateSpinningLEDs() {
  unsigned long currentTime = millis();
  int ledIndex = ((currentTime - spinStartTime) * spinSpeed / 100) % NUM_LEDS;
  updateLEDs(ledIndex);
  FastLED.show();
}

void startSlowdown() {
  unsigned long duration = random(3000, 5000);
  unsigned long endTime = millis() + duration;
  while (millis() < endTime) {
    unsigned long elapsed = millis() - spinStartTime;
    int ledIndex = (elapsed * spinSpeed / 100) % NUM_LEDS;
    updateLEDs(ledIndex);
    FastLED.show();
    delay(10);
  }
  stopLED = random(NUM_LEDS);
  updateLEDs(stopLED);
  FastLED.show();
}

void blinkStoppedLED() {
  unsigned long startTime = millis();
  unsigned long blinkDuration = 5000;
  blinking = true;
  while (millis() - startTime < blinkDuration) {
    if (digitalRead(buttonPin) == LOW) {
      blinking = false;
      return;
    }
    leds[stopLED] = CHSV(random(255), 255, 255);
    FastLED.show();
    delay(250);
    leds[stopLED] = CRGB::Black;
    FastLED.show();
    delay(250);
  }
  blinking = false;
}

void winEffect() {
  // Distribute a rainbow across the solid LEDs (all except the winning LED)
  for (int i = 0; i < NUM_LEDS; i++) {
    if (i != stopLED) {
      leds[i] = CHSV(i * (255 / NUM_LEDS), 255, 255);  // Rainbow effect
    }
  }

  // Pulse the winning LED through the full spectrum
  unsigned long pulseStartTime = millis();
  while (millis() - pulseStartTime < 5000) {  // Pulse for 5 seconds
    if (digitalRead(buttonPin) == LOW) {
      // Transition to spinning immediately if button is pressed
      return;
    }
    leds[stopLED] = CHSV((millis() / 10) % 255, 255, 255);  // Change hue over time
    FastLED.show();
    delay(10);
  }
}

void updateLEDs(int index) {
  FastLED.clear();
  if (index >= 0 && index < NUM_LEDS) {
    leds[index] = CHSV(random(255), 255, 255);
  }
  FastLED.show();
}
