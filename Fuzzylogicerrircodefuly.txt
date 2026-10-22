/*
 * FUZZY WASH LAB 3D — COMPLETE MAMDANI PIPELINE
 * APDE / Processing Android Java Mode
 *
 * Screen 1: 3D washer, input sliders, fuzzification, firing matrix,
 *           aggregated output graph and centroid.
 * Screen 2: canonical rules, live numerical sample matrix, formulas,
 *           clipping, MAX aggregation and centroid summation.
 *
 * No external libraries required.
 *
 * PIECEWISE MEMBERSHIP EQUATIONS
 * For a normal triangle a < b < c:
 *   mu(x)=0                    when x <= a or x >= c
 *   mu(x)=(x-a)/(b-a)          when a < x <= b
 *   mu(x)=(c-x)/(c-b)          when b < x < c
 *   mu(b)=1
 *
 * Left shoulder (a=b<c):
 *   mu(x)=1                    when x <= b
 *   mu(x)=(c-x)/(c-b)          when b < x < c
 *   mu(x)=0                    when x >= c
 *
 * Right shoulder (a<b=c):
 *   mu(x)=0                    when x <= a
 *   mu(x)=(x-a)/(b-a)          when a < x < b
 *   mu(x)=1                    when x >= b
 *
 * Dirt functions:
 *   LOW  = tri(d, 0, 0, 5)
 *   MED  = tri(d, 0, 5, 10)
 *   HIGH = tri(d, 5, 10, 10)
 *
 * Load functions:
 *   LOW  = tri(load, 0, 0, 4)
 *   MED  = tri(load, 0, 4, 8)
 *   HIGH = tri(load, 4, 8, 8)
 *
 * Output functions (minutes):
 *   SHORT  = tri(z, 0, 10, 20)
 *   NORMAL = tri(z, 10, 20, 30)
 *   LONG   = tri(z, 20, 30, 40)
 *
 * Mamdani pipeline:
 *   alpha[i][j] = min(muD[i], muL[j])
 *   activation[c] = max(all alpha values whose rule consequent is c)
 *   clipped[c](z) = min(activation[c], muC[c](z))
 *   aggregate(z) = max(clippedSHORT, clippedNORMAL, clippedLONG)
 *   centroid = sum(z * aggregate(z)) / sum(aggregate(z))
 *
 * Canonical rule base:
 *             LOAD LOW   LOAD MED   LOAD HIGH
 * DIRT LOW       SHORT      SHORT       NORMAL
 * DIRT MED       NORMAL     NORMAL      LONG
 * DIRT HIGH      NORMAL     LONG        LONG
 */

float dirt = 6.0;
float loadKg = 4.0;

boolean running = true;
int screenMode = 1;

float spinAngle = 0;
float animPulse = 0;
int animStep = 0;
int numericalSample = 0;

String[] names = {"LOW", "MED", "HIGH"};
String[] dirtLabels = {"DIRT LOW", "DIRT MED", "DIRT HIGH"};
String[] loadLabels = {"LOAD LOW", "LOAD MED", "LOAD HIGH"};
String[] outNames = {"SHORT", "NORMAL", "LONG"};
String[] consequentLabels = {"SHORT", "NORMAL", "LONG"};

// Consequent indices: 0 = SHORT, 1 = NORMAL, 2 = LONG.
int[][] rules = {
  {0, 0, 1},
  {1, 1, 2},
  {1, 2, 2}
};

// Membership grades and rule firing strengths.
float[] dirtMu = new float[3];
float[] loadMu = new float[3];
float[][] firing = new float[3][3];
float[] outputActivation = new float[3];

// Output universe: 0 to 40 minutes, step 0.25 = 161 samples.
final int NUM_SAMPLES = 161;
final float DZ = 0.25;

float[] sampleZ = new float[NUM_SAMPLES];
float[] sampleShort = new float[NUM_SAMPLES];
float[] sampleNormal = new float[NUM_SAMPLES];
float[] sampleLong = new float[NUM_SAMPLES];
float[] sampleAggregate = new float[NUM_SAMPLES];
float[] sampleMoment = new float[NUM_SAMPLES];

float mamdaniTime = 0;
float weightedTime = 0;
float totalFiring = 0;
float weightedNumerator = 0;
float areaSum = 0;
float momentSum = 0;
float areaApprox = 0;
float momentApprox = 0;

float s, leftX, leftW, rightX, rightW, centerX;
float headerH, footerY;

// Palette.
final int BG = 0xFF09111E;
final int PANEL = 0xFF142236;
final int PANEL2 = 0xFF1D3047;
final int WHITE = 0xFFF2F7FC;
final int MUTED = 0xFFA8BCD0;
final int CYAN = 0xFF48DCE8;
final int GREEN = 0xFF66E6A5;
final int ORANGE = 0xFFFFBD69;
final int RED = 0xFFFF777F;
final int PURPLE = 0xFFB4A0FF;
final int BLUE = 0xFF6DA9FF;

void setup() {
  orientation(LANDSCAPE);
  fullScreen(P3D);
  smooth(4);

  s = constrain(min(width / 1280.0, height / 720.0), 0.55, 1.25);
  headerH = height * 0.09;
  footerY = height - 18 * s;

  leftX = 10 * s;
  leftW = width * 0.235;
  rightW = width * 0.285;
  rightX = width - rightW - 10 * s;
  centerX = width * 0.50;

  calculateFuzzySystem();
  calculateNumericalTrace();
}

void draw() {
  background(BG);

  if (running) {
    spinAngle += 0.035 + loadKg * 0.006;
    if (spinAngle > TWO_PI) spinAngle -= TWO_PI;
    animPulse += 0.08;
    animStep++;
  }

  calculateFuzzySystem();
  calculateNumericalTrace();

  if (screenMode == 2) {
    camera();
    ortho();
    hint(DISABLE_DEPTH_TEST);
    drawCalculationScreen();
    hint(ENABLE_DEPTH_TEST);
    return;
  }

  drawMachine3D();

  camera();
  ortho();
  hint(DISABLE_DEPTH_TEST);
  drawHeader();
  drawLeft();
  drawCenter();
  drawRight();
  drawFooter();
  hint(ENABLE_DEPTH_TEST);
}

// ============================================================
// FUZZY ENGINE
// ============================================================

float tri(float x, float a, float b, float c) {
  if (a == b && b == c) return x == a ? 1.0 : 0.0;

  // Left-shoulder triangle.
  if (a == b) {
    if (x <= b) return 1.0;
    if (x >= c) return 0.0;
    return (c - x) / (c - b);
  }

  // Right-shoulder triangle.
  if (b == c) {
    if (x >= b) return 1.0;
    if (x <= a) return 0.0;
    return (x - a) / (b - a);
  }

  // Normal triangular membership function.
  if (x <= a || x >= c) return 0.0;
  if (x == b) return 1.0;

  return x < b
    ? (x - a) / (b - a)
    : (c - x) / (c - b);
}

void calculateFuzzySystem() {
  // 1. FUZZIFICATION.
  dirtMu[0] = tri(dirt, 0, 0, 5);
  dirtMu[1] = tri(dirt, 0, 5, 10);
  dirtMu[2] = tri(dirt, 5, 10, 10);

  loadMu[0] = tri(loadKg, 0, 0, 4);
  loadMu[1] = tri(loadKg, 0, 4, 8);
  loadMu[2] = tri(loadKg, 4, 8, 8);

  // 2. RULE FIRING: alpha_ij = MIN(muD_i, muL_j).
  totalFiring = 0;
  weightedNumerator = 0;

  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      firing[i][j] = min(dirtMu[i], loadMu[j]);
      int out = rules[i][j];

      totalFiring += firing[i][j];

      // Sugeno-style weighted average for comparison only.
      // This is NOT the Mamdani centroid result.
      float singleton = 10.0 + 10.0 * out;
      weightedNumerator += firing[i][j] * singleton;
    }
  }

  weightedTime = totalFiring > 0
    ? weightedNumerator / totalFiring : 0;

  // 3. CONSEQUENT ACTIVATION: MAX of rules sharing each output.
  for (int i = 0; i < 3; i++) {
    outputActivation[i] = 0;
  }

  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      int out = rules[i][j];
      outputActivation[out] =
        max(outputActivation[out], firing[i][j]);
    }
  }

  // 4. MAMDANI IMPLICATION: MIN clipping.
  // 5. AGGREGATION: pointwise MAX.
  areaSum = 0;
  momentSum = 0;
  areaApprox = 0;
  momentApprox = 0;

  for (int k = 0; k < NUM_SAMPLES; k++) {
    float z = k * DZ;

    sampleZ[k] = z;

    sampleShort[k] =
      min(outputActivation[0], tri(z, 0, 10, 20));

    sampleNormal[k] =
      min(outputActivation[1], tri(z, 10, 20, 30));

    sampleLong[k] =
      min(outputActivation[2], tri(z, 20, 30, 40));

    sampleAggregate[k] = max(
      sampleShort[k],
      max(sampleNormal[k], sampleLong[k])
    );

    sampleMoment[k] = z * sampleAggregate[k];

    areaSum += sampleAggregate[k];
    momentSum += sampleMoment[k];
  }

  // The common delta-z factor cancels in the centroid ratio.
  mamdaniTime = areaSum > 0
    ? momentSum / areaSum : 0;

  // Riemann-sum approximations for area and first moment.
  areaApprox = areaSum * DZ;
  momentApprox = momentSum * DZ;
}

void calculateNumericalTrace() {
  numericalSample = constrain(
    numericalSample, 0, NUM_SAMPLES - 1
  );
}

// ============================================================
// 3D WASHING MACHINE
// ============================================================

void drawMachine3D() {
  float cabW = min(width * 0.205, height * 0.34);
  float cabH = height * 0.48;
  float dep = height * 0.15;
  float cy = height * 0.49;

  ambientLight(100, 110, 130);
  directionalLight(210, 220, 240, -0.3, 0.4, -1);
  pointLight(60, 180, 220, centerX, cy - cabH, dep);

  pushMatrix();
  translate(centerX, cy, 0);
  rotateY(-0.10);
  noStroke();

  fill(0xFF8FA5B9);
  box(cabW, cabH, dep);

  // Top control panel.
  pushMatrix();
  translate(0, -cabH * 0.39, dep * 0.51);

  fill(0xFFDCE6EF);
  box(cabW * 0.9, cabH * 0.14, 4);

  fill(0xFF142538);
  pushMatrix();
  translate(-cabW * 0.25, 0, 4);
  box(cabW * 0.23, cabH * 0.055, 3);
  popMatrix();

  pushMatrix();
  translate(cabW * 0.24, 0, 5);
  fill(0xFF52677B);
  sphereDetail(12);
  sphere(cabW * 0.035);
  popMatrix();

  popMatrix();

  // Circular front door.
  float door = cabW * 0.61;
  float doorZ = dep * 0.51 + 4;

  pushMatrix();
  translate(0, cabH * 0.08, doorZ);

  fill(0xFF263A50);
  ellipse(0, 0, door * 1.13, door * 1.13);

  fill(0xFFE1EAF1);
  ellipse(0, 0, door * 1.04, door * 1.04);

  fill(0xFF40566D);
  ellipse(0, 0, door * 0.91, door * 0.91);

  fill(0xFF071A2B);
  ellipse(0, 0, door * 0.82, door * 0.82);

  // Animated clothes inside the door.
  pushMatrix();
  rotateZ(spinAngle);

  fill(CYAN);
  ellipse(-door * 0.17, -door * 0.11,
          door * 0.24, door * 0.38);

  fill(ORANGE);
  ellipse(door * 0.16, -door * 0.12,
          door * 0.25, door * 0.32);

  fill(BLUE);
  ellipse(door * 0.06, door * 0.18,
          door * 0.34, door * 0.19);

  fill(GREEN);
  ellipse(-door * 0.19, door * 0.17,
          door * 0.19, door * 0.25);

  noFill();
  stroke(0xFF9BB6CC);
  strokeWeight(1.2 * s);

  for (int i = 0; i < 14; i++) {
    float a = TWO_PI * i / 14.0;
    point(
      cos(a) * door * 0.32,
      sin(a) * door * 0.32
    );
  }

  popMatrix();

  noStroke();
  fill(70, 190, 225, 45);
  ellipse(-door * 0.15, -door * 0.18,
          door * 0.34, door * 0.14);

  popMatrix();
  popMatrix();
}

// ============================================================
// COMMON UI HELPERS
// ============================================================

void drawHeader() {
  noStroke();
  fill(0xFF101D2D);
  rect(0, 0, width, headerH);

  fill(WHITE);
  textAlign(LEFT, CENTER);
  textSize(18 * s);
  text("FUZZY WASH LAB 3D", 12 * s, headerH * 0.34);

  fill(CYAN);
  textSize(8.5 * s);
  text(
    "MAMDANI: FUZZIFY > MIN FIRING > MIN CLIP > MAX AGGREGATION > CENTROID",
    13 * s, headerH * 0.72
  );

  fill(running ? GREEN : ORANGE);
  textAlign(RIGHT, CENTER);
  textSize(10 * s);
  text(
    running ? "LIVE ANIMATION" : "PAUSED",
    width - 12 * s, headerH * 0.43
  );

  textAlign(LEFT, TOP);
}

void panel(float x, float y, float w, float h) {
  noStroke();
  fill(PANEL);
  rect(x, y, w, h, 8 * s);
}

void label(String t, float x, float y, float sz) {
  fill(MUTED);
  textAlign(LEFT, TOP);
  textSize(sz * s);
  text(t, x, y);
}

void value(String t, float x, float y, float sz, int c) {
  fill(c);
  textAlign(LEFT, TOP);
  textSize(sz * s);
  text(t, x, y);
}

void button(float x, float y, float w, float h,
            String t, int c) {
  noStroke();
  fill(c);
  rect(x, y, w, h, 5 * s);

  fill(WHITE);
  textAlign(CENTER, CENTER);
  textSize(8 * s);
  text(t, x + w / 2, y + h / 2);

  textAlign(LEFT, TOP);
}

void drawCell(float x, float y, float w, float h,
              String content, int bg, float size) {
  noStroke();
  fill(bg);
  rect(x + s, y + s, w - 2 * s, h - 2 * s, 3 * s);

  fill(bg == CYAN ? BG : WHITE);
  textAlign(CENTER, CENTER);
  textSize(size * s);
  text(content, x + w / 2, y + h / 2);

  textAlign(LEFT, TOP);
}

void drawMembershipBars(float x, float y,
                        float w, float[] vals) {
  float gap = 4 * s;
  float bw = (w - 2 * gap) / 3;

  for (int i = 0; i < 3; i++) {
    float bx = x + i * (bw + gap);

    noStroke();
    fill(PANEL2);
    rect(bx, y, bw, 9 * s, 3 * s);

    fill(i == 0 ? CYAN : (i == 1 ? ORANGE : GREEN));
    rect(bx, y, bw * vals[i], 9 * s, 3 * s);

    fill(WHITE);
    textAlign(CENTER, TOP);
    textSize(7 * s);
    text(
      names[i] + " " + nf(vals[i], 1, 2),
      bx + bw / 2, y + 11 * s
    );
  }

  textAlign(LEFT, TOP);
}

void slider(float x, float y, float w, float f) {
  stroke(PANEL2);
  strokeWeight(4 * s);
  strokeCap(ROUND);
  line(x, y, x + w, y);

  stroke(CYAN);
  line(x, y, x + w * f, y);

  noStroke();
  fill(WHITE);
  ellipse(x + w * f, y, 10 * s, 10 * s);
}

// ============================================================
// SCREEN 1 — DASHBOARD
// ============================================================

void drawLeft() {
  float x = leftX;
  float w = leftW;
  float y = headerH + 7 * s;

  panel(x, y, w, height * 0.39);

  label("1 INPUTS + FUZZIFICATION",
        x + 8 * s, y + 7 * s, 10);

  label("DIRT 0–10", x + 8 * s, y + 27 * s, 9);
  value(nf(dirt, 1, 1),
        x + w - 42 * s, y + 27 * s, 10, WHITE);

  slider(x + 9 * s, y + 45 * s,
         w - 18 * s, dirt / 10.0);

  label("LOAD 0–8 kg", x + 8 * s, y + 61 * s, 9);
  value(nf(loadKg, 1, 1) + " kg",
        x + w - 48 * s, y + 61 * s, 10, WHITE);

  slider(x + 9 * s, y + 79 * s,
         w - 18 * s, loadKg / 8.0);

  label("DIRT MEMBERSHIP μD",
        x + 8 * s, y + 98 * s, 8);

  drawMembershipBars(
    x + 8 * s, y + 113 * s,
    w - 16 * s, dirtMu
  );

  label("LOAD MEMBERSHIP μL",
        x + 8 * s, y + 149 * s, 8);

  drawMembershipBars(
    x + 8 * s, y + 164 * s,
    w - 16 * s, loadMu
  );

  label("LIVE GRADES: LOW / MED / HIGH",
        x + 8 * s, y + 199 * s, 7.5);

  value(
    nf(dirtMu[0], 1, 2) + "  " +
    nf(dirtMu[1], 1, 2) + "  " +
    nf(dirtMu[2], 1, 2),
    x + 8 * s, y + 212 * s, 8, CYAN
  );

  value(
    nf(loadMu[0], 1, 2) + "  " +
    nf(loadMu[1], 1, 2) + "  " +
    nf(loadMu[2], 1, 2),
    x + 8 * s, y + 225 * s, 8, ORANGE
  );

  float y2 = y + height * 0.405;
  panel(x, y2, w, height * 0.34);

  label("6 DEFUZZIFICATION",
        x + 8 * s, y2 + 7 * s, 10);

  value(nf(mamdaniTime, 1, 3) + " min",
        x + 8 * s, y2 + 27 * s, 22, GREEN);

  label("z* = Σ(z μagg) / Σμagg",
        x + 8 * s, y2 + 53 * s, 8);

  label("Samples = 161; Δz = 0.25",
        x + 8 * s, y2 + 68 * s, 8);

  label("Σ μagg = " + nf(areaSum, 1, 3),
        x + 8 * s, y2 + 84 * s, 8);

  label("Σ z μagg = " + nf(momentSum, 1, 3),
        x + 8 * s, y2 + 98 * s, 8);

  label("Area approx = " + nf(areaApprox, 1, 3),
        x + 8 * s, y2 + 112 * s, 8);

  label("Moment approx = " + nf(momentApprox, 1, 3),
        x + 8 * s, y2 + 126 * s, 8);

  label("Sugeno comparison (not centroid):",
        x + 8 * s, y2 + 145 * s, 7.5);

  value(nf(weightedTime, 1, 2) + " min",
        x + 8 * s, y2 + 158 * s, 8, ORANGE);
}

void drawCenter() {
  float x = leftX + leftW + 7 * s;
  float w = rightX - x - 7 * s;
  float y = headerH + 7 * s;

  fill(WHITE);
  textAlign(CENTER, TOP);
  textSize(11 * s);
  text("SMART WASHER • MAMDANI CONTROL",
       centerX, y + 3 * s);

  fill(CYAN);
  textSize(8 * s);
  text(
    "DIRT " + nf(dirt, 1, 1) +
    " / 10     LOAD " + nf(loadKg, 1, 1) + " kg",
    centerX, y + 19 * s
  );

  String[] stages = {
    "FUZZIFY", "FIRE MIN", "CLIP", "MAX AGG", "CENTROID"
  };

  float totalW = min(w * 0.93, height * 0.52);
  float bx = centerX - totalW / 2;
  float sy = height * 0.68;
  float sw = totalW / 5.0;

  for (int i = 0; i < 5; i++) {
    boolean hot = ((animStep / 18) % 5) == i && running;

    fill(hot ? 0xFF176C78 : PANEL2);
    noStroke();
    rect(bx + i * sw, sy, sw - 3 * s, 22 * s, 4 * s);

    fill(hot ? WHITE : MUTED);
    textAlign(CENTER, CENTER);
    textSize(7 * s);
    text(stages[i],
         bx + i * sw + (sw - 3 * s) / 2,
         sy + 11 * s);

    if (i < 4) {
      stroke(CYAN);
      strokeWeight(1.2 * s);
      line(
        bx + i * sw + sw - 3 * s, sy + 11 * s,
        bx + (i + 1) * sw, sy + 11 * s
      );
    }
  }

  float mx = centerX - 74 * s;
  float my = height * 0.735;

  label("RULE FIRING MATRIX αij",
        mx - 30 * s, my - 12 * s, 8);

  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      float cw = 48 * s;
      float ch = 24 * s;

      float xx = mx + j * (cw + 3 * s);
      float yy = my + i * (ch + 3 * s);

      boolean pulse =
        running && ((animStep / 12) % 9) == (i * 3 + j);

      int c = firing[i][j] > 0 ? 0xFF155C66 : PANEL2;
      if (pulse) c = CYAN;

      noStroke();
      fill(c);
      rect(xx, yy, cw, ch, 4 * s);

      fill(pulse ? BG : WHITE);
      textAlign(CENTER, CENTER);
      textSize(8 * s);
      text(nf(firing[i][j], 1, 2),
           xx + cw / 2, yy + ch / 2);
    }
  }

  label(
    "Output MAX: S " + nf(outputActivation[0], 1, 2) +
    " | N " + nf(outputActivation[1], 1, 2) +
    " | L " + nf(outputActivation[2], 1, 2),
    centerX - 100 * s, height * 0.855, 7.5
  );

  float by = height * 0.895;
  float bw = 72 * s;
  float bh = 24 * s;

  button(
    centerX - bw - 6 * s, by, bw, bh,
    running ? "PAUSE" : "START",
    running ? 0xFF176C78 : 0xFF256447
  );

  button(
    centerX + 6 * s, by, bw, bh,
    "RESET", 0xFF63452C
  );

  fill(MUTED);
  textAlign(CENTER, TOP);
  textSize(7 * s);
  text(
    "Tap bottom-left to open Screen 2 • drag sliders to change inputs",
    centerX, height * 0.94
  );

  textAlign(LEFT, TOP);
}

void drawRight() {
  float x = rightX;
  float w = rightW;
  float y = headerH + 7 * s;

  panel(x, y, w, height * 0.285);

  label("2 RULE MATRIX • AND = MIN",
        x + 7 * s, y + 6 * s, 9);

  label("Rows: dirt L/M/H; columns: load L/M/H",
        x + 7 * s, y + 21 * s, 6.5);

  float gx = x + 7 * s;
  float gy = y + 36 * s;
  float cw = (w - 14 * s) / 3;
  float ch = height * 0.045;

  for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
      float xx = gx + j * cw;
      float yy = gy + i * ch;

      int c = firing[i][j] > 0 ? 0xFF145965 : PANEL2;

      noStroke();
      fill(c);
      rect(xx, yy, cw - 3 * s, ch - 3 * s, 3 * s);

      fill(WHITE);
      textAlign(CENTER, CENTER);
      textSize(7.5 * s);
      text(
        "α=" + nf(firing[i][j], 1, 2),
        xx + (cw - 3 * s) / 2,
        yy + (ch - 3 * s) / 2
      );
    }
  }

  label(
    "Consequents: S S N / N N L / N L L",
    x + 7 * s, y + height * 0.225, 7
  );

  float y2 = y + height * 0.30;
  panel(x, y2, w, height * 0.39);

  label("3–5 CLIP + MAX AGGREGATION",
        x + 7 * s, y2 + 6 * s, 8.5);

  label(
    "μ' = min(α, μoutput); μagg = max(μ'S, μ'N, μ'L)",
    x + 7 * s, y2 + 20 * s, 6.2
  );

  float px = x + 10 * s;
  float py = y2 + 34 * s;
  float pw = w - 20 * s;
  float ph = height * 0.19;

  stroke(PANEL2);
  strokeWeight(1 * s);

  for (int q = 0; q <= 2; q++) {
    line(px, py + ph * q / 2, px + pw, py + ph * q / 2);
  }

  drawCurve(sampleShort, px, py, pw, ph, CYAN, 1.2 * s);
  drawCurve(sampleNormal, px, py, pw, ph, ORANGE, 1.2 * s);
  drawCurve(sampleLong, px, py, pw, ph, PURPLE, 1.2 * s);
  drawCurve(sampleAggregate, px, py, pw, ph, WHITE, 2 * s);

  float cx = px + pw * mamdaniTime / 40.0;

  stroke(RED);
  strokeWeight(1.5 * s);
  line(cx, py, cx, py + ph);

  noStroke();
  fill(WHITE);
  textSize(7 * s);
  textAlign(CENTER, TOP);
  text("0", px, py + ph + 2 * s);
  text("20", px + pw / 2, py + ph + 2 * s);
  text("40 min", px + pw, py + ph + 2 * s);
  textAlign(LEFT, TOP);

  label("SHORT", x + 8 * s, y2 + height * 0.255, 7);
  label("NORMAL", x + 66 * s, y2 + height * 0.255, 7);
  label("LONG", x + 130 * s, y2 + height * 0.255, 7);
  label("WHITE = MAX AGGREGATE",
        x + 8 * s, y2 + height * 0.285, 7);

  value(
    "Centroid = " + nf(mamdaniTime, 1, 3) + " min",
    x + 8 * s, y2 + height * 0.315, 9, GREEN
  );

  float y3 = y2 + height * 0.405;
  panel(x, y3, w, height * 0.25);

  label("CALCULATION TRACE",
        x + 7 * s, y3 + 6 * s, 9);

  label(
    "1. μD = [" + nf(dirtMu[0], 1, 2) + ", " +
    nf(dirtMu[1], 1, 2) + ", " + nf(dirtMu[2], 1, 2) + "]",
    x + 7 * s, y3 + 23 * s, 7
  );

  label(
    "2. μL = [" + nf(loadMu[0], 1, 2) + ", " +
    nf(loadMu[1], 1, 2) + ", " + nf(loadMu[2], 1, 2) + "]",
    x + 7 * s, y3 + 37 * s, 7
  );

  label("3. αij = min(μDi, μLj)",
        x + 7 * s, y3 + 51 * s, 7);

  label("4. Clip each consequent; combine using MAX",
        x + 7 * s, y3 + 65 * s, 7);

  label("5. z* = Σ(z μagg) / Σμagg",
        x + 7 * s, y3 + 79 * s, 7);

  label(
    "Σμ=" + nf(areaSum, 1, 2) +
    "  Σzμ=" + nf(momentSum, 1, 2),
    x + 7 * s, y3 + 93 * s, 7
  );
}

void drawCurve(float[] vals, float x, float y,
               float w, float h, int c, float sw) {
  noFill();
  stroke(c);
  strokeWeight(sw);

  beginShape();
  for (int k = 0; k < vals.length; k++) {
    float z = k * DZ;
    vertex(
      x + w * z / 40.0,
      y + h * (1 - vals[k])
    );
  }
  endShape();
}

// ============================================================
// SCREEN 2 — CANONICAL + NUMERICAL MATRIX CALCULATION LAB
// ============================================================

void drawCalculationScreen() {
  float W = width;
  float H = height;

  noStroke();
  fill(0xFF101D2D);
  rect(0, 0, W, H * 0.105);

  fill(WHITE);
  textAlign(LEFT, CENTER);
  textSize(15 * s);
  text("SCREEN 2 | CANONICAL + NUMERICAL LAB",
       12 * s, H * 0.030);

  fill(CYAN);
  textSize(7.5 * s);
  text(
    "FUZZIFICATION > RULE MATRIX > MIN IMPLICATION > MAX AGGREGATION > CENTROID",
    12 * s, H * 0.075
  );

  fill(running ? GREEN : ORANGE);
  textAlign(RIGHT, CENTER);
  textSize(8.5 * s);
  text(
    running ? "ANIMATION RUNNING" : "PAUSED",
    W - 12 * s, H * 0.035
  );

  float margin = 8 * s;
  float top = H * 0.12;
  float bottom = H * 0.91;
  float panelH = bottom - top;
  float leftW2 = W * 0.40;
  float gap = 7 * s;
  float rightX2 = margin + leftW2 + gap;
  float rightW2 = W - rightX2 - margin;

  panel(margin, top, leftW2, panelH);
  panel(rightX2, top, rightW2, panelH);

  drawCanonicalRulePanel(margin, top, leftW2, panelH);
  drawNumericalMatrixPanel(rightX2, top, rightW2, panelH);

  noStroke();
  fill(0xFF101D2D);
  rect(0, H * 0.925, W, H * 0.075);

  fill(MUTED);
  textAlign(LEFT, CENTER);
  textSize(7.5 * s);
  text(
    "TAP BOTTOM-LEFT: DASHBOARD | TAP TABLE LEFT/RIGHT: PREVIOUS/NEXT SAMPLE",
    12 * s, H * 0.962
  );

  fill(CYAN);
  textAlign(RIGHT, CENTER);
  text(
    "CENTROID = " + nf(mamdaniTime, 1, 4) + " min",
    W - 12 * s, H * 0.962
  );

  textAlign(LEFT, TOP);

  if (running && frameCount % 5 == 0) {
    numericalSample++;
    if (numericalSample >= NUM_SAMPLES) numericalSample = 0;
  }
}

void drawCanonicalRulePanel(float x, float y,
                            float w, float h) {
  float pad = 8 * s;

  label("A. CANONICAL RULE BASE",
        x + pad, y + 7 * s, 9);

  value(
    "Inputs: dirt=" + nf(dirt, 1, 2) +
    "   load=" + nf(loadKg, 1, 2) + " kg",
    x + pad, y + 23 * s, 7.5, CYAN
  );

  label("AND = MIN; αij = min(μD_i, μL_j)",
        x + pad, y + 38 * s, 7);

  float mx = x + pad;
  float my = y + 54 * s;
  float tableW = w - 2 * pad;
  float cellW = tableW / 4.0;
  float cellH = 22 * s;

  drawCell(mx, my, cellW, cellH,
           "DIRT/LOAD", PANEL2, 6);

  for (int j = 0; j < 3; j++) {
    drawCell(
      mx + (j + 1) * cellW, my,
      cellW, cellH, loadLabels[j], PANEL2, 6.5
    );
  }

  for (int i = 0; i < 3; i++) {
    drawCell(
      mx, my + (i + 1) * cellH,
      cellW, cellH, dirtLabels[i], PANEL2, 6.5
    );

    for (int j = 0; j < 3; j++) {
      int r = i * 3 + j;

      boolean pulse =
        running && ((animStep / 10) % 9 == r);

      int c = firing[i][j] > 0 ? 0xFF15515E : PANEL2;
      if (pulse) c = CYAN;

      String cellText =
        "R" + (r + 1) +
        " α=" + nf(firing[i][j], 1, 2) +
        "\n" + consequentLabels[rules[i][j]];

      drawCell(
        mx + (j + 1) * cellW,
        my + (i + 1) * cellH,
        cellW, cellH, cellText, c, 6.2
      );
    }
  }

  float yy = my + 4 * cellH + 7 * s;

  label("B. NINE CANONICAL IF–THEN RULES",
        x + pad, yy, 8);

  yy += 14 * s;

  String[] ruleText = {
    "R1 IF dirt LOW AND load LOW THEN SHORT",
    "R2 IF dirt LOW AND load MED THEN SHORT",
    "R3 IF dirt LOW AND load HIGH THEN NORMAL",
    "R4 IF dirt MED AND load LOW THEN NORMAL",
    "R5 IF dirt MED AND load MED THEN NORMAL",
    "R6 IF dirt MED AND load HIGH THEN LONG",
    "R7 IF dirt HIGH AND load LOW THEN NORMAL",
    "R8 IF dirt HIGH AND load MED THEN LONG",
    "R9 IF dirt HIGH AND load HIGH THEN LONG"
  };

  for (int r = 0; r < 9; r++) {
    fill(
      running && ((animStep / 10) % 9 == r)
      ? CYAN : WHITE
    );

    textAlign(LEFT, TOP);
    textSize(6.0 * s);
    text(ruleText[r], x + pad, yy);
    yy += 11.2 * s;
  }

  yy += 1 * s;

  label("C. FUZZIFICATION VECTORS",
        x + pad, yy, 7.5);

  yy += 13 * s;

  value(
    "μD = [" + nf(dirtMu[0], 1, 3) + ", " +
    nf(dirtMu[1], 1, 3) + ", " +
    nf(dirtMu[2], 1, 3) + "]",
    x + pad, yy, 7.3, CYAN
  );

  yy += 13 * s;

  value(
    "μL = [" + nf(loadMu[0], 1, 3) + ", " +
    nf(loadMu[1], 1, 3) + ", " +
    nf(loadMu[2], 1, 3) + "]",
    x + pad, yy, 7.3, ORANGE
  );

  yy += 15 * s;

  label("D. CONSEQUENT ACTIVATION = MAX",
        x + pad, yy, 7.5);

  yy += 13 * s;

  value(
    "SHORT " + nf(outputActivation[0], 1, 3) +
    " | NORMAL " + nf(outputActivation[1], 1, 3) +
    " | LONG " + nf(outputActivation[2], 1, 3),
    x + pad, yy, 6.8, GREEN
  );

  yy += 14 * s;

  label("E. MAMDANI EQUATIONS",
        x + pad, yy, 7.5);

  yy += 12 * s;

  label("μ'out(z) = min(αrule, μout(z))",
        x + pad, yy, 6.8);

  yy += 11 * s;

  label("μagg(z) = max(μ'S, μ'N, μ'L)",
        x + pad, yy, 6.8);

  textAlign(LEFT, TOP);
}

void drawNumericalMatrixPanel(float x, float y,
                              float w, float h) {
  float pad = 8 * s;

  label("F. NUMERICAL OUTPUT MATRIX",
        x + pad, y + 7 * s, 9);

  label("z = kΔz; Δz=0.25 min; k=0...160",
        x + pad, y + 22 * s, 6.8);

  label(
    "Each row evaluates clipping, MAX aggregation and z·μagg.",
    x + pad, y + 35 * s, 6.5
  );

  float tx = x + pad;
  float ty = y + 49 * s;
  float tableW = w - 2 * pad;

  float[] widths = {
    tableW * 0.09,
    tableW * 0.12,
    tableW * 0.14,
    tableW * 0.14,
    tableW * 0.14,
    tableW * 0.15,
    tableW * 0.22
  };

  String[] heads = {
    "k", "z", "SHORT", "NORMAL", "LONG", "AGG", "z × AGG"
  };

  float cx = tx;
  float hh = 18 * s;

  for (int c = 0; c < heads.length; c++) {
    drawCell(cx, ty, widths[c], hh,
             heads[c], PANEL2, 5.7);
    cx += widths[c];
  }

  float rowH = 18 * s;
  int visibleRows = 7;

  int start = constrain(
    numericalSample - 3, 0,
    NUM_SAMPLES - visibleRows
  );

  for (int row = 0; row < visibleRows; row++) {
    int k = start + row;
    float ry = ty + hh + row * rowH;

    cx = tx;

    String[] vals = {
      str(k),
      nf(sampleZ[k], 1, 2),
      nf(sampleShort[k], 1, 3),
      nf(sampleNormal[k], 1, 3),
      nf(sampleLong[k], 1, 3),
      nf(sampleAggregate[k], 1, 3),
      nf(sampleMoment[k], 1, 4)
    };

    int bg = k == numericalSample
      ? 0xFF176C78 : PANEL2;

    for (int c = 0; c < vals.length; c++) {
      drawCell(
        cx, ry, widths[c], rowH,
        vals[c], bg, 5.8
      );
      cx += widths[c];
    }
  }

  float by = ty + hh + visibleRows * rowH + 7 * s;

  label("G. SELECTED SAMPLE — STEP BY STEP",
        x + pad, by, 8);

  by += 14 * s;

  int k = numericalSample;
  float z = sampleZ[k];

  value(
    "k=" + k + "  z=kΔz=" + nf(z, 1, 2) + " min",
    x + pad, by, 7.3, CYAN
  );

  by += 13 * s;

  label(
    "SHORT: min(" + nf(outputActivation[0], 1, 3) +
    ", tri(z)) = " + nf(sampleShort[k], 1, 4),
    x + pad, by, 6.4
  );

  by += 12 * s;

  label(
    "NORMAL: min(" + nf(outputActivation[1], 1, 3) +
    ", tri(z)) = " + nf(sampleNormal[k], 1, 4),
    x + pad, by, 6.4
  );

  by += 12 * s;

  label(
    "LONG: min(" + nf(outputActivation[2], 1, 3) +
    ", tri(z)) = " + nf(sampleLong[k], 1, 4),
    x + pad, by, 6.4
  );

  by += 12 * s;

  value(
    "μagg=" + nf(sampleAggregate[k], 1, 4) +
    "   zμagg=" + nf(sampleMoment[k], 1, 4),
    x + pad, by, 6.8, ORANGE
  );

  by += 15 * s;

  label("H. DISCRETE CENTROID SUMS",
        x + pad, by, 8);

  by += 13 * s;

  label("Σμagg = " + nf(areaSum, 1, 5),
        x + pad, by, 7);

  by += 12 * s;

  label("Σ(zμagg) = " + nf(momentSum, 1, 5),
        x + pad, by, 7);

  by += 13 * s;

  value(
    "z* = Σ(zμagg)/Σμagg = " +
    nf(mamdaniTime, 1, 5) + " min",
    x + pad, by, 7.5, GREEN
  );

  by += 13 * s;

  label(
    "Area ≈ ΔzΣμagg = " + nf(areaApprox, 1, 4),
    x + pad, by, 6.5
  );

  by += 11 * s;

  label(
    "Moment ≈ ΔzΣ(zμagg) = " + nf(momentApprox, 1, 4),
    x + pad, by, 6.5
  );

  textAlign(LEFT, TOP);
}

// ============================================================
// FOOTER
// ============================================================

void drawFooter() {
  fill(MUTED);
  textAlign(CENTER, CENTER);
  textSize(6.8 * s);

  text(
    "MAMDANI: FUZZIFICATION > RULE FIRING (MIN) > " +
    "IMPLICATION (MIN) > AGGREGATION (MAX) > CENTROID",
    width / 2, height - 7 * s
  );

  textAlign(LEFT, TOP);
}

// ============================================================
// TOUCH AND KEYBOARD CONTROLS
// ============================================================

void mousePressed() {
  // Screen 2 navigation and sample selection.
  if (screenMode == 2) {
    if (mouseY >= height * 0.925 &&
        mouseX <= width * 0.30) {
      screenMode = 1;
      return;
    }

    float tableTop =
      height * 0.12 + 49 * s + 18 * s;

    float tableBottom =
      tableTop + 7 * 18 * s;

    if (mouseY >= tableTop && mouseY <= tableBottom) {
      if (mouseX < width * 0.70) {
        numericalSample =
          max(0, numericalSample - 1);
      } else {
        numericalSample =
          min(NUM_SAMPLES - 1, numericalSample + 1);
      }
      return;
    }

    if (mouseY < height * 0.105 &&
        mouseX > width * 0.75) {
      running = !running;
    }

    return;
  }

  // Screen 1 bottom-left navigation to Screen 2.
  if (mouseY >= height * 0.925 &&
      mouseX <= width * 0.30) {
    screenMode = 2;
    numericalSample = 0;
    return;
  }

  if (hit(
    centerX - 78 * s,
    height * 0.895,
    72 * s,
    24 * s
  )) {
    running = !running;
    return;
  }

  if (hit(
    centerX + 6 * s,
    height * 0.895,
    72 * s,
    24 * s
  )) {
    dirt = 6;
    loadKg = 4;
    spinAngle = 0;
    animStep = 0;
    numericalSample = 0;
    running = true;

    calculateFuzzySystem();
    return;
  }

  float x = leftX;
  float w = leftW;
  float y = headerH + 7 * s;

  if (mouseX >= x && mouseX <= x + w) {
    if (abs(mouseY - (y + 45 * s)) < 14 * s) {
      setDirt(mouseX, x + 9 * s, w - 18 * s);
      return;
    }

    if (abs(mouseY - (y + 79 * s)) < 14 * s) {
      setLoad(mouseX, x + 9 * s, w - 18 * s);
      return;
    }
  }

  if (mouseX > leftX + leftW &&
      mouseX < rightX &&
      mouseY > headerH &&
      mouseY < height * 0.66) {
    running = !running;
  }
}

void mouseDragged() {
  if (screenMode != 1) return;

  float x = leftX;
  float w = leftW;
  float y = headerH + 7 * s;

  if (mouseX >= x && mouseX <= x + w) {
    if (abs(mouseY - (y + 45 * s)) < 18 * s) {
      setDirt(mouseX, x + 9 * s, w - 18 * s);
    } else if (abs(mouseY - (y + 79 * s)) < 18 * s) {
      setLoad(mouseX, x + 9 * s, w - 18 * s);
    }
  }
}

boolean hit(float x, float y, float w, float h) {
  return mouseX >= x && mouseX <= x + w &&
         mouseY >= y && mouseY <= y + h;
}

void setDirt(float mx, float sx, float sw) {
  dirt = constrain(map(mx, sx, sx + sw, 0, 10), 0, 10);
  calculateFuzzySystem();
}

void setLoad(float mx, float sx, float sw) {
  loadKg = constrain(map(mx, sx, sx + sw, 0, 8), 0, 8);
  calculateFuzzySystem();
}

void keyPressed() {
  if (key == '1') screenMode = 1;
  if (key == '2') screenMode = 2;
  if (key == ' ') running = !running;

  if (key == 'r' || key == 'R') {
    dirt = 6;
    loadKg = 4;
    numericalSample = 0;
    spinAngle = 0;
    animStep = 0;
    running = true;
    calculateFuzzySystem();
  }

  if (keyCode == LEFT) {
    numericalSample = max(0, numericalSample - 1);
  }

  if (keyCode == RIGHT) {
    numericalSample =
      min(NUM_SAMPLES - 1, numericalSample + 1);
  }
}
