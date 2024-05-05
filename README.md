#include <ESP8266WiFi.h>
#include "HX711.h"

// Dane sieciowe
const char* ssid = "Orange_Swiatlowod_2C0C2g";
const char* password = "26USLJC57WV3";
const int serverPort = 80; // Port HTTP

// Definicje pinów HX711
const int HX711_dout_pin = D1; // podłączony do pinu D1
const int HX711_sck_pin = D0; // podłączony do pinu D0

// Deklaracja obiektu HX711
HX711 scale;

// Liczba próbek użytych do obliczenia średniej
const int numSamples = 1000;
const int discardSamples = 5;
float samples[numSamples];
int sampleIndex = 0;

// Wartość tarowania i współczynnik skali
float tareValue = 0;
float scaleFactor = 1;

WiFiServer server(serverPort);
int visits = 0; // Licznik odwiedzin strony

void setup() {
  // Inicjalizacja portu szeregowego
  Serial.begin(115200);
  delay(100); // Krótkie opóźnienie na potrzeby stabilizacji po uruchomieniu

  // Połącz z siecią WiFi
  connectToWiFi();

  // Inicjalizacja czujnika HX711
  scale.begin(HX711_dout_pin, HX711_sck_pin);

  // Wykonaj tarowanie
  tare();

  // Inicjalizacja serwera HTTP
  server.begin();

  Serial.println("ESP8266 uruchomiony.");
}

void loop() {
  // Akceptuj nowe połączenia
  WiFiClient client = server.available();
  if (client) {
    handleClient(client);
    delay(10); // Krótkie opóźnienie po obsłudze klienta
  }

  // Wykonaj odczyt wagi
  float weight = (scale.get_units() - tareValue) * scaleFactor;

  // Dodaj odczytaną wagę do tablicy próbek
  samples[sampleIndex] = weight;
  sampleIndex = (sampleIndex + 1) % numSamples;

  // Oblicz średnią wagę
  float sum = 0;
  int effectiveSamples = min(sampleIndex, numSamples);
  for (int i = 0; i < effectiveSamples; i++) {
    sum += samples[i];
  }
  float averageWeight = sum / effectiveSamples;

  // Oblicz odchyłkę aktualnego pomiaru od średniej wagi
  float deviation = weight - averageWeight;

  // Wyświetl wyniki na Serial Monitor
  Serial.print("Aktualna waga: ");
  Serial.print(weight);
  Serial.print(" kg | Średnia waga: ");
  Serial.print(averageWeight);
  Serial.print(" kg | Odchyłka: ");
  Serial.print(deviation);
  Serial.println(" kg");

  delay(1000);
}

void connectToWiFi() {
  Serial.println();
  Serial.print("Łączenie z siecią WiFi: ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    attempts++;
    if(attempts > 20) { // Przerwij próby łączenia się po 10 sekundach
      Serial.println("Nie udało się połączyć z siecią WiFi. Restartuję urządzenie.");
      delay(1000);
      ESP.restart(); // Restartuj urządzenie ESP8266
    }
  }

  Serial.println("");
  Serial.println("WiFi połączone");
  Serial.println("Adres IP: ");
  Serial.println(WiFi.localIP());
}

void handleClient(WiFiClient client) {
  // Zwiększenie licznika odwiedzin
  visits++;

  // Sprawdzenie, czy żądanie dotyczy przycisku tarowania lub odświeżania
  if (client.available()) {
    String request = client.readStringUntil('\r');
    if (request.indexOf("/tare") != -1) {
      tare();
    }
    if (request.indexOf("/refresh") != -1) {
      client.println("HTTP/1.1 301 Moved Permanently");
      client.println("Location: /");
      client.println("Connection: close");
      client.println();
      return;
    }
  }

  // Wysłanie nagłówka HTTP
  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: text/html; charset=UTF-8");
  client.println("Connection: close");  // Zamknięcie połączenia po przesłaniu danych
  client.println();
  
  // Wysłanie treści strony
  client.println("<!DOCTYPE HTML>");
  client.println("<html>");
  client.println("<head><title>ESP8266 Server</title></head>");
  client.println("<body style=\"font-family: Arial, sans-serif;\">"); // Ustawienie czcionki na Arial
  client.println("<h1>ESP8266 Server</h1>");
  client.print("<p>Liczba odwiedzin: ");
  client.print(visits);
  client.println("</p>");
  client.println("<form action=\"/tare\">");
  client.println("<button type=\"submit\">Taruj wagę</button>");
  client.println("</form>");
  client.println("<form action=\"/refresh\">");
  client.println("<button type=\"submit\">Odśwież</button>");
  client.println("</form>");
  client.print("<p>Aktualna waga: ");
  client.print((scale.get_units() - tareValue) * scaleFactor);
  client.println(" kg</p>");

  // Generuj wykres wagi
  generateWeightChart(client);

  client.println("</body>");
  client.println("</html>");

  // Zakończenie połączenia
  client.stop();
}

void generateWeightChart(WiFiClient client) {
  // Utworzenie diva dla wykresu
  client.println("<div style=\"width: 600px; position: relative; top: 10px; left: 50px;\">");
  client.println("<canvas id=\"weightChart\" width=\"600\" height=\"400\"></canvas>");
  client.println("<div style=\"display: flex; justify-content: center;\">"); // Rozpoczęcie diva dla opisów osi
  client.println("<div style=\"margin-right: 0px;\">"); // Rozpoczęcie diva dla opisu osi Y
  client.println("<div style=\"transform: rotate(-90deg); position: absolute; left: -50px; bottom: 200px;\">Waga (kg)</div>"); // Opis osi Y
  client.println("</div>"); // Zamknięcie diva dla osi Y
  client.println("<div style=\"position: absolute; bottom: 20; width: 100%; text-align: center;\">"); // Rozpoczęcie diva dla opisu osi X
  client.println("Czas (numer próbki)"); // Opis osi X
  client.println("</div>"); // Zamknięcie diva dla opisu osi X
  client.println("</div>"); // Zamknięcie diva dla opisów osi

  // Skrypt JavaScript do generowania wykresu
  client.println("<script src=\"https://cdnjs.cloudflare.com/ajax/libs/Chart.js/3.7.0/chart.min.js\"></script>");
  client.println("<script>");
  client.println("var ctx = document.getElementById('weightChart').getContext('2d');");
  client.println("var weightChart = new Chart(ctx, {");
  client.println("type: 'line',");
  client.println("data: {");
  client.println("labels: ["); // Dane osi X
  for (int i = 0; i < numSamples; i++) {
    client.print(i);
    if (i < numSamples - 1) {
      client.print(",");
    }
  }
  client.println("],");
  client.println("datasets: [{");
  client.println("label: 'Waga (kg)',");
  client.println("borderColor: 'rgb(75, 192, 192)',");
  client.println("fill: false,");
  client.println("data: ["); // Dane osi Y
  for (int i = 0; i < numSamples; i++) {
    client.print(samples[i]);
    if (i < numSamples - 1) {
      client.print(",");
    }
  }
  client.println("]");
  client.println("}]");
  client.println("},");
  client.println("options: {");
  client.println("scales: {");
  client.println("xAxes: [{");
  client.println("display: true,");
  client.println("scaleLabel: {");
  client.println("display: true,");
  client.println("labelString: 'Czas (numer próbki)'"); // Opis osi X
  client.println("},");
  client.println("ticks: {");
  client.println("autoSkip: true,");
  client.println("maxTicksLimit: 10"); // Ograniczenie liczby etykiet na osi X do 10
  client.println("}");
  client.println("}],");
  client.println("yAxes: [{");
  client.println("display: true,");
  client.println("scaleLabel: {");
  client.println("display: true,");
  client.println("labelString: 'Waga (kg)'"); // Opis osi Y
  client.println("}");
  client.println("}]");
  client.println("}");
  client.println("}");
  client.println("});");
  client.println("</script>");
}

void tare() {
  scale.tare();
  tareValue = scale.get_units();
  Serial.print("Wartość tarowania ustawiona na: ");
  Serial.println(tareValue);
}
