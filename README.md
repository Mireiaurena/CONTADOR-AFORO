# CONTADOR-AFORO
# 📊 Contador de Aforo Inteligente

Este proyecto implementa un sistema inteligente de control de aforo utilizando un microcontrolador **ESP32**, pulsadores físicos, una pantalla OLED y una interfaz web en tiempo real.

## 🚀 Objetivos

- Llevar un conteo de personas que entran y salen de un espacio físico.
- Activar una alerta visual cuando se supera un límite de 30 personas.
- Mostrar en una pantalla OLED el número actual de personas o el mensaje **"AFORO LLENO"**.
- Visualizar y controlar el aforo desde una página web embebida en el ESP32.

## 🧰 Componentes Utilizados

- ESP32
- 2 Pulsadores (Incrementar / Decrementar)
- 1 Pantalla OLED (SSD1306, I2C)
- 1 LED rojo (aforo lleno)
- 1 LED verde (aforo permitido)
- Cables de conexión
- Plataforma: PlatformIO

## ⚙️ Funcionamiento

- Cada pulsador controla el ingreso o la salida de una persona.
- Se actualiza la cuenta en la pantalla OLED y en la web.
- El LED verde se enciende si el aforo es inferior al máximo.
- El LED rojo se enciende cuando se alcanza el límite.
- La web muestra en tiempo real el estado del aforo, con botones para sumar o restar remotamente.

## 🌐 Interfaz Web

- Página HTML, CSS y JavaScript embebida en el ESP32.
- Actualización automática cada segundo (`fetch` + `setInterval`).
- Botones ➕ y ➖ controlan remotamente el aforo.
- Indicador visual del estado: **"✅ Espacio disponible"** o **"🚫 ¡Lleno!"**.

## 📄 Estructura del Código

``` cpp
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// Pines de botones y LEDs
const int buttonPlusPin = 18;
const int buttonMinusPin = 16;
const int ledGreenPin = 48;
const int ledRedPin = 12;

// OLED display settings
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Contador
const int maxCount = 30;
int count = 0;

// WiFi
const char* ssid = "DIGIFIBRA-HsA7";      // Reemplaza con tu SSID real
const char* password = "dsQRX3Uuuh";       // Reemplaza con tu contraseña real
WebServer server(80);

// Estados botones
bool lastButtonPlusState = HIGH;
bool lastButtonMinusState = HIGH;

// Declaraciones
void mostrarEstado();
void checkButtonPlus();
void checkButtonMinus();
void actualizarIndicadores();
void handleRoot();
void handleAdd();
void handleSubtract();
void handleCount();

void setup() {
  Serial.begin(115200);

  // Configurar I2C en los pines 21 (SDA) y 20 (SCL)
  Wire.begin(21, 20);

  // Inicializar pantalla OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { // Dirección 0x3C típica
    Serial.println(F("❌ No se encontró la pantalla OLED"));
    while (true); // Queda detenido si no encuentra la pantalla
  }
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Contador:");
  display.setCursor(0, 30);
  display.print(count);
  display.display();

  // Pines de entrada/salida
  pinMode(buttonPlusPin, INPUT_PULLUP);
  pinMode(buttonMinusPin, INPUT_PULLUP);
  pinMode(ledGreenPin, OUTPUT);
  pinMode(ledRedPin, OUTPUT);
  digitalWrite(ledGreenPin, LOW);
  digitalWrite(ledRedPin, LOW);

  // Conectar al WiFi
  Serial.print("🔌 Conectando a WiFi: ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);

  // Esperar hasta que se conecte a la red WiFi
  int retries = 0;
  while (WiFi.status() != WL_CONNECTED && retries < 20) {
    delay(500);
    Serial.print(".");
    retries++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n✅ Conectado a WiFi");
    Serial.print("🌐 Dirección IP: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("\n❌ Error al conectar a WiFi. Revisa SSID o clave.");
    Serial.print("Código de error: ");
    Serial.println(WiFi.status());
  }

  // Configuración del servidor web
  server.on("/", handleRoot);
  server.on("/add", handleAdd);
  server.on("/subtract", handleSubtract);
  server.on("/count", handleCount);

  server.begin();
  Serial.println("🌐 Servidor web iniciado.");
}

void loop() {
  server.handleClient();
  checkButtonPlus();
  checkButtonMinus();
  actualizarIndicadores();
}

void checkButtonPlus() {
  int reading = digitalRead(buttonPlusPin);
  if (reading == LOW && lastButtonPlusState == HIGH) {
    if (count < maxCount) {
      count++;
      mostrarEstado();
    } else {
      Serial.println("⚠️ Límite alcanzado.");
    }
    delay(200);
  }
  lastButtonPlusState = reading;
}

void checkButtonMinus() {
  int reading = digitalRead(buttonMinusPin);
  if (reading == LOW && lastButtonMinusState == HIGH) {
    if (count > 0) {
      count--;
      mostrarEstado();
    }
    delay(200);
  }
  lastButtonMinusState = reading;
}

void actualizarIndicadores() {
  digitalWrite(ledGreenPin, count < maxCount ? HIGH : LOW);
  digitalWrite(ledRedPin, count >= maxCount ? HIGH : LOW);
}

void mostrarEstado() {
  Serial.print("📟 Conteo actual: ");
  Serial.println(count);

  // Actualizar pantalla OLED
  display.clearDisplay();
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);

  if (count >= maxCount) {
    display.print("AFORO");
    display.setCursor(0, 30);
    display.print("LLENO");
  } else {
    display.print("Personas:");
    display.setCursor(0, 30);
    display.print(count);
  }
  
  display.display();
}

void handleRoot() {
  String html = R"rawliteral(
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="utf-8" />
      <title>Contador Inteligente</title>
      <style>
        body {
          font-family: 'Segoe UI', sans-serif;
          background: linear-gradient(to right, #1e3c72, #2a5298);
          color: white;
          text-align: center;
          padding: 40px;
        }
        h1 {
          font-size: 60px;
        }
        .status {
          font-size: 40px;
          margin-top: 20px;
          font-weight: bold;
        }
        .buttons {
          margin-top: 30px;
        }
        button {
          font-size: 30px;
          padding: 15px 30px;
          margin: 10px;
          border: none;
          border-radius: 10px;
          cursor: pointer;
          transition: background 0.3s;
        }
        .sumar {
          background-color: #4CAF50;
          color: white;
        }
        .restar {
          background-color: #f44336;
          color: white;
        }
        button:hover {
          opacity: 0.9;
        }
      </style>
    </head>
    <body>
      <h1>🚀 Contador en Tiempo Real</h1>
      <div class="status">Personas dentro: <span id="count">...</span></div>
      <div class="status" id="estado">⏳ Cargando...</div>
      <div class="buttons">
        <button class="sumar" onclick="modificar('add')">➕ Sumar</button>
        <button class="restar" onclick="modificar('subtract')">➖ Restar</button>
      </div>

      <script>
        function actualizar() {
          fetch('/count')
            .then(res => res.json())
            .then(data => {
              document.getElementById('count').innerText = data.count;
              document.getElementById('estado').innerText =
                data.count < data.max ? '✅ Espacio disponible' : '🚫 ¡Lleno!';
              document.getElementById('estado').style.color =
                data.count < data.max ? 'lightgreen' : 'tomato';
            });
        }

        function modificar(action) {
          fetch('/' + action)
            .then(() => actualizar());
        }

        setInterval(actualizar, 1000);
        actualizar();
      </script>
    </body>
    </html>
  )rawliteral";

  server.send(200, "text/html", html);
}

void handleAdd() {
  if (count < maxCount) {
    count++;
    mostrarEstado();
  }
  server.send(200, "text/plain", "OK");
}

void handleSubtract() {
  if (count > 0) {
    count--;
    mostrarEstado();
  }
  server.send(200, "text/plain", "OK");
}

void handleCount() {
  String json = "{ \"count\": " + String(count) + ", \"max\": " + String(maxCount) + " }";
  server.send(200, "application/json", json);
}
```

### setup()
- Inicializa pantalla OLED, pines y conexión WiFi.
- Configura el servidor web.

### loop()
- Escanea continuamente los botones físicos.
- Atiende peticiones web.

### Funciones principales:
- `checkButtonPlus()`, `checkButtonMinus()` – control de aforo físico.
- `mostrarEstado()` – actualiza OLED.
- `actualizarIndicadores()` – gestiona los LEDs.
- Rutas web: `/`, `/add`, `/subtract`, `/count`.


## 🛠 Requisitos

- PlatformIO
- Librerías:
  - `Adafruit_GFX`
  - `Adafruit_SSD1306`
  - `WiFi.h`
  - `WebServer.h`
  - `Wire.h`



