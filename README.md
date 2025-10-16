#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <UniversalTelegramBot.h>
#include <ArduinoJson.h>

// Wi-Fi credentials
#define WIFI_SSID "UTC_A2"
#define WIFI_PASSWORD ""

// Telegram Bot credentials
#define BOT_TOKEN "7488731268:AAENJkBFZrZKWBmuqvloSzRAoP7Kxqrq_Ok"
#define CHAT_ID "1745652217"

// Pin definitions
#define LED_XANH 18
#define LED_DO 19
#define COI 5
#define CAM_BIEN_LUA 23
#define CAM_BIEN_GAS 22

WiFiClientSecure client;
UniversalTelegramBot bot(BOT_TOKEN, client);

unsigned long lastUpdate = 0;
const long interval = 1000;
unsigned long lastTelegramCheck = 0;
bool coiState = false;
bool lastAlertState = false;
bool wifiConnected = false;

void connectToWiFi() {
  Serial.print("Connecting to Wi-Fi...");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  
  int wifiTimeout = 0;
  while (WiFi.status() != WL_CONNECTED && wifiTimeout < 30) {
    delay(500);
    Serial.print(".");
    wifiTimeout++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    wifiConnected = true;
    Serial.println("\nConnected to Wi-Fi");
    client.setCACert(TELEGRAM_CERTIFICATE_ROOT);
    bot.sendMessage(CHAT_ID, " Hệ thống khởi động thành công!", "");
  } else {
    wifiConnected = false;
    Serial.println("\nFailed to connect to Wi-Fi");
  }
}

void handleNewMessages(int numNewMessages) {
  for (int i = 0; i < numNewMessages; i++) {
    if (bot.messages[i].chat_id != CHAT_ID) {
      bot.sendMessage(bot.messages[i].chat_id, "⚠️ Bạn không có quyền truy cập!", "");
      continue;
    }

    String text = bot.messages[i].text;
    if (text == "/status") {
      String message = " Trạng thái hệ thống:\n";
      message += "• Lửa: " + String(digitalRead(CAM_BIEN_LUA) == LOW ? "PHÁT HIỆN " : "Bình thường ") + "\n";
      message += "• Khí gas: " + String(digitalRead(CAM_BIEN_GAS) == LOW ? "PHÁT HIỆN " : "Bình thường ");
      bot.sendMessage(CHAT_ID, message, "");
    } 
    else if (text == "/help" || text == "/start") {
      String helpMsg = "🛟 Hướng dẫn sử dụng:\n";
      helpMsg += "/status - Kiểm tra trạng thái\n";
      helpMsg += "/help - Hiển thị hướng dẫn";
      bot.sendMessage(CHAT_ID, helpMsg, "");
    }
  }
}

void setup() {
  pinMode(LED_XANH, OUTPUT);
  pinMode(LED_DO, OUTPUT);
  pinMode(COI, OUTPUT);
  pinMode(CAM_BIEN_LUA, INPUT_PULLUP);
  pinMode(CAM_BIEN_GAS, INPUT_PULLUP);

  Serial.begin(115200);
  
  // Test hardware
  digitalWrite(LED_XANH, HIGH);
  digitalWrite(LED_DO, HIGH);
  digitalWrite(COI, HIGH);
  delay(500);
  digitalWrite(LED_DO, LOW);
  digitalWrite(COI, LOW);
  
  connectToWiFi();
  digitalWrite(LED_XANH, HIGH);
}

void loop() {
  bool flameDetected = digitalRead(CAM_BIEN_LUA) == LOW;
  bool gasDetected = digitalRead(CAM_BIEN_GAS) == LOW;
  unsigned long currentMillis = millis();

  if (WiFi.status() != WL_CONNECTED) {
    if (wifiConnected) {
      wifiConnected = false;
      Serial.println("Mất kết nối WiFi!");
    }
    digitalWrite(LED_XANH, LOW);
    digitalWrite(LED_DO, !digitalRead(LED_DO));
    delay(500);
  } 
  else if (!wifiConnected) {
    connectToWiFi();
  }

  if (flameDetected || gasDetected) {
    digitalWrite(LED_XANH, LOW);
    digitalWrite(LED_DO, HIGH);

    if (currentMillis - lastUpdate >= interval) {
      coiState = !coiState;
      digitalWrite(COI, coiState);
      lastUpdate = currentMillis;
    }

    if (!lastAlertState && wifiConnected) {
      String alertMsg = " CẢNH BÁO!\n";
      if (flameDetected) alertMsg += " PHÁT HIỆN LỬA!\n";
      if (gasDetected) alertMsg += " PHÁT HIỆN KHÍ GAS!";
      
      bot.sendMessage(CHAT_ID, alertMsg, "");
      lastAlertState = true;
    }
  } 
  else {
    digitalWrite(LED_DO, LOW);
    digitalWrite(COI, LOW);
    digitalWrite(LED_XANH, HIGH);

    if (lastAlertState && wifiConnected) {
      bot.sendMessage(CHAT_ID, " Hệ thống bình thường", "");
      lastAlertState = false;
    }
  }

  if (wifiConnected && currentMillis - lastTelegramCheck >= 10000) {
    int numNewMessages = bot.getUpdates(bot.last_message_received + 1);
    if (numNewMessages) handleNewMessages(numNewMessages);
    lastTelegramCheck = currentMillis;
  }
}
