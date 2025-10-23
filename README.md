# Sistemas Inteligentes - Robot
Codigos para el sonar con Arduino y Processing

## Arduino .ino

```
/*
 -------------------
 -- Sonar Arduino --
 -------------------
*/

#include <Servo.h>

// sensor ultrasonico HC-SR04
const int trigPin = 0xa;
const int echoPin = 0xb;
long duration;
int distance;
// servo motor SG90
Servo myServo;
const int servoPin = 0xc;
const int angleMin = 0xf;
const int angleMax = 0xa5;
const int delayServo = 0x1e;

void setup() {
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  Serial.begin(9600);
  myServo.attach(servoPin);
}

void loop() {
  for (int i = angleMin; i <= angleMax; i++) {
    myServo.write(i);
    delay(delayServo);
    distance = calculateDistance();
    Serial.print(i);
    Serial.print(",");
    Serial.print(distance);
    Serial.print(".");
  }
  for (int i = angleMax; i > angleMin; i--) {
    myServo.write(i);
    delay(delayServo);
    distance = calculateDistance();
    Serial.print(i);
    Serial.print(",");
    Serial.print(distance);
    Serial.print(".");
  }
}

int calculateDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  duration = pulseIn(echoPin, HIGH);
  distance = duration * 0.034 / 2;
  return distance;
}
```

## Processing .pde

IMPORTANTE: linea 6, el puerto debe ser el mismo que en Arduino IDE.

```
import processing.serial.*;
import java.awt.event.KeyEvent;
import java.io.IOException;

// ---- config ----
final String SERIAL_PORT = "COM3"; //  mismo puerto que Arduino!!!
final int BAUD_RATE = 9600;

final int SCREEN_WIDTH = 1300;
final int SCREEN_HEIGHT = 750;

final int INFO_PANEL_HEIGHT = 50; // altura barra negra abajo
final int MAX_RANGE_CM = 40;     // maxima distancia del sonar (cm)
final int SWEEP_ANGLE_STEP = 30; // espacio entre lineas anguladas de la cuadricula (grados: 30, 45, etc)

// Colors
final color RADAR_COLOR = color(98, 245, 31); // verde
final color OBJECT_COLOR = color(255, 10, 10); // Rojo
final color BACKGROUND_COLOR = color(0, 4); // casi negro

// ---- global vars ----
Serial myPort; 
String rawData = "";
int currentAngle = 0;
int currentDistance = 0; // distancia actual de Arduino

// Visual Metrics (Calculated in setup for responsiveness)
float radarCenterX;
float radarCenterY;
float radarRadius; // The radius corresponding to MAX_RANGE_CM
float pixelsPerCM; // The conversion factor for drawing objects


// ---- setup ----
void setup() {
  // Use explicit numbers here to avoid the size() error in certain Processing modes
  size(1200, 700); 
  smooth();
  
  // Calculate dynamic visual metrics based on current width/height
  radarCenterX = width / 2;
  
  // Set the pivot point just above the info panel
  radarCenterY = height - INFO_PANEL_HEIGHT; 
  
  // Determine the maximum available radius (min of half-width or height above info panel)
  float maxAvailableRadius = min(width / 2, height - INFO_PANEL_HEIGHT);
  
  // MAXIMIZE RADAR SWEEP: Use 95% of available space
  radarRadius = maxAvailableRadius * 0.95; 
  
  // Calculate the scaling factor for distance-to-pixel conversion
  pixelsPerCM = radarRadius / MAX_RANGE_CM;

  // Initialize Serial Communication
  try {
    myPort = new Serial(this, SERIAL_PORT, BAUD_RATE);
    myPort.bufferUntil('.'); 
  } catch (Exception e) {
    println("Error: Could not open serial port " + SERIAL_PORT);
    println("Check if the port is correct and not busy (e.g., by Arduino Serial Monitor).");
  }
}

// ---- draw loop ----
void draw() {
  // Simulates motion blur/fade by drawing a semi-transparent background
  noStroke();
  fill(BACKGROUND_COLOR);
  // Only fade the radar area, not the info panel
  rect(0, 0, width, height - INFO_PANEL_HEIGHT); 
  
  // Set main drawing color
  fill(RADAR_COLOR); 

  // Calls drawing functions
  drawRadar();  
  drawLine();
  drawObject();
  drawText();
}

// ---- eventos serial ----
void serialEvent (Serial p) {
  // Read data packet (e.g., "90,25.")
  rawData = p.readStringUntil('.');
  if (rawData != null) {
    rawData = rawData.substring(0, rawData.length() - 1); // Remove '.'
    
    int separatorIndex = rawData.indexOf(",");
    
    if (separatorIndex != -1 && separatorIndex < rawData.length() - 1) {
        // Extract angle and distance
        String angleString = rawData.substring(0, separatorIndex);
        String distanceString = rawData.substring(separatorIndex + 1);

        // Convert strings to integers
        // Use trim() for robustness against leading/trailing whitespace
        try {
            currentAngle = int(angleString.trim()); 
            currentDistance = int(distanceString.trim());
        } catch (NumberFormatException e) {
            println("Error parsing serial data: " + rawData);
        }
    }
  }
}

// ---- funciones de visualizacion ----
/** Draws the stationary background grid (arcs and angle lines). */
void drawRadar() {
  pushMatrix();
  translate(radarCenterX, radarCenterY); // Move origin to the pivot point
  
  noFill();
  strokeWeight(2);
  stroke(RADAR_COLOR);
  
  // --- Draw Distance Arcs (Rings) ---
  int numRings = 4;
  for (int i = 1; i <= numRings; i++) {
    // Radius scales dynamically based on total range (MAX_RANGE_CM)
    float ringDistance = (MAX_RANGE_CM / numRings) * i;
    float ringRadius = ringDistance * pixelsPerCM;
    // The radar is a semi-circle (PI to TWO_PI)
    arc(0, 0, ringRadius * 2, ringRadius * 2, PI, TWO_PI);
  }

  // --- Draw Angle Lines (Spokes) ---
  for (int a = 0; a <= 180; a += SWEEP_ANGLE_STEP) {
    float rad = radians(a);
    // x = -r * cos(angle), y = -r * sin(angle) (Y is inverted for PGraphics for the top semi-circle)
    line(0, 0, -radarRadius * cos(rad), -radarRadius * sin(rad));
  }
  
  popMatrix();
}

/** Draws the detected object (a red mark) if it's within range. */
void drawObject() {
  pushMatrix();
  translate(radarCenterX, radarCenterY); // Move to the pivot point
  
  // Check if distance is valid and within the defined max range
  if (currentDistance > 0 && currentDistance <= MAX_RANGE_CM) {
    strokeWeight(9);
    stroke(OBJECT_COLOR); 
    
    // Convert distance from cm to pixels
    float pixsDistance = currentDistance * pixelsPerCM;
    float rad = radians(currentAngle);
    
    // Calculate object coordinates
    float obj_x = -pixsDistance * cos(rad);
    float obj_y = -pixsDistance * sin(rad);
    
    // Draw the object as a short red line segment
    // This draws a small segment 5% longer than the object's position to mark it clearly
    line(obj_x, obj_y, obj_x * 1.05, obj_y * 1.05); 
  }
  popMatrix();
}

/** Draws the sweeping radar line (the current sonar beam). */
void drawLine() {
  pushMatrix();
  strokeWeight(5);
  stroke(RADAR_COLOR);
  translate(radarCenterX, radarCenterY); // Move to the pivot point

  // Draw the sweeping line from center to the edge of the radar
  float rad = radians(currentAngle);
  line(0, 0, -radarRadius * cos(rad), -radarRadius * sin(rad));
  popMatrix();
}

/** Draws the informational text and labels. */
void drawText() {
  // --- Info Panel Background ---
  fill(0); // Black background for text area
  noStroke();
  rect(0, height - INFO_PANEL_HEIGHT, width, INFO_PANEL_HEIGHT); 
  
  fill(RADAR_COLOR);
  
  // --- Dynamic Info ---
  textSize(30);
  
  // Angle Display (Left side)
  text("Angle: " + currentAngle + " °", 150, height - 25);
  
  // Distance Display (Right side)
  String distanceText;
  if (currentDistance > MAX_RANGE_CM || currentDistance == 0) {
    distanceText = "Out of Range (" + MAX_RANGE_CM + " cm)";
  } else {
    distanceText = currentDistance + " cm";
  }
  text("Distance: " + distanceText, width - 300, height - 25); // Position 300px from right edge
  
  // --- Distance Labels (Rings) ---
  int numRings = 4;
  textSize(20);
  textAlign(CENTER, BOTTOM); // Center text over the line, aligned to the bottom
  
  // Calculate a vertical offset to ensure the text is clear of the arc line
  float textVOffset = radarCenterY - 15;
  
  for (int i = 1; i <= numRings; i++) {
    int ringDistance = (MAX_RANGE_CM / numRings) * i;
    float ringRadius = ringDistance * pixelsPerCM;
    
    // Position text exactly over the center of the arc on the right side
    text(ringDistance + "cm", radarCenterX + ringRadius, textVOffset);
  }
  
  // --- Angle Labels (Spokes) ---
  textSize(18);
  textAlign(CENTER, CENTER); // Center text entirely
  
  for (int a = SWEEP_ANGLE_STEP; a < 180; a += SWEEP_ANGLE_STEP) {
    pushMatrix();
    translate(radarCenterX, radarCenterY);
    float rad = radians(a);
    
    // Calculate the position for the label just outside the radar ring (1.1 factor)
    float textX = -radarRadius * cos(rad) * 1.1;
    float textY = -radarRadius * sin(rad) * 1.1;
    
    // Move to that position
    translate(textX, textY); 
    
    // Rotate text so it is generally aligned with the spoke. PI/2 (90 deg) is vertical.
    // We rotate to be perpendicular to the spoke line.
    rotate(-rad + PI/2); 
    
    text(a + "°", 0, 0);
    popMatrix();
  }
}
```
