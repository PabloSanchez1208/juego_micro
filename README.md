/*
    Juego "Rock Breaker"
    Tamaño de pantalla: 240x320 pixeles
*/

// ==========================
// Librerías
// ==========================
#include <Adafruit_GFX.h>
#include <Adafruit_ILI9341.h>

// ==========================
// Definición de Pines
// ==========================
#define TFT_DC   7
#define TFT_CS   6
#define TFT_MOSI 11
#define TFT_CLK  13
#define TFT_RST  10
#define TFT_MISO 12

#define BTN_LEFT  18
#define BTN_RIGHT 19
#define BOTON_DISPARO 2

#define PIN_BUZZER 8

// ==========================
// Objetos de la pantalla
// ==========================
Adafruit_ILI9341 screen = Adafruit_ILI9341(TFT_CS, TFT_DC, TFT_MOSI, TFT_CLK, TFT_RST, TFT_MISO);

// ==========================
// Inclusión de Clases
// ==========================
#include "roca.h" 
#include "nave.h"
#include "bala.h"

// ==========================
// Objetos del juego
// ==========================
Nave jugador(100, 280);

const int NUM_ROCAS = 5;
Roca rocas[NUM_ROCAS] = {
  Roca(random(0, 240), 0, random(6, 12)),
  Roca(random(0, 240), -30, random(6, 12)),
  Roca(random(0, 240), -60, random(6, 12)),
  Roca(random(0, 240), -90, random(6, 12)),
  Roca(random(0, 240), -120, random(6, 12)),
};

const int MAX_BALAS = 10;
Bala* balas[MAX_BALAS];
int numBalas = 0;

// ==========================
// Variables de tiempo y juego
// ==========================
unsigned long lastUpdate = 0;
const int frameDelay = 5;
int vidas = 2;
int puntaje = 0;

// ==========================
// Música
// ==========================
int melody[] = {
  262, 294, 330, 349, 392, 440, 392, 349,
  330, 294, 262, 294, 330, 262
};
int noteDurations[] = {
  8, 8, 4, 8, 4, 4, 8, 8,
  4, 4, 4, 8, 4, 2
};

int currentNote = 0;
unsigned long noteStartTime = 0;
bool musicaActiva = true;

// ==========================
// Funciones auxiliares organizadas
// ==========================

void mostrarPantallaCarga() {
  screen.setTextColor(ILI9341_WHITE);
  screen.setTextSize(2);
  screen.setCursor(20, 50);
  screen.print("Cargando...");

  int barX = 10, barY = 100, barW = 200, barH = 20;
  screen.drawRect(barX, barY, barW, barH, ILI9341_WHITE);
  for (int i = 0; i <= barW - 4; i += 4) {
    screen.fillRect(barX + 2, barY + 2, i, barH - 4, ILI9341_GREEN);
    delay(50);
  }

  delay(1000);
}

void pantallaPrincipal() {
  screen.fillScreen(ILI9341_BLACK);
  screen.setCursor(5, 60);
  screen.setTextColor(ILI9341_YELLOW);
  screen.setTextSize(3);
  screen.print("ROCK BREAKER!");
}

void pantallaPerder() {
  musicaActiva = false;
  noTone(PIN_BUZZER);

  screen.fillScreen(ILI9341_BLACK);
  screen.setCursor(30, 80);
  screen.setTextColor(ILI9341_RED);
  screen.setTextSize(3);
  screen.print("¡PERDISTE!");

  screen.setCursor(30, 130);
  screen.setTextColor(ILI9341_WHITE);
  screen.setTextSize(2);
  screen.print("Puntaje final: ");
  screen.print(puntaje);
}

void configurarPines() {
  pinMode(BTN_LEFT, INPUT_PULLUP);
  pinMode(BTN_RIGHT, INPUT_PULLUP);
  pinMode(BOTON_DISPARO, INPUT_PULLUP);
  pinMode(PIN_BUZZER, OUTPUT);
}

void inicializarJuego() {
  jugador.dibujar();
  for (int i = 0; i < NUM_ROCAS; i++) {
    rocas[i].dibujar();
  }
}

bool colisionBalaRoca(Bala& bala, Roca& roca) {
  return !(
    bala.getX() + 3 < roca.getX() ||
    bala.getX() > roca.getX() + ROCA_WIDTH ||
    bala.getY() + 6 < roca.getY() ||
    bala.getY() > roca.getY() + ROCA_HEIGHT
  );
}

bool colisiona(const Nave& nave, const Roca& roca) {
  return !(
    nave.getX() + NAVE_WIDTH < roca.getX() ||
    nave.getX() > roca.getX() + ROCA_WIDTH ||
    nave.getY() + NAVE_HEIGHT < roca.getY() ||
    nave.getY() > roca.getY() + ROCA_HEIGHT
  );
}

// ==========================
// Setup del juego
// ==========================
void setup() {
  screen.begin();
  screen.setRotation(0);
  screen.fillScreen(ILI9341_BLACK);

  mostrarPantallaCarga();
  pantallaPrincipal();
  delay(1000);
  screen.fillScreen(ILI9341_BLACK);

  configurarPines();
  inicializarJuego();
}

// ==========================
// Loop principal del juego
// ==========================
void loop() {
  unsigned long now = millis();
  if (now - lastUpdate < frameDelay) return;
  lastUpdate = now;

  // Música
  if (musicaActiva) {
    int duration = 1000 / noteDurations[currentNote];
    if (now - noteStartTime > duration * 1.3) {
      tone(PIN_BUZZER, melody[currentNote], duration);
      noteStartTime = now;
      currentNote = (currentNote + 1) % 8;
    }
  }

  // Movimiento del jugador
  if (digitalRead(BTN_LEFT) == LOW) jugador.moverIzquierda();
  if (digitalRead(BTN_RIGHT) == LOW) jugador.moverDerecha();

  // Rocas
  for (int i = 0; i < NUM_ROCAS; i++) {
    rocas[i].actualizar();

    if (colisiona(jugador, rocas[i])) {
      if (vidas > 1) {
        vidas--;
        jugador.reiniciar();
        for (int j = 0; j < NUM_ROCAS; j++) rocas[j].reiniciar();
      } else {
        pantallaPerder();
        while (true); // Fin del juego
      }
    }

    if (rocas[i].fueraDePantalla()) rocas[i].reiniciar();
  }

  // Disparo
  if (digitalRead(BOTON_DISPARO) == LOW) {
    if (numBalas < MAX_BALAS) {
      balas[numBalas] = new Bala(jugador.getX() + NAVE_WIDTH / 2 - 1, jugador.getY() - 6);
      numBalas++;
    }
    delay(100); // Antirebote
  }

  // Balas
  for (int i = 0; i < numBalas; i++) {
    balas[i]->mover();
    balas[i]->dibujar(screen);
  }

  // Colisiones bala-roca
  for (int i = 0; i < numBalas; i++) {
    if (!balas[i]->estaActiva()) continue;
    for (int j = 0; j < NUM_ROCAS; j++) {
      if (colisionBalaRoca(*balas[i], rocas[j])) {
        rocas[j].reiniciar();
        balas[i]->borrar();
        puntaje++;
        break;
      }
    }
  }

  // Eliminar balas inactivas
  for (int i = 0; i < numBalas; i++) {
    if (!balas[i]->estaActiva()) {
      delete balas[i];
      for (int j = i; j < numBalas - 1; j++) {
        balas[j] = balas[j + 1];
      }
      numBalas--;
      i--;
    }
  }
}
