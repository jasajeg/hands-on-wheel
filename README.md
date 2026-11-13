# hands-on-wheel
#include <CapacitiveSensor.h>

/*
 * Capacitive "Hands-On-Wheel" Detection Circuit
 * Author: Jasakan Jegatheesvaran
 * Description: Embedded C logic for a microcontroller to detect physical driver engagement
 * using capacitive copper tape sensors. Includes a "dead man's switch" timer
 * to mimic industry-standard autonomous vehicle safety protocols (Autopilot Nag).
 */

// Pin Definitions
#define SEND_PIN 4            // Common send pin connected to a high-value resistor (e.g., 1M Ohm)
#define RECEIVE_PIN_LEFT 2    // Receive pin connected to left copper tape zone
#define RECEIVE_PIN_RIGHT 3   // Receive pin connected to right copper tape zone
#define LED_BLUE 8            // Indicator for Autopilot Active
#define LED_RED 9             // Indicator for Warning / Disengaged
#define BUZZER 10             // Audio alert for Steering Wheel Nag

// Thresholds and Timers
#define CAP_THRESHOLD 1000    // Capacitance threshold (needs calibration based on resistor/tape size)
#define MAX_TIMEOUT 15000     // 15 seconds max before forced disengagement
#define WARNING_TIMEOUT 5000  // 5 seconds remaining triggers the "Nag"

// Initialize Capacitive Sensors (Send Pin, Receive Pin)
CapacitiveSensor leftSensor = CapacitiveSensor(SEND_PIN, RECEIVE_PIN_LEFT);
CapacitiveSensor rightSensor = CapacitiveSensor(SEND_PIN, RECEIVE_PIN_RIGHT);

unsigned long lastTouchTime = 0;
bool autopilotEngaged = true;

void setup() {
    Serial.begin(9600);
    
    // Set output pins
    pinMode(LED_BLUE, OUTPUT);
    pinMode(LED_RED, OUTPUT);
    pinMode(BUZZER, OUTPUT);
    
    // Initial State
    lastTouchTime = millis();
    digitalWrite(LED_BLUE, HIGH);
    digitalWrite(LED_RED, LOW);
    
    Serial.println("System Initialized: Autopilot Engaged.");
}

void loop() {
    // 1. Check if system has been locked out
    if (!autopilotEngaged) {
        // Failure Condition: Locked out state until hardware reset
        digitalWrite(LED_BLUE, LOW);
        digitalWrite(LED_RED, HIGH);
        tone(BUZZER, 1000); // Continuous loud buzz
        return; // Halt further loop execution
    }

    // 2. Read Sensors (30 samples for stability)
    long leftVal = leftSensor.capacitiveSensor(30);
    long rightVal = rightSensor.capacitiveSensor(30);
    
    // 3. Calculate Timing
    unsigned long currentTime = millis();
    unsigned long timeSinceLastTouch = currentTime - lastTouchTime;
    unsigned long timeRemaining = (MAX_TIMEOUT > timeSinceLastTouch) ? (MAX_TIMEOUT - timeSinceLastTouch) : 0;

    // 4. Determine if hands are on the wheel
    bool handsOnWheel = (leftVal > CAP_THRESHOLD) || (rightVal > CAP_THRESHOLD);

    // 5. State Machine Logic
    if (handsOnWheel) {
        // Reset Condition
        lastTouchTime = currentTime;
        digitalWrite(LED_BLUE, HIGH);
        digitalWrite(LED_RED, LOW);
        noTone(BUZZER);
        Serial.println("Driver Engaged: Timer Reset.");
    } else {
        if (timeRemaining == 0) {
            // Failure Condition
            autopilotEngaged = false;
            Serial.println("CRITICAL: Autopilot Disengaged due to driver inattention.");
        } else if (timeRemaining <= WARNING_TIMEOUT) {
            // Warning Condition (Nag)
            digitalWrite(LED_BLUE, (currentTime / 250) % 2); // Flash blue LED rapidly
            
            // Beep the buzzer on and off
            if ((currentTime / 500) % 2) {
                tone(BUZZER, 500);
            } else {
                noTone(BUZZER);
            }
            Serial.print("WARNING: Please place hands on the wheel. Time remaining: ");
            Serial.println(timeRemaining);
        } else {
            // Normal Countdown (Hands off, but still in safe window)
            digitalWrite(LED_BLUE, HIGH);
            digitalWrite(LED_RED, LOW);
            noTone(BUZZER);
        }
    }
    
    delay(100); // Small delay to prevent serial flooding
}

<img width="1320" height="762" alt="blueled" src="https://github.com/user-attachments/assets/f9eccd01-de81-4783-8984-0b06c2db4530" />

Blue LED: Autopilot Active & Driver Prompt
System Logic & State Machine Behavior:

Solid ON: Indicates nominal operation. The capacitive sensors detect physical engagement, the "dead man's switch" timer is reset to 15 seconds, and the simulated Autopilot is safely engaged.

Rapid Flashing: Indicates the Warning/Nag state. Triggered when the timer drops below 5 seconds. Acts as a visual prompt alongside the audio buzzer for the driver to return their hands to the wheel.

OFF: Indicates a system lockout or critical failure.

Hardware & Electrical Characteristics:

Component: Standard 5mm Blue Light Emitting Diode.

Forward Voltage: ~3.0V – 3.3V (Requires a wider bandgap semiconductor material, typically Indium Gallium Nitride / InGaN).

Wavelength: ~450 nm – 470 nm.

Circuit Integration: Driven by microcontroller Digital Pin 8. Requires a smaller current-limiting resistor (e.g., 100Ω - 220Ω) compared to the red LED due to its higher turn-on voltage requirements against a standard 5V logic level.



<img width="1320" height="762" alt="redled" src="https://github.com/user-attachments/assets/a0ea339d-252e-4eb5-bcda-a250053bc2a1" />


Red LED: Critical Warning & System Lockout
System Logic & State Machine Behavior:

Solid ON: Indicates a Critical Lockout state. Triggered when the 15-second timer reaches zero without capacitive feedback. The simulated Autopilot is forcefully disengaged, and the system is halted until a hard physical reset occurs.

OFF: Indicates nominal operation or the standard warning phase. The red LED remains entirely dark unless a critical inattention threshold is breached.

Hardware & Electrical Characteristics:

Component: Standard 5mm Red Light Emitting Diode.

Forward Voltage: ~1.8V – 2.0V (Utilizes a lower bandgap material, typically Gallium Arsenide Phosphide / GaAsP or Aluminum Gallium Arsenide / AlGaAs).

Wavelength: ~620 nm – 630 nm.

Circuit Integration: Driven by microcontroller Digital Pin 9. Requires a larger current-limiting resistor (e.g., 220Ω - 330Ω) to safely drop the remaining voltage from the 5V logic pin and prevent diode burnout.


