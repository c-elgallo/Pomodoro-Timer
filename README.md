# Pomodoro-Timer
This is the most up to date Pomodoro Timer Code for the Spring of 2025 MESA Project led by Jessica Mora Jacinto

// Pomodoro Timer with Multi-State Support using millis()
// Includes Adafruit OLED Display for visual feedback

#include <SPI.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH110X.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C

Adafruit_SH1106G display = Adafruit_SH1106G(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int greenLEDPin = 9;
const int redLEDPin = 11;
const int buttonMiddlePin = 3;
const int buttonLeftPin = 2;
const int buttonRightPin = 4;
const int buzzerPin = 5;

enum State {
  MainMenu,
  Working,
  Break,
  WorkingEdit,
  BreakEdit
};
State currentState = MainMenu;

unsigned long workDuration = 1 * 60000;  // 25 minutes in milliseconds
unsigned long breakDuration = 5 * 60000;  // 5 minutes in milliseconds
unsigned long timerStartMillis = 0;

bool lastMiddleState = HIGH;
bool lastLeftState = HIGH;
bool lastRightState = HIGH;

unsigned long currentMillis = 0;
const unsigned long debounceDelay = 50;
unsigned long lastDebounceMiddle = 0, lastDebounceLeft = 0, lastDebounceRight = 0;

void setup() {
  pinMode(greenLEDPin, OUTPUT);
  pinMode(redLEDPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);

  pinMode(buttonMiddlePin, INPUT_PULLUP);
  pinMode(buttonLeftPin, INPUT_PULLUP);
  pinMode(buttonRightPin, INPUT_PULLUP);

  Serial.begin(9600);
  display.begin(SCREEN_ADDRESS, true);
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SH110X_WHITE);
  display.setCursor(0, 0);
  display.println("Pomodoro Timer");
  display.display();
  delay(1000);
}

void loop() {
  currentMillis = millis();
  handleButtons();
  updateDisplay();

  if (currentState == Working || currentState == Break) {
    unsigned long duration = (currentState == Working) ? workDuration : breakDuration;
    unsigned long elapsed = currentMillis - timerStartMillis;
    unsigned long remaining = (duration > elapsed) ? (duration - elapsed) : 0;

    // If time is up, transition to the next state
    if (remaining == 0) {
      if (currentState == Working) {
        currentState = Break;
        timerStartMillis = currentMillis;
        digitalWrite(greenLEDPin, LOW);
        digitalWrite(redLEDPin, HIGH);
        tone(buzzerPin, 1000, 500);
      } else if (currentState == Break) {
        currentState = MainMenu;
        digitalWrite(redLEDPin, LOW);
        tone(buzzerPin, 1500, 500);
      }
    }
  }
}

void handleButtons() {
  bool middlePressed = digitalRead(buttonMiddlePin) == LOW;
  bool leftPressed = digitalRead(buttonLeftPin) == LOW;
  bool rightPressed = digitalRead(buttonRightPin) == LOW;

  if (middlePressed && lastMiddleState == HIGH && (millis() - lastDebounceMiddle > debounceDelay)) {
    lastDebounceMiddle = millis();
    handleMiddlePress();
  }
  if (leftPressed && lastLeftState == HIGH && (millis() - lastDebounceLeft > debounceDelay)) {
    lastDebounceLeft = millis();
    handleLeftPress();
  }
  if (rightPressed && lastRightState == HIGH && (millis() - lastDebounceRight > debounceDelay)) {
    lastDebounceRight = millis();
    handleRightPress();
  }

  lastMiddleState = !middlePressed;
  lastLeftState = !leftPressed;
  lastRightState = !rightPressed;
}

void handleMiddlePress() {
  if (currentState == MainMenu) {
    currentState = Working;
    timerStartMillis = currentMillis;  // Start the timer when working
    digitalWrite(greenLEDPin, HIGH);
  } else if (currentState == Working || currentState == Break) {
    currentState = MainMenu;
    digitalWrite(greenLEDPin, LOW);
    digitalWrite(redLEDPin, LOW);
  } else {
    currentState = MainMenu;
  }
}

void handleLeftPress() {
  switch (currentState) {
    case MainMenu:
      currentState = BreakEdit;
      break;
    case WorkingEdit:
      if (workDuration >= 1 * 60 * 60) workDuration -= 60 * 60;
      break;
    case BreakEdit:
      if (breakDuration >= 1 * 60 * 60) breakDuration -= 60 * 60;
      break;
  }
}

void handleRightPress() {
  switch (currentState) {
    case MainMenu:
      currentState = WorkingEdit;
      break;
    case WorkingEdit:
      workDuration += 60 * 60;
      break;
    case BreakEdit:
      breakDuration += 60 * 60;
      break;
  }
}

void formatAndPrintTime(const char* label, unsigned long durationMs) {
  unsigned long durationSec = durationMs / 60;  // Convert to seconds
  int minutes = durationSec / 1000;  // Convert to minutes
  int seconds = durationSec % 1000;  // Get remaining seconds

  display.print(label);
  display.print(minutes);
  display.print(":");
  if (seconds < 10) display.print("0");
  display.println(seconds);
}

void updateDisplay() {
  display.clearDisplay();
  display.setCursor(0, 0);
  display.setTextSize(1);

  display.print("State: ");
  switch (currentState) {
    case MainMenu: display.println("Main Menu"); break;
    case Working: display.println("Working"); break;
    case Break: display.println("Break"); break;
    case WorkingEdit: display.println("Edit Work"); break;
    case BreakEdit: display.println("Edit Break"); break;
  }

  formatAndPrintTime("Work: ", workDuration);
  formatAndPrintTime("Break: ", breakDuration);

  if (currentState == Working || currentState == Break) {
    unsigned long duration = (currentState == Working) ? workDuration : breakDuration;
    unsigned long elapsed = currentMillis - timerStartMillis;
    unsigned long remaining = (duration > elapsed) ? (duration - elapsed) : 0;

    int minutes = (remaining / 1000) / 60;  // Convert remaining time to minutes
    int seconds = (remaining / 1000) % 60;  // Convert remaining time to seconds

    display.setCursor(0, 40);
    display.setTextSize(2);
    display.print(minutes);
    display.print(":");
    if (seconds < 10) display.print("0");
    display.print(seconds);

    int progressBarWidth = 100;
    int barX = 14, barY = 60;
    int fillWidth = map(elapsed, 0, duration, 0, progressBarWidth);
    display.drawRect(barX, barY, progressBarWidth, 4, SH110X_WHITE);
    display.fillRect(barX, barY, fillWidth, 4, SH110X_WHITE);
  }

  display.display();
}
