#include <Arduino.h>
#include <Adafruit_Arcada.h>
#include <Adafruit_GFX.h>
#include <math.h>

Adafruit_Arcada arcada;

// ----- Sine LUT (kept from previous code, though not used here) -----
#define LUT_SIZE 1024
float sineLUT[LUT_SIZE];
void buildSineLUT() {
  for (int i = 0; i < LUT_SIZE; i++) {
    sineLUT[i] = sin(2 * PI * i / LUT_SIZE);
  }
}
float fastSin(float angle) {
  while (angle < 0) angle += 2 * PI;
  while (angle >= 2 * PI) angle -= 2 * PI;
  int index = (int)(angle / (2 * PI) * LUT_SIZE) % LUT_SIZE;
  return sineLUT[index];
}

// ----- Color Definitions -----
#define COLOR_WHITE 0xFFFF
#define COLOR_BLACK 0x0000
#define COLOR_MIDGRAY 0xFFFF  // color for radial lines

// ----- Parameters for the Polar Dot Grid (Bottom Layer) -----
int polarRingCount = 90;           // Number of concentric rings
float polarRingSpacing = 7.0;      // Distance between rings (in pixels)
int polarRadialCount = 70;         // Number of dots per ring
int polarDotDiameter = 5;          // Dot diameter (in pixels)
int polarDr = 1;                   // Subtract this from each ring's radius for inner placement

// ----- Bottom Grid Offset -----
// New variables: adjust these to offset the bottom grid.
int bottomGridOffsetX = 5;         // Change to shift bottom grid horizontally
int bottomGridOffsetY = 5;         // Change to shift bottom grid vertically

// ----- Top Grid Control Parameters (for the polar grid) -----
int topGridOffsetX = 0;          // Additional x offset for top grid (in pixels)
int topGridOffsetY = 0;           // Additional y offset for top grid (in pixels)
float topGridRotation = 0.0;       // Initial rotation angle (degrees) for top grid
float topGridRotationSpeed = 2.0;  // Degrees to rotate per loop iteration

// ----- Bottom Grid Rotation Parameter -----
// The bottom grid will rotate in the opposite direction.
float bottomGridRotation = 0.0;    // Initial rotation angle (degrees) for bottom grid

// ----- Parameters for Radial Lines Background -----
const int numRadialLines = 12;         // Number of radial lines
const int radialLineCenterThickness = 1; // Thickness at center (pixels)
const int radialLineEdgeThickness = 50;  // Thickness at edge (pixels)
const int radialLineStep = 15;           // Step size (in pixels) for drawing segments

// ----- Helper: Draw a filled circle.
void drawFilledCircle(GFXcanvas16 *canvas, int cx, int cy, int diameter, uint16_t color) {
  int radius = diameter / 2;
  canvas->fillCircle(cx, cy, radius, color);
}

// ----- Custom fillEllipse function -----
// Draws a filled ellipse centered at (cx,cy) with horizontal radius rx and vertical radius ry.
void fillEllipse(GFXcanvas16 *canvas, int cx, int cy, int rx, int ry, uint16_t color) {
  for (int y = -ry; y <= ry; y++) {
    float dy = (float)y;
    float ratio = (ry == 0) ? 0 : (dy * dy) / ((float)ry * ry);
    if (ratio > 1.0) ratio = 1.0;
    int xSpan = (int)(rx * sqrt(1.0 - ratio));
    canvas->drawFastHLine(cx - xSpan, cy + y, 2 * xSpan + 1, color);
  }
}

// ----- Draw a thick line by drawing several parallel offset lines.
void drawThickLine(GFXcanvas16 *canvas, int x0, int y0, int x1, int y1, uint16_t color, int thickness) {
  float dx = x1 - x0;
  float dy = y1 - y0;
  float dist = sqrt(dx * dx + dy * dy);
  if (dist == 0) dist = 1;
  int half = thickness / 2;
  for (int i = -half; i <= half; i++){
    int offsetX = round(i * (-dy) / dist);
    int offsetY = round(i * (dx) / dist);
    canvas->drawLine(x0 + offsetX, y0 + offsetY, x1 + offsetX, y1 + offsetY, color);
  }
}

// ----- Draw a gradient radial line in segments.
// Interpolates thickness from centerThickness at r=0 to edgeThickness at r=maxR.
void drawGradientRadialLine(GFXcanvas16 *canvas, int cx, int cy, float angleRad, int maxR, int centerThickness, int edgeThickness, int step) {
  for (int r = 0; r < maxR; r += step) {
    float t = (float)r / maxR;
    int currentThickness = centerThickness + (int)((edgeThickness - centerThickness) * t);
    int x0 = cx + (int)(r * cos(angleRad));
    int y0 = cy + (int)(r * sin(angleRad));
    int x1 = cx + (int)((r + step) * cos(angleRad));
    int y1 = cy + (int)((r + step) * sin(angleRad));
    drawThickLine(canvas, x0, y0, x1, y1, COLOR_MIDGRAY, currentThickness);
  }
}

// ----- Draw the background radial lines.
void drawRadialLinesBackground() {
  GFXcanvas16 *canvas = arcada.getCanvas();
  int w = canvas->width();
  int h = canvas->height();
  int cx = w / 2;
  int cy = h / 2;
  int maxR = sqrt(w * w + h * h);
  float angleStep = 360.0 / numRadialLines;
  for (int i = 0; i < numRadialLines; i++) {
    float angleDeg = i * angleStep;
    float angleRad = angleDeg * PI / 180.0;
    drawGradientRadialLine(canvas, cx, cy, angleRad, maxR, radialLineCenterThickness, radialLineEdgeThickness, radialLineStep);
  }
  canvas->fillCircle(cx, cy, 5, COLOR_WHITE);
}

// ----- Draw the bottom polar dot grid (with offset and rotation).
// This grid arranges dots along concentric rings using polar coordinates.
// Now, we apply a bottom grid offset and rotation.
void drawBottomPolarGrid(GFXcanvas16 *canvas) {
  int w = canvas->width();
  int h = canvas->height();
  int cx = w / 2 + bottomGridOffsetX;
  int cy = h / 2 + bottomGridOffsetY;
  
  // Draw the center dot.
  canvas->fillCircle(cx, cy, polarDotDiameter / 2, COLOR_BLACK);
  
  // For rings 1 to polarRingCount.
  for (int ring = 1; ring <= polarRingCount; ring++) {
    float r = ring * polarRingSpacing - polarDr;
    float angleStep = 360.0 / polarRadialCount;
    for (int i = 0; i < polarRadialCount; i++) {
      // Apply bottom grid rotation.
      float angleDeg = i * angleStep + bottomGridRotation;
      float angleRad = angleDeg * PI / 180.0;
      int x = cx + (int)(r * cos(angleRad));
      int y = cy + (int)(r * sin(angleRad));
      canvas->fillCircle(x, y, polarDotDiameter / 2, COLOR_BLACK);
    }
  }
}

// ----- Draw the top polar dot grid (duplicate of bottom) -----
// This grid uses the same polar arrangement, then applies an offset and rotation.
void drawTopPolarGrid(GFXcanvas16 *canvas) {
  int w = canvas->width();
  int h = canvas->height();
  int cx = w / 2;
  int cy = h / 2;
  
  // Draw the center dot with top grid offset.
  int xCenter = cx + topGridOffsetX;
  int yCenter = cy + topGridOffsetY;
  canvas->fillCircle(xCenter, yCenter, polarDotDiameter / 2, COLOR_BLACK);
  
  // For rings 1 to polarRingCount.
  for (int ring = 1; ring <= polarRingCount; ring++) {
    float r = ring * polarRingSpacing - polarDr;
    float angleStep = 360.0 / polarRadialCount;
    for (int i = 0; i < polarRadialCount; i++) {
      float angleDeg = i * angleStep;
      // Apply top grid rotation.
      angleDeg += topGridRotation;
      float angleRad = angleDeg * PI / 180.0;
      int x = cx + (int)(r * cos(angleRad));
      int y = cy + (int)(r * sin(angleRad));
      // Apply top grid offset.
      x += topGridOffsetX;
      y += topGridOffsetY;
      canvas->fillCircle(x, y, polarDotDiameter / 2, COLOR_BLACK);
    }
  }
}

void setup() {
  Serial.begin(115200);
  buildSineLUT();
  if (!arcada.arcadaBegin()){
    while (1);
  }
  arcada.displayBegin();
  arcada.setBacklight(255);
  if (!arcada.createFrameBuffer(ARCADA_TFT_WIDTH, ARCADA_TFT_HEIGHT)){
    while (1);
  }
}

void loop() {
  GFXcanvas16 *canvas = arcada.getCanvas();
  canvas->fillScreen(COLOR_WHITE);
  
  // Draw the bottom polar grid with its own offset and rotation.
  drawBottomPolarGrid(canvas);
  
  // Draw the background radial lines.
  drawRadialLinesBackground();
  
  // Draw the top polar grid (offset & rotated).
  drawTopPolarGrid(canvas);
  
  // Update rotations:
  // Bottom grid rotates counterclockwise.
  bottomGridRotation -= topGridRotationSpeed;
  if (bottomGridRotation < 0) bottomGridRotation += 360.0;
  // Top grid rotates clockwise.
  topGridRotation += topGridRotationSpeed;
  if (topGridRotation >= 360.0) topGridRotation -= 360.0;
  
  arcada.blitFrameBuffer(0, 0, false, true);
  delay(30);
}
