#include <Adafruit_NeoPixel.h>
#include <DFRobotDFPlayerMini.h>
#include <SoftwareSerial.h>

#define LED_PIN_1 6        // Broche pour la première bande LED
#define LED_PIN_2 7        // Broche pour la deuxième bande LED
#define LED_PIN_3 8        // Broche pour la troisième bande LED
#define NUM_LEDS 7         // Nombre total de LEDs par bande
#define MIC_PIN A0         // Broche pour le microphone
#define POT_SENS_PIN A2    // Potentiomètre pour la sensibilité
#define POT_LUM_PIN A1     // Potentiomètre pour la luminosité
#define DFPLAYER_RX 10     // RX du MP3-TF-16P sur D10
#define DFPLAYER_TX 11     // TX du MP3-TF-16P sur D11

// Objets pour les trois bandes de LEDs et le DFPlayer
Adafruit_NeoPixel strip1 = Adafruit_NeoPixel(NUM_LEDS, LED_PIN_1, NEO_GRB + NEO_KHZ800);
Adafruit_NeoPixel strip2 = Adafruit_NeoPixel(NUM_LEDS, LED_PIN_2, NEO_GRB + NEO_KHZ800);
Adafruit_NeoPixel strip3 = Adafruit_NeoPixel(NUM_LEDS, LED_PIN_3, NEO_GRB + NEO_KHZ800);
SoftwareSerial mySoftwareSerial(DFPLAYER_TX, DFPLAYER_RX); // Liaison série pour le MP3-TF-16P
DFRobotDFPlayerMini myDFPlayer;

const unsigned long playInterval = 15000; // Intervalle de 15 secondes pour réautoriser la lecture
unsigned long lastPlayTime = 0; // Moment de la dernière lecture
int ledLevels[NUM_LEDS] = {0};  // Niveaux de luminosité pour chaque LED

void testSequence() {
  uint32_t colors[] = {
    strip1.Color(255, 0, 0),    // Rouge
    strip1.Color(0, 255, 0),    // Vert
    strip1.Color(0, 0, 255),    // Bleu
    strip1.Color(255, 255, 0)   // Jaune
  };

  for (int repeat = 0; repeat < 2; repeat++) {   // Effectuer 2 répétitions de la séquence
    for (int c = 0; c < 4; c++) {                // Boucle pour chaque couleur
      for (int i = 0; i < NUM_LEDS; i++) {       // Monter de bas en haut
        strip1.clear();
        strip2.clear();
        strip3.clear();
        strip1.setPixelColor(i, colors[c]);
        strip2.setPixelColor(i, colors[c]);
        strip3.setPixelColor(i, colors[c]);
        strip1.show();
        strip2.show();
        strip3.show();
        delay(100);                              // Pause pour chaque étape du chenillard
      }
      for (int i = NUM_LEDS - 1; i >= 0; i--) {  // Descendre de haut en bas
        strip1.clear();
        strip2.clear();
        strip3.clear();
        strip1.setPixelColor(i, colors[c]);
        strip2.setPixelColor(i, colors[c]);
        strip3.setPixelColor(i, colors[c]);
        strip1.show();
        strip2.show();
        strip3.show();
        delay(100);                              // Pause pour chaque étape du chenillard
      }
    }
  }
}

void setup() {
  Serial.begin(9600);
  mySoftwareSerial.begin(9600);

  // Initialisation du lecteur MP3
  if (!myDFPlayer.begin(mySoftwareSerial)) {  
    Serial.println("Erreur : MP3-TF-16P non détecté !");
    while (true);  // Bloque le programme si le lecteur MP3 n'est pas détecté
  }
  
  Serial.println("MP3-TF-16P détecté avec succès !");
  myDFPlayer.volume(30);       // Fixe le son à 30 (maximum)

  // Délai d'attente pour la stabilisation du DFPlayer
  delay(2000);

  // Initialisation des bandes de LEDs
  strip1.begin();
  strip2.begin();
  strip3.begin();
  strip1.show(); 
  strip2.show();
  strip3.show();

  // Séquence de test au démarrage
  testSequence();

  // Configuration des broches
  pinMode(MIC_PIN, INPUT);
  pinMode(POT_LUM_PIN, INPUT);
  pinMode(POT_SENS_PIN, INPUT);
}

void loop() {
  unsigned long currentTime = millis();

  int potLumValue = analogRead(POT_LUM_PIN);
  int brightness = map(potLumValue, 0, 1023, 255, 10);
  bool mp3Enabled = brightness < 255;

  uint32_t witnessColor = strip1.Color(0, brightness, 0);
  strip1.setPixelColor(0, witnessColor);
  strip2.setPixelColor(0, witnessColor);
  strip3.setPixelColor(0, witnessColor);

  int micValue = analogRead(MIC_PIN);
  int potSensValue = analogRead(POT_SENS_PIN);
  int sensitivity = map(potSensValue, 0, 1023, 30, 750);

  int numLedsToLight = map(micValue, sensitivity, 1000, 0, NUM_LEDS);
  numLedsToLight = constrain(numLedsToLight, 0, NUM_LEDS);

  for (int i = 1; i < NUM_LEDS; i++) {
    if (numLedsToLight > i) {
      ledLevels[i] = brightness;
    } else {
      ledLevels[i] = max(0, ledLevels[i] - 20);
    }
  }

  for (int i = 1; i < NUM_LEDS; i++) {
    uint32_t color;
    if (i < 3) {
      color = strip1.Color(0, ledLevels[i], 0); 
    } else if (i < 5) {
      color = strip1.Color(ledLevels[i], ledLevels[i] / 2, 0);
    } else {
      color = strip1.Color(ledLevels[i], 0, 0); 
    }

    strip1.setPixelColor(i, color);
    strip2.setPixelColor(i, color);
    strip3.setPixelColor(i, color);
  }

  strip1.show();
  strip2.show();
  strip3.show();

  if (mp3Enabled && (numLedsToLight == NUM_LEDS) && (currentTime - lastPlayTime >= playInterval)) {
    myDFPlayer.play(1);
    lastPlayTime = currentTime;
    Serial.println("Lecture du MP3 démarrée.");
  }

  delay(10);
}
