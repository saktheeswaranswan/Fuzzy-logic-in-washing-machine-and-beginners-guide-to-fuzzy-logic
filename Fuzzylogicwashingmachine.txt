
/*
  CYBER FUZZY WASHING MACHINE 3D
  MULTI-SCREEN WHIRLPOOL + FUZZY INFERENCE LAB
  Processing Java Mode / APDE

  SCREENS:
    0 = LIVE 3D WASH
    1 = STAIN INSPECTION + TIMESTAMP HISTORY
    2 = FUZZY INFERENCE + RULE FIRING
    3 = DEFUZZIFICATION + OUTPUT ANALYSIS

  CONTROLS:
    NEXT / PREV : switch screens and record observations
    START       : start washing
    PAUSE       : pause / resume
    RESET       : reset wash cycle
    1,2,3       : pant, shirt, chudidhar
    4,5,6,7     : oil, grease, masala, curry
    F,B,L,V     : front, back, left, right
    T           : top-down camera
    M           : angled camera
    Arrow keys  : rotate camera
    Mouse drag  : rotate 3D view
    +/-         : zoom
    SPACE       : pause / resume
    S           : start
    R           : reset

  The fuzzy controller uses Mamdani-style max-min
  inference and numerical centroid defuzzification.
  All cleaning and fluid effects are educational
  simulations, not experimentally validated physics.
*/

final int LIVE_SCREEN = 0;
final int STAIN_SCREEN = 1;
final int INFERENCE_SCREEN = 2;
final int DEFUZZ_SCREEN = 3;
final int SCREEN_COUNT = 4;

final int HISTORY_SIZE = 32;
final int PATCH_COUNT = 12;
final int BUBBLE_COUNT = 100;
final int STREAM_COUNT = 18;
final int STREAM_POINTS = 65;

int screenID = LIVE_SCREEN;
int garmentType = 0;
int stainType = 0;
int viewSide = 0;

boolean washing = false;
boolean paused = false;
boolean topView = false;

float cycleTime = 0;
float maxCycleTime = 120;
float remainingDirt = 100;
float stainRemoval = 0;
float initialDirt = 100;

float waterSupply = 75;
float detergentLevel = 70;
float fabricSensitivity = 35;
float dirtIntensity = 85;
float stainDifficulty = 68;

float washPower = 0;
float waterUsed = 0;
float foamAmount = 0;
float drumAngle = 0;
float vortexAngle = 0;
float washElapsed = 0;

float cameraRX = -0.20;
float cameraRY = -0.40;
float cameraZoom = 1.0;

float lastMouseX;
float lastMouseY;
boolean dragging = false;

float[] patchX = new float[PATCH_COUNT];
float[] patchY = new float[PATCH_COUNT];
float[] patchSize = new float[PATCH_COUNT];

float[] bubbleX = new float[BUBBLE_COUNT];
float[] bubbleY = new float[BUBBLE_COUNT];
float[] bubbleZ = new float[BUBBLE_COUNT];
float[] bubbleR = new float[BUBBLE_COUNT];

float[] historyTime = new float[HISTORY_SIZE];
float[] historyDirt = new float[HISTORY_SIZE];
float[] historyPower = new float[HISTORY_SIZE];
float[] historyWater = new float[HISTORY_SIZE];
int[] historyScreen = new int[HISTORY_SIZE];
int historyCount = 0;
int historyIndex = 0;

float dirtLow, dirtMedium, dirtHigh;
float difficultyLow, difficultyMedium, difficultyHigh;
float fabricDelicate, fabricRobust;
float waterLow, waterGood;
float detergentLow, detergentGood;

float ruleLow, ruleMedium, ruleHigh;
float ruleDelicate, ruleResources, ruleHeavy;
float fuzzyOutput = 25;

float centroidNumerator = 0;
float centroidDenominator = 0;

float[] outputAggregate = new float[101];
float[] outputLowMF = new float[101];
float[] outputMediumMF = new float[101];
float[] outputHighMF = new float[101];

PFont uiFont;

// =====================================================
// SETUP
// =====================================================

void setup() {
  fullScreen(P3D);
  orientation(LANDSCAPE);
  smooth(4);
  sphereDetail(8);

  uiFont = createFont("SansSerif", 16, true);
  textFont(uiFont);

  initializePatches();
  initializeBubbles();
  calculateFuzzyInference();
  calculateDefuzzification();
}

void initializePatches() {
  for (int i = 0; i < PATCH_COUNT; i++) {
    patchX[i] = random(-0.8, 0.8);
    patchY[i] = random(-0.8, 0.8);
    patchSize[i] = random(0.055, 0.13);
  }
}

void initializeBubbles() {
  for (int i = 0; i < BUBBLE_COUNT; i++) {
    bubbleX[i] = random(-230, 230);
    bubbleY[i] = random(-230, 230);
    bubbleZ[i] = random(-130, 130);
    bubbleR[i] = random(2.5, 7.0);
  }
}

// =====================================================
// MAIN DRAW
// =====================================================

void draw() {
  background(3, 7, 19);

  if (washing && !paused) {
    updateWash();
  }

  calculateFuzzyInference();
  calculateDefuzzification();

  if (screenID == LIVE_SCREEN) {
    drawLiveScreen();
  } else if (screenID == STAIN_SCREEN) {
    drawStainScreen();
  } else if (screenID == INFERENCE_SCREEN) {
    drawInferenceScreen();
  } else {
    drawDefuzzScreen();
  }

  drawNavigation();
}

// =====================================================
// WASH MODEL
// =====================================================

void updateWash() {
  float dt = min(2.0, max(0.1, 60.0 / max(1, frameRate)));

  cycleTime = min(maxCycleTime, cycleTime + dt);
  washElapsed += dt;

  drumAngle += dt * (0.02 + washPower * 0.00045);
  vortexAngle += dt * (0.025 + waterSupply * 0.0007);

  foamAmount = constrain(
    foamAmount + detergentLevel * 0.0005 * dt - 0.01 * dt,
    0, 100
  );

  waterUsed += waterSupply * 0.00012 * dt;

  float detergentFactor = map(
    detergentLevel, 0, 100, 0.10, 1.0
  );

  float waterFactor = map(
    waterSupply, 0, 100, 0.05, 1.0
  );

  float fabricFactor = map(
    fabricSensitivity, 0, 100, 1.0, 0.50
  );

  float stainFactor = map(
    stainDifficulty, 0, 100, 1.0, 0.40
  );

  // Illustrative simulated stain-removal rate.
  float removalRate =
    (0.055 + washPower * 0.0011) *
    detergentFactor *
    waterFactor *
    fabricFactor *
    stainFactor;

  remainingDirt = max(
    0, remainingDirt - removalRate * dt
  );

  stainRemoval = 100 - remainingDirt;

  if (cycleTime >= maxCycleTime || remainingDirt <= 0.1) {
    remainingDirt = max(0, remainingDirt);
    stainRemoval = 100 - remainingDirt;
    washing = false;
    paused = false;
  }

  // Recalculate controller output continuously.
  calculateFuzzyInference();
  calculateDefuzzification();

  washPower = lerp(washPower, fuzzyOutput, 0.12);
}

// =====================================================
// FUZZY MEMBERSHIP FUNCTIONS
// =====================================================

float leftShoulder(float x, float a, float b) {
  if (x <= a) return 1;
  if (x >= b) return 0;
  return (b - x) / (b - a);
}

float rightShoulder(float x, float a, float b) {
  if (x <= a) return 0;
  if (x >= b) return 1;
  return (x - a) / (b - a);
}

float triangleMF(float x, float a, float b, float c) {
  if (x <= a || x >= c) return 0;
  if (x == b) return 1;
  if (x < b) return (x - a) / (b - a);
  return (c - x) / (c - b);
}

float trapezoidMF(
  float x, float a, float b, float c, float d
) {
  if (x <= a || x >= d) return 0;
  if (x >= b && x <= c) return 1;
  if (x < b) return (x - a) / (b - a);
  return (d - x) / (d - c);
}

// =====================================================
// MAMDANI FUZZY INFERENCE
// =====================================================

void calculateFuzzyInference() {
  float dirt = constrain(
    remainingDirt * dirtIntensity / 100.0, 0, 100
  );

  dirtLow = leftShoulder(dirt, 20, 55);
  dirtMedium = triangleMF(dirt, 25, 50, 75);
  dirtHigh = rightShoulder(dirt, 45, 80);

  difficultyLow = leftShoulder(stainDifficulty, 20, 55);
  difficultyMedium = triangleMF(
    stainDifficulty, 25, 50, 75
  );
  difficultyHigh = rightShoulder(
    stainDifficulty, 45, 80
  );

  fabricDelicate = rightShoulder(
    fabricSensitivity, 35, 75
  );

  fabricRobust = leftShoulder(
    fabricSensitivity, 25, 65
  );

  waterLow = leftShoulder(waterSupply, 20, 55);
  waterGood = rightShoulder(waterSupply, 40, 75);

  detergentLow = leftShoulder(
    detergentLevel, 20, 55
  );

  detergentGood = rightShoulder(
    detergentLevel, 40, 75
  );

  // RULE 1:
  // Delicate fabric => low power.
  ruleDelicate = fabricDelicate;

  // RULE 2:
  // Low water OR low detergent => reduce power.
  ruleResources = max(waterLow, detergentLow);

  // RULE 3:
  // Heavy dirt AND difficult stain AND adequate resources
  // AND robust fabric => high power.
  ruleHeavy = min(
    min(dirtHigh, difficultyHigh),
    min(
      min(waterGood, detergentGood),
      fabricRobust
    )
  );

  // RULE 4:
  // Medium dirt OR medium difficulty => medium power.
  float mediumDirtRule = max(
    dirtMedium, difficultyMedium
  );

  // RULE 5:
  // Low dirt => low power.
  float lowDirtRule = dirtLow;

  // Aggregate all rule firing strengths by output class.
  ruleLow = max(
    max(ruleDelicate, ruleResources * 0.85),
    lowDirtRule
  );

  ruleHigh = ruleHeavy;

  ruleMedium = max(
    mediumDirtRule,
    min(waterGood, detergentGood) * 0.50
  );

  // Safety: a delicate fabric must not receive
  // a high-power firing strength.
  if (fabricSensitivity > 70) {
    ruleHigh = min(ruleHigh, 0.15);
    ruleMedium = min(ruleMedium, 0.65);
  }

  ruleLow = constrain(ruleLow, 0, 1);
  ruleMedium = constrain(ruleMedium, 0, 1);
  ruleHigh = constrain(ruleHigh, 0, 1);
}

// =====================================================
// OUTPUT MEMBERSHIP FUNCTIONS AND CENTROID DEFUZZIFICATION
// =====================================================

void calculateDefuzzification() {
  centroidNumerator = 0;
  centroidDenominator = 0;

  for (int x = 0; x <= 100; x++) {
    float lowMF = leftShoulder(x, 20, 45);
    float mediumMF = triangleMF(x, 20, 50, 80);
    float highMF = rightShoulder(x, 55, 90);

    outputLowMF[x] = lowMF;
    outputMediumMF[x] = mediumMF;
    outputHighMF[x] = highMF;

    // Mamdani implication: clip each output MF by
    // its rule firing strength.
    float lowClip = min(ruleLow, lowMF);
    float mediumClip = min(ruleMedium, mediumMF);
    float highClip = min(ruleHigh, highMF);

    // Max aggregation of the clipped output sets.
    float aggregate = max(
      lowClip, max(mediumClip, highClip)
    );

    outputAggregate[x] = aggregate;

    centroidNumerator += x * aggregate;
    centroidDenominator += aggregate;
  }

  if (centroidDenominator > 0.0001) {
    fuzzyOutput = centroidNumerator / centroidDenominator;
  } else {
    fuzzyOutput = 25;
  }

  fuzzyOutput = constrain(fuzzyOutput, 0, 100);
}

// =====================================================
// LIVE WASH SCREEN
// =====================================================

void drawLiveScreen() {
  draw3DMachine();

  drawPanel(
    16, 95, min(300, width * 0.30), 260,
    "LIVE WASH STATUS"
  );

  float px = 28;
  float py = 135;

  drawText(
    "Fabric: " + garmentName(),
    px, py, 13, color(255)
  );

  drawText(
    "Stain: " + stainName(),
    px, py+23, 13, color(255)
  );

  drawText(
    "Cycle: " + nf(100*cycleTime/maxCycleTime, 1, 1) + "%",
    px, py+46, 13, color(0, 255, 255)
  );

  drawText(
    "Dirt remaining: " + nf(remainingDirt, 1, 1) + "%",
    px, py+69, 13, color(255, 190, 90)
  );

  drawText(
    "Stain removed: " + nf(stainRemoval, 1, 1) + "%",
    px, py+92, 13, color(80, 255, 130)
  );

  drawText(
    "Wash power: " + nf(washPower, 1, 1) + "%",
    px, py+115, 13, color(255, 80, 230)
  );

  drawText(
    "Water: " + nf(waterSupply, 0, 0) + "%",
    px, py+138, 13, color(0, 220, 255)
  );

  drawText(
    "Detergent: " + nf(detergentLevel, 0, 0) + "%",
    px, py+161, 13, color(255, 100, 220)
  );

  drawBar(
    px, py+185, 240, 10,
    remainingDirt, color(255, 100, 40)
  );

  drawText(
    washing ? (paused ? "PAUSED" : "WASHING")
            : "READY / COMPLETE",
    px, py+207, 13, color(0, 255, 255)
  );

  drawText(
    "Drag scene to rotate in 3D",
    px, py+229, 12, color(200)
  );

  drawControlPanel();
}

// =====================================================
// 3D MACHINE, WHIRLPOOL AND STREAMLINES
// =====================================================

void draw3DMachine() {
  hint(ENABLE_DEPTH_TEST);
  lights();

  ambientLight(40, 50, 70);
  directionalLight(
    0, 220, 255, -0.4, 0.7, -1
  );
  directionalLight(
    255, 0, 180, 0.6, -0.4, 1
  );

  pushMatrix();

  translate(width*0.57, height*0.55, -420);
  scale(cameraZoom);

  if (topView) {
    rotateX(0);
    rotateY(0);
    rotateZ(cameraRY);
  } else {
    rotateX(cameraRX);
    rotateY(cameraRY);
  }

  // Machine cabinet.
  pushStyle();
  noFill();
  stroke(0, 220, 255, 130);
  strokeWeight(3);
  box(680, 650, 250);

  stroke(255, 0, 190, 80);
  strokeWeight(1);
  box(700, 670, 270);
  popStyle();

  // Drum rotates independently.
  pushMatrix();
  rotateZ(drumAngle);

  noFill();
  stroke(0, 255, 255, 220);
  strokeWeight(6);
  ellipse(0, 0, 500, 500);

  stroke(255, 0, 210, 160);
  strokeWeight(2);
  ellipse(0, 0, 470, 470);

  for (int i=0; i<32; i++) {
    float a = TWO_PI*i/32.0;
    line(
      0, 0, 0,
      240*cos(a), 240*sin(a), 0
    );
  }

  popMatrix();

  drawWhirlpool();
  drawStreamlines();
  drawWaterJets();
  drawGarment3D();
  drawFoam3D();
  drawBubbles3D();

  popMatrix();

  hint(DISABLE_DEPTH_TEST);
}

// =====================================================
// WHIRLPOOL VORTEX
// =====================================================

void drawWhirlpool() {
  noStroke();

  // Concentric transparent water layers.
  for (int ring=0; ring<12; ring++) {
    pushMatrix();

    rotateZ(
      vortexAngle * (1.0 + ring*0.07)
    );

    float diameter = 420 - ring*25;
    float alpha = 10 + ring*2;

    fill(0, 100 + ring*7, 255, alpha);
    ellipse(0, 0, diameter, diameter);

    popMatrix();
  }

  // Vortex funnel, built from circular rings.
  for (int j=0; j<12; j++) {
    float z = -65 + j*12;
    float radius = 18 + j*15;

    pushMatrix();
    translate(0, 0, z);
    rotateZ(vortexAngle*(1.3 + j*0.04));

    noFill();
    stroke(0, 220, 255, 80 + j*7);
    strokeWeight(2);
    ellipse(0, 0, radius*2, radius*2);

    popMatrix();
  }

  // Bright central vortex.
  pushMatrix();
  translate(0, 0, 10);
  noStroke();
  fill(0, 170, 255, 100);
  sphere(24);
  fill(0, 255, 255, 180);
  sphere(10);
  popMatrix();
}

// =====================================================
// CURVED VORTEX STREAMLINES
// =====================================================

void drawStreamlines() {
  noFill();

  for (int s=0; s<STREAM_COUNT; s++) {
    float baseAngle = TWO_PI*s/STREAM_COUNT;
    float radiusBase = 55 + s*8;

    stroke(
      s%2 == 0 ? color(0, 240, 255, 190)
               : color(255, 50, 220, 140)
    );

    strokeWeight(s%3 == 0 ? 2.5 : 1.4);
    beginShape();

    for (int p=0; p<STREAM_POINTS; p++) {
      float t = p/(float)(STREAM_POINTS-1);

      float radius = radiusBase*(1.0-t) + 12;
      float a = baseAngle + vortexAngle + t*TWO_PI*1.5;

      float x = radius*cos(a);
      float y = radius*sin(a);
      float z = -75 + 150*t + 12*sin(a*2);

      vertex(x, y, z);
    }

    endShape();
  }

  // Small moving markers travelling along the vortex.
  for (int i=0; i<STREAM_COUNT; i++) {
    float t = (frameCount*0.004 + i*0.071) % 1.0;
    float radius = 60 + 115*(1.0-t);
    float a = TWO_PI*i/STREAM_COUNT +
              vortexAngle + t*TWO_PI*1.5;

    pushMatrix();
    translate(
      radius*cos(a),
      radius*sin(a),
      -75 + 150*t
    );

    noStroke();
    fill(150, 250, 255, 230);
    sphere(4);
    popMatrix();
  }
}

// =====================================================
// WATER JETS
// =====================================================

void drawWaterJets() {
  stroke(0, 190, 255, 170);
  strokeWeight(3);

  for (int i=0; i<16; i++) {
    float a = TWO_PI*i/16.0 + vortexAngle;

    float x1 = 235*cos(a);
    float y1 = 235*sin(a);
    float x2 = 150*cos(a+0.45);
    float y2 = 150*sin(a+0.45);

    line(
      x1, y1, 30,
      x2, y2, 25*sin(frameCount*0.06+i)
    );
  }

  noStroke();
}

// =====================================================
// GARMENT MODEL AND LIVE STAIN PATCHES
// =====================================================

void drawGarment3D() {
  pushMatrix();

  rotateY(viewSide*HALF_PI);
  rotateZ(sin(frameCount*0.015)*0.04);
  rotateX(sin(frameCount*0.011)*0.03);

  noStroke();

  if (garmentType == 0) {
    // Pants.
    fill(220, 225, 240);
    box(150, 55, 20);

    pushMatrix();
    translate(-38, 100, 0);
    box(65, 180, 20);
    popMatrix();

    pushMatrix();
    translate(38, 100, 0);
    box(65, 180, 20);
    popMatrix();

  } else if (garmentType == 1) {
    // Shirt.
    fill(220, 230, 245);
    box(165, 190, 22);

    pushMatrix();
    translate(-105, -35, 0);
    rotateZ(-0.15);
    box(55, 120, 18);
    popMatrix();

    pushMatrix();
    translate(105, -35, 0);
    rotateZ(0.15);
    box(55, 120, 18);
    popMatrix();

    fill(140, 170, 205);
    triangle(-25, -95, 0, -60, 0, -95);
    triangle(25, -95, 0, -60, 0, -95);

  } else {
    // Chudidhar tunic and leggings.
    fill(225, 220, 240);

    beginShape(QUADS);
    vertex(-65, -110, 12);
    vertex(65, -110, 12);
    vertex(100, 100, 12);
    vertex(-100, 100, 12);
    endShape();

    fill(195, 195, 225);

    pushMatrix();
    translate(-32, 175, 0);
    box(48, 170, 18);
    popMatrix();

    pushMatrix();
    translate(32, 175, 0);
    box(48, 170, 18);
    popMatrix();
  }

  drawGarmentStains();

  // Seams.
  stroke(110, 140, 175, 180);
  strokeWeight(2);
  line(-55, -70, 14, 55, -70, 14);
  line(0, -50, 14, 0, 90, 14);
  noStroke();

  popMatrix();
}

void drawGarmentStains() {
  // Use the same patch positions for all timestamps.
  // Their displayed size and alpha depend on remaining dirt.
  float dirtFraction = constrain(remainingDirt/100.0, 0, 1);

  for (int i=0; i<PATCH_COUNT; i++) {
    float patchScale = patchSize[i] *
                       500 *
                       (0.18 + dirtFraction*0.82);

    float alpha = 235*dirtFraction;

    if (alpha < 1) continue;

    pushMatrix();

    translate(
      patchX[i]*100,
      patchY[i]*100,
      14
    );

    rotateZ(i*0.73 + frameCount*0.0005);

    noStroke();

    if (stainType == 0) {
      fill(90, 65, 25, alpha);
    } else if (stainType == 1) {
      fill(55, 45, 35, alpha);
    } else if (stainType == 2) {
      fill(195, 65, 20, alpha);
    } else {
      fill(165, 65, 20, alpha);
    }

    beginShape();

    for (int k=0; k<12; k++) {
      float a = TWO_PI*k/12.0;
      float irregular = 0.75 + 0.25*sin(k*2.1+i);

      vertex(
        patchScale*cos(a)*irregular,
        patchScale*sin(a)*0.65*irregular,
        0
      );
    }

    endShape(CLOSE);

    fill(35, 25, 15, alpha*0.65);
    ellipse(
      -patchScale*0.12,
      patchScale*0.08,
      patchScale*0.35,
      patchScale*0.23
    );

    popMatrix();
  }
}

// =====================================================
// FOAM AND BUBBLES
// =====================================================

void drawFoam3D() {
  if (detergentLevel < 1) return;

  for (int i=0; i<25; i++) {
    float a = TWO_PI*i/25.0 + vortexAngle;
    float r = 165 + 16*sin(frameCount*0.035+i);

    pushMatrix();
    translate(
      r*cos(a),
      r*sin(a),
      30*sin(i+frameCount*0.02)
    );

    noStroke();
    fill(220, 250, 255, 40+foamAmount);
    sphere(5+detergentLevel*0.025);
    popMatrix();
  }
}

void drawBubbles3D() {
  for (int i=0; i<BUBBLE_COUNT; i++) {
    if (washing && !paused) {
      bubbleY[i] -= 0.7 + waterSupply*0.015;
      bubbleX[i] += sin(frameCount*0.025+i)*0.45;

      if (bubbleY[i] < -240) {
        bubbleY[i] = random(160, 260);
        bubbleX[i] = random(-200, 200);
      }
    }

    pushMatrix();
    translate(bubbleX[i], bubbleY[i], bubbleZ[i]);

    noFill();
    stroke(110, 225, 255, 125);
    strokeWeight(1.3);
    sphere(bubbleR[i]);

    popMatrix();
  }

  noStroke();
}

// =====================================================
// SCREEN 1: STAIN INSPECTION AND HISTORY
// =====================================================

void drawStainScreen() {
  draw3DMachine();

  drawPanel(
    16, 95, min(330, width*0.32), 300,
    "STAIN INSPECTION"
  );

  float px = 28;
  float py = 138;

  drawText(
    "Current timestamp: " + timestampString(),
    px, py, 12, color(0,255,255)
  );

  drawText(
    "Garment: " + garmentName(),
    px, py+26, 13, color(255)
  );

  drawText(
    "Stain: " + stainName(),
    px, py+49, 13, color(255)
  );

  drawText(
    "Remaining: " + nf(remainingDirt,1,2) + "%",
    px, py+72, 13, color(255,160,80)
  );

  drawText(
    "Removed: " + nf(stainRemoval,1,2) + "%",
    px, py+95, 13, color(80,255,130)
  );

  drawText(
    "Patch scale: " + nf(remainingDirt,1,1) + "%",
    px, py+118, 13, color(255,220,100)
  );

  drawText(
    "Samples recorded: " + historyCount,
    px, py+141, 13, color(0,255,255)
  );

  drawText(
    "NEXT records another observation.",
    px, py+166, 12, color(200)
  );

  // History chart.
  float gx = px;
  float gy = py+225;
  float gw = min(285, width*0.28);
  float gh = 45;

  stroke(100,130,160);
  line(gx, gy, gx+gw, gy);
  line(gx, gy-gh, gx, gy);

  if (historyCount > 1) {
    noFill();
    stroke(255,120,60);
    strokeWeight(2);
    beginShape();

    int start = (historyIndex-historyCount+HISTORY_SIZE)
                % HISTORY_SIZE;

    for (int i=0; i<historyCount; i++) {
      int idx = (start+i)%HISTORY_SIZE;
      float x = map(i, 0, historyCount-1, gx, gx+gw);
      float y = gy - gh*historyDirt[idx]/100.0;
      vertex(x,y);
    }

    endShape();
  }

  noStroke();
  drawText(
    "Dirt history: high to low",
    gx, gy+7, 11, color(200)
  );

  drawControlPanel();
}

// =====================================================
// SCREEN 2: FUZZY INFERENCE
// =====================================================

void drawInferenceScreen() {
  background(4, 8, 23);
  drawGridBackground();

  drawPanel(
    20, 95, width-40, height-155,
    "FUZZY INFERENCE — MEMBERSHIP + RULE FIRING"
  );

  float left = 42;
  float top = 150;
  float columnW = (width-110)/3.0;
  float graphH = min(180, height*0.23);

  drawMembershipGraph(
    left, top, columnW-20, graphH,
    "DIRT MEMBERSHIP",
    dirtLow, dirtMedium, dirtHigh,
    remainingDirt*dirtIntensity/100.0
  );

  drawMembershipGraph(
    left+columnW, top, columnW-20, graphH,
    "STAIN DIFFICULTY",
    difficultyLow, difficultyMedium,
    difficultyHigh, stainDifficulty
  );

  drawMembershipGraph(
    left+2*columnW, top, columnW-35, graphH,
    "RESOURCE MEMBERSHIP",
    waterLow, waterGood, detergentGood,
    waterSupply
  );

  float ry = top+graphH+35;

  drawText(
    "RULE FIRING STRENGTHS",
    left, ry, 16, color(0,255,255)
  );

  drawText(
    "R1 Delicate fabric -> LOW: " + nf(ruleDelicate,1,3),
    left, ry+30, 13, color(255)
  );

  drawText(
    "R2 Low resources -> LOW: " + nf(ruleResources,1,3),
    left, ry+54, 13, color(255)
  );

  drawText(
    "R3 Heavy dirt + difficult stain + resources + robust fabric -> HIGH: "
    + nf(ruleHeavy,1,3),
    left, ry+78, 12, color(255)
  );

  drawText(
    "Aggregated LOW: " + nf(ruleLow,1,3),
    left, ry+112, 13, color(100,255,140)
  );

  drawText(
    "Aggregated MEDIUM: " + nf(ruleMedium,1,3),
    left+columnW, ry+112, 13, color(255,220,80)
  );

  drawText(
    "Aggregated HIGH: " + nf(ruleHigh,1,3),
    left+2*columnW, ry+112, 13, color(255,100,100)
  );

  drawText(
    "Inputs are recalculated from the current wash state.",
    left, ry+145, 12, color(180)
  );

  drawControlPanel();
}

// =====================================================
// SCREEN 3: DEFUZZIFICATION
// =====================================================

void drawDefuzzScreen() {
  background(3, 7, 20);
  drawGridBackground();

  drawPanel(
    20, 95, width-40, height-155,
    "DEFUZZIFICATION — MAMDANI CENTROID METHOD"
  );

  float gx = 60;
  float gy = 250;
  float gw = width-120;
  float gh = min(230, height*0.31);

  // Axes.
  stroke(150,170,190);
  strokeWeight(1);
  line(gx, gy-gh, gx, gy);
  line(gx, gy, gx+gw, gy);

  // Output membership functions.
  drawOutputCurve(
    gx, gy, gw, gh, outputLowMF,
    color(0,255,130)
  );

  drawOutputCurve(
    gx, gy, gw, gh, outputMediumMF,
    color(255,220,0)
  );

  drawOutputCurve(
    gx, gy, gw, gh, outputHighMF,
    color(255,70,80)
  );

  drawOutputCurve(
    gx, gy, gw, gh, outputAggregate,
    color(0,230,255)
  );

  // Centroid marker.
  float centroidX = gx + gw*fuzzyOutput/100.0;

  stroke(255,255,255);
  strokeWeight(3);
  line(centroidX, gy-gh, centroidX, gy);

  noStroke();

  drawText(
    "LOW",
    gx, gy+12, 12, color(0,255,130)
  );

  drawText(
    "MEDIUM",
    gx+gw*0.45, gy+12, 12, color(255,220,0)
  );

  drawText(
    "HIGH",
    gx+gw-45, gy+12, 12, color(255,70,80)
  );

  float iy = gy+45;

  drawText(
    "Centroid numerator: " + nf(centroidNumerator,1,3),
    gx, iy, 13, color(255)
  );

  drawText(
    "Centroid denominator: " + nf(centroidDenominator,1,3),
    gx, iy+26, 13, color(255)
  );

  drawText(
    "Defuzzified wash power = numerator / denominator",
    gx, iy+52, 13, color(0,255,255)
  );

  drawText(
    "Output: " + nf(fuzzyOutput,1,2) + "%",
    gx, iy+78, 18, color(255,100,230)
  );

  drawText(
    "Applied power: " + nf(washPower,1,2) + "%",
    gx, iy+108, 14, color(255)
  );

  drawText(
    "Time: " + timestampString(),
    gx, iy+135, 12, color(190)
  );

  drawControlPanel();
}

// =====================================================
// MEMBERSHIP GRAPH
// =====================================================

void drawMembershipGraph(
  float x, float y, float w, float h,
  String titleText,
  float lowValue, float medValue, float highValue,
  float currentValue
) {
  drawText(titleText, x, y-25, 13, color(255));

  stroke(120,140,165);
  line(x,y+h,x+w,y+h);
  line(x,y,x,y+h);

  // LOW shoulder.
  noFill();
  stroke(0,255,130);
  strokeWeight(2);
  beginShape();
  for (int i=0; i<=100; i+=2) {
    vertex(
      x+w*i/100.0,
      y+h*(1-leftShoulder(i,20,55))
    );
  }
  endShape();

  // MEDIUM triangle.
  stroke(255,220,0);
  beginShape();
  for (int i=0; i<=100; i+=2) {
    vertex(
      x+w*i/100.0,
      y+h*(1-triangleMF(i,25,50,75))
    );
  }
  endShape();

  // HIGH shoulder.
  stroke(255,70,80);
  beginShape();
  for (int i=0; i<=100; i+=2) {
    vertex(
      x+w*i/100.0,
      y+h*(1-rightShoulder(i,45,80))
    );
  }
  endShape();

  float markerX = x+w*constrain(currentValue/100.0,0,1);

  stroke(0,230,255);
  strokeWeight(3);
  line(markerX,y,markerX,y+h);

  noStroke();

  drawText(
    "L " + nf(lowValue,1,2),
    x, y+h+7, 11, color(0,255,130)
  );

  drawText(
    "M " + nf(medValue,1,2),
    x+w*0.35, y+h+7, 11, color(255,220,0)
  );

  drawText(
    "H " + nf(highValue,1,2),
    x+w*0.68, y+h+7, 11, color(255,70,80)
  );
}

// =====================================================
// DRAW OUTPUT MEMBERSHIP CURVES
// =====================================================

void drawOutputCurve(
  float x, float y, float w, float h,
  float[] values, int curveColor
) {
  noFill();
  stroke(curveColor);
  strokeWeight(curveColor == color(0,230,255) ? 3 : 1.8);

  beginShape();

  for (int i=0; i<=100; i++) {
    vertex(
      x+w*i/100.0,
      y-h*values[i]
    );
  }

  endShape();
}

// =====================================================
// BACKGROUND GRID
// =====================================================

void drawGridBackground() {
  hint(DISABLE_DEPTH_TEST);
  camera();
  noLights();

  stroke(0,80,110,65);
  strokeWeight(1);

  for (int x=0; x<width; x+=35) {
    line(x,80,x,height-60);
  }

  for (int y=80; y<height-60; y+=35) {
    line(0,y,width,y);
  }

  noStroke();
}

// =====================================================
// UI PANELS
// =====================================================

void drawPanel(
  float x, float y, float w, float h,
  String titleText
) {
  hint(DISABLE_DEPTH_TEST);
  camera();
  noLights();

  fill(2,10,27,225);
  stroke(0,220,255,210);
  strokeWeight(1.5);
  rect(x,y,w,h,12);

  noStroke();
  fill(0,255,255);
  textAlign(LEFT,TOP);
  textSize(16);
  text(titleText,x+12,y+10);
}

void drawText(
  String value, float x, float y,
  float sizeValue, int textColor
) {
  fill(textColor);
  textAlign(LEFT,TOP);
  textSize(sizeValue);
  text(value,x,y);
}

void drawBar(
  float x, float y, float w, float h,
  float value, int barColor
) {
  noStroke();
  fill(35,45,65);
  rect(x,y,w,h,4);

  fill(barColor);
  rect(x,y,w*constrain(value/100.0,0,1),h,4);
}

void drawControlPanel() {
  hint(DISABLE_DEPTH_TEST);
  camera();
  noLights();

  float x = width-255;
  float y = height-130;

  fill(2,10,25,225);
  stroke(255,0,200,180);
  rect(x,y,238,62,8);

  noStroke();

  drawText(
    "WATER " + nf(waterSupply,0,0) + "%",
    x+10,y+7,12,color(0,230,255)
  );

  drawBar(
    x+95,y+9,128,8,waterSupply,color(0,200,255)
  );

  drawText(
    "DETERGENT " + nf(detergentLevel,0,0) + "%",
    x+10,y+31,12,color(255,100,220)
  );

  drawBar(
    x+95,y+33,128,8,detergentLevel,color(255,0,200)
  );

  drawText(
    "Tap either bar to change",
    x+10,y+47,10,color(200)
  );
}

// =====================================================
// NAVIGATION AND TIMESTAMPED OBSERVATIONS
// =====================================================

void drawNavigation() {
  hint(DISABLE_DEPTH_TEST);
  camera();
  noLights();

  noStroke();
  fill(2,8,20,240);
  rect(0,0,width,72);

  fill(0,255,255);
  textAlign(LEFT,CENTER);
  textSize(16);

  text(
    "CYBER FUZZY WASH LAB",
    16,20
  );

  fill(255);
  textSize(12);

  String[] names = {
    "LIVE 3D",
    "STAIN HISTORY",
    "FUZZY INFERENCE",
    "DEFUZZIFICATION"
  };

  float tabStart = 210;
  float tabW = max(85,(width-520)/4.0);

  for (int i=0; i<SCREEN_COUNT; i++) {
    float tx = tabStart+i*tabW;

    fill(
      screenID == i ? color(0,180,220,220)
                    : color(20,35,55,230)
    );

    stroke(0,220,255,180);
    rect(tx,7,tabW-5,30,5);

    noStroke();
    fill(255);
    textAlign(CENTER,CENTER);
    text(names[i],tx+(tabW-5)/2,22);
  }

  drawButton(
    width-265,8,74,30,
    "PREV",color(90,35,90)
  );

  drawButton(
    width-185,8,78,30,
    "NEXT",color(0,110,135)
  );

  drawButton(
    width-100,8,88,30,
    washing ? (paused ? "RESUME" : "PAUSE")
            : "START",
    color(105,30,100)
  );

  fill(180,220,235);
  textAlign(LEFT,CENTER);
  textSize(11);

  text(
    timestampString() +
    " | OBSERVATIONS " + historyCount +
    " | SCREEN " + (screenID+1) + "/4",
    16,53
  );

  drawButton(
    16, height-49, 75, 30,
    "RESET",color(110,35,45)
  );

  drawButton(
    100, height-49, 90, 30,
    "TOP VIEW",color(20,75,110)
  );

  drawButton(
    198, height-49, 92, 30,
    "ANGLE VIEW",color(20,75,110)
  );

  drawButton(
    298, height-49, 88, 30,
    "ROTATE -",color(30,50,85)
  );

  drawButton(
    394, height-49, 88, 30,
    "ROTATE +",color(30,50,85)
  );
}

void drawButton(
  float x,float y,float w,float h,
  String labelText,int buttonColor
) {
  fill(buttonColor);
  stroke(0,230,255,180);
  strokeWeight(1);
  rect(x,y,w,h,6);

  noStroke();
  fill(255);
  textAlign(CENTER,CENTER);
  textSize(11);
  text(labelText,x+w/2,y+h/2);
}

void recordObservation() {
  historyTime[historyIndex] = cycleTime;
  historyDirt[historyIndex] = remainingDirt;
  historyPower[historyIndex] = washPower;
  historyWater[historyIndex] = waterUsed;
  historyScreen[historyIndex] = screenID;

  historyIndex = (historyIndex+1)%HISTORY_SIZE;
  historyCount = min(HISTORY_SIZE,historyCount+1);
}

void nextScreen() {
  recordObservation();
  screenID = (screenID+1)%SCREEN_COUNT;
}

void previousScreen() {
  recordObservation();
  screenID = (screenID-1+SCREEN_COUNT)%SCREEN_COUNT;
}

String timestampString() {
  return nf(int(cycleTime/60),2) + ":" +
         nf(int(cycleTime)%60,2) +
         "  [t=" + nf(cycleTime,1,1) + "s]";
}

// =====================================================
// LABEL HELPERS
// =====================================================

String garmentName() {
  if (garmentType == 0) return "PANT";
  if (garmentType == 1) return "SHIRT";
  return "CHUDIDHAR";
}

String stainName() {
  if (stainType == 0) return "OIL";
  if (stainType == 1) return "GREASE";
  if (stainType == 2) return "MASALA";
  return "CURRY SAUCE";
}

// =====================================================
// RESET / START / PAUSE
// =====================================================

void startWash() {
  if (remainingDirt <= 0.1) {
    remainingDirt = 100;
    stainRemoval = 0;
    cycleTime = 0;
    initializePatches();
  }

  washing = true;
  paused = false;
}

void resetWash() {
  washing = false;
  paused = false;

  cycleTime = 0;
  remainingDirt = initialDirt;
  stainRemoval = 0;

  washPower = 0;
  waterUsed = 0;
  foamAmount = 0;

  drumAngle = 0;
  vortexAngle = 0;
  washElapsed = 0;

  historyCount = 0;
  historyIndex = 0;

  initializePatches();
  calculateFuzzyInference();
  calculateDefuzzification();
}

void selectGarment(int g) {
  garmentType = constrain(g,0,2);

  if (garmentType == 0) fabricSensitivity = 30;
  if (garmentType == 1) fabricSensitivity = 38;
  if (garmentType == 2) fabricSensitivity = 48;

  resetWash();
}

void selectStain(int s) {
  stainType = constrain(s,0,3);

  if (s == 0) {
    stainDifficulty = 68;
    dirtIntensity = 82;
  }

  if (s == 1) {
    stainDifficulty = 92;
    dirtIntensity = 90;
  }

  if (s == 2) {
    stainDifficulty = 62;
    dirtIntensity = 88;
  }

  if (s == 3) {
    stainDifficulty = 72;
    dirtIntensity = 86;
  }

  resetWash();
}

// =====================================================
// TOUCH INPUT
// =====================================================

void mousePressed() {
  lastMouseX = mouseX;
  lastMouseY = mouseY;
  dragging = false;

  // Navigation tabs.
  float tabStart = 210;
  float tabW = max(85,(width-520)/4.0);

  if (mouseY >= 7 && mouseY <= 40) {
    if (mouseX >= width-265 && mouseX <= width-191) {
      previousScreen();
      return;
    }

    if (mouseX >= width-185 && mouseX <= width-107) {
      nextScreen();
      return;
    }

    if (mouseX >= width-100 && mouseX <= width-12) {
      if (!washing) {
        startWash();
      } else {
        paused = !paused;
      }
      return;
    }

    for (int i=0; i<SCREEN_COUNT; i++) {
      float tx = tabStart+i*tabW;

      if (mouseX >= tx && mouseX <= tx+tabW-5) {
        recordObservation();
        screenID = i;
        return;
      }
    }
  }

  // Bottom navigation.
  if (mouseY >= height-50) {
    if (mouseX < 95) {
      resetWash();
      return;
    }

    if (mouseX < 195) {
      topView = true;
      return;
    }

    if (mouseX < 293) {
      topView = false;
      cameraRX = -0.20;
      cameraRY = -0.40;
      return;
    }

    if (mouseX < 390) {
      cameraRY -= 0.20;
      return;
    }

    if (mouseX < 490) {
      cameraRY += 0.20;
      return;
    }
  }

  // Water and detergent sliders.
  float sx = width-255;
  float sy = height-130;

  if (
    mouseX >= sx && mouseX <= sx+238 &&
    mouseY >= sy+5 && mouseY <= sy+23
  ) {
    waterSupply = constrain(
      map(mouseX,sx+95,sx+223,0,100),0,100
    );
    return;
  }

  if (
    mouseX >= sx && mouseX <= sx+238 &&
    mouseY >= sy+27 && mouseY <= sy+45
  ) {
    detergentLevel = constrain(
      map(mouseX,sx+95,sx+223,0,100),0,100
    );
    return;
  }

  // A drag in the main 3D area rotates the machine.
  if (
    screenID == LIVE_SCREEN ||
    screenID == STAIN_SCREEN
  ) {
    if (mouseY > 75 && mouseY < height-55) {
      dragging = true;
    }
  }
}

void mouseDragged() {
  if (!dragging) return;

  float dx = mouseX-lastMouseX;
  float dy = mouseY-lastMouseY;

  cameraRY += dx*0.008;

  if (!topView) {
    cameraRX += dy*0.008;
    cameraRX = constrain(cameraRX,-1.45,1.45);
  }

  lastMouseX = mouseX;
  lastMouseY = mouseY;
}

void mouseReleased() {
  dragging = false;
}

// =====================================================
// KEYBOARD INPUT
// =====================================================

void keyPressed() {
  if (key == ' ') {
    if (!washing) startWash();
    else paused = !paused;
  }

  if (key == 's' || key == 'S') startWash();
  if (key == 'r' || key == 'R') resetWash();

  if (key == 'n' || key == 'N') nextScreen();
  if (key == 'p' || key == 'P') previousScreen();

  if (key == '1') selectGarment(0);
  if (key == '2') selectGarment(1);
  if (key == '3') selectGarment(2);

  if (key == '4') selectStain(0);
  if (key == '5') selectStain(1);
  if (key == '6') selectStain(2);
  if (key == '7') selectStain(3);

  if (key == 'f' || key == 'F') viewSide = 0;
  if (key == 'b' || key == 'B') viewSide = 1;
  if (key == 'l' || key == 'L') viewSide = 2;
  if (key == 'v' || key == 'V') viewSide = 3;

  if (key == 't' || key == 'T') topView = true;
  if (key == 'm' || key == 'M') topView = false;

  if (key == '+' || key == '=') {
    cameraZoom = min(1.5,cameraZoom+0.05);
  }

  if (key == '-' || key == '_') {
    cameraZoom = max(0.55,cameraZoom-0.05);
  }

  if (keyCode == LEFT) cameraRY -= 0.10;
  if (keyCode == RIGHT) cameraRY += 0.10;

  if (!topView && keyCode == UP) {
    cameraRX = max(-1.45,cameraRX-0.10);
  }

  if (!topView && keyCode == DOWN) {
    cameraRX = min(1.45,cameraRX+0.10);
  }
}
