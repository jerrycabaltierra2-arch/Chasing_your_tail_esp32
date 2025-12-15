/*
  chasing_your_tail.ino
  ESP32 proof-of-concept: Wi‑Fi promiscuous sniffer + BLE scanner
  Device tracker, TFT UI, encoder menu, NeoPixel and speaker alerts

  Notes:
  - Designed for ESP32 (Arduino core).
  - Adjust pin numbers to your wiring.
  - Threshold for suspicious devices is 10 detections.
  - This is a compact, single-file prototype for iteration.
*/

#include <WiFi.h>
#include <esp_wifi.h>
#include <TFT_eSPI.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <string>
#include <cstdio>
#include <Adafruit_NeoPixel.h>

// -------- Configuration (adjust pins) --------
#define PIN_NEOPIXEL 13
#define NUM_PIXELS 8
#define PIN_SPEAKER 25
#define PIN_ENCODER_A 32
#define PIN_ENCODER_B 33
#define PIN_ENCODER_BTN 27

// Wi‑Fi channel range
int wifiChannel = 1;

// Threshold
const int SUSPICIOUS_THRESHOLD = 10;

// TFT
TFT_eSPI tft = TFT_eSPI(); // TFT display instance

// NeoPixel
Adafruit_NeoPixel pixels(NUM_PIXELS, PIN_NEOPIXEL, NEO_GRB + NEO_KHZ800);

// Device types
enum DeviceType : uint8_t { DEVICE_WIFI = 1, DEVICE_BLE = 2 };

// Fixed-size hash table tracker for memory/speed
constexpr int TABLE_SIZE = 128; // power of two for masking (smaller footprint)
struct Entry {
  uint64_t key;       // 48-bit MAC packed into low bytes
  uint8_t type;       // bitfield
  uint16_t count;
  uint32_t lastSeen;  // millis() of last sighting
  bool used;
  bool suspicious;
};
Entry table[TABLE_SIZE];
constexpr int TABLE_MASK = TABLE_SIZE - 1;

// Helpers to pack/unpack MAC
static inline uint64_t macBytesToKey(const uint8_t mac[6]) {
  uint64_t k = 0;
  for (int i = 0; i < 6; ++i) k = (k << 8) | mac[i];
  return k;
}

static inline void keyToMacStr(uint64_t key, char *buf) {
  uint8_t mac[6];
  for (int i = 5; i >= 0; --i) { mac[i] = key & 0xFF; key >>= 8; }
  sprintf(buf, "%02X:%02X:%02X:%02X:%02X:%02X",
          mac[0], mac[1], mac[2], mac[3], mac[4], mac[5]);
}

// Find slot by key (returns index). If table is full, select a victim to evict.
static int findSlot(uint64_t key) {
  uint32_t h = (uint32_t)(key ^ (key >> 32));
  int idx = h & TABLE_MASK;
  for (int i = 0; i < TABLE_SIZE; ++i) {
    int j = (idx + i) & TABLE_MASK;
    if (!table[j].used) return j;        // empty slot
    if (table[j].key == key) return j;   // existing
  }
  // table full: find victim with smallest count, tie-breaker = oldest lastSeen
  int victim = 0;
  uint16_t minCount = table[0].count;
  uint32_t oldest = table[0].lastSeen;
  for (int i = 1; i < TABLE_SIZE; ++i) {
    if (table[i].count < minCount || (table[i].count == minCount && table[i].lastSeen < oldest)) {
      victim = i;
      minCount = table[i].count;
      oldest = table[i].lastSeen;
    }
  }
  // evict victim (will be overwritten by caller) — log eviction
  {
    char macbuf[32];
    keyToMacStr(table[victim].key, macbuf);
    char dbg[80];
    snprintf(dbg, sizeof(dbg), "Evicting %s count=%u lastSeen=%lu slot=%d", macbuf, (unsigned)table[victim].count, (unsigned long)table[victim].lastSeen, victim);
    Serial.println(dbg);
  }
  table[victim].used = false;
  return victim;
}
// UI state
enum Screen { SCREEN_SUMMARY, SCREEN_LIST, SCREEN_ALERTS, SCREEN_SETTINGS };
Screen currentScreen = SCREEN_SUMMARY;
int menuIndex = 0;

// Encoder state (simple polling)
int lastMenuIndex = 0;
unsigned long lastButtonPress = 0;

// BLE scan
BLEScan* pBLEScan;

// Add or update device in tracker from raw MAC bytes
void addOrUpdateDeviceBytes(const uint8_t mac[6], uint8_t type)
{
  uint64_t key = macBytesToKey(mac);
  int slot = findSlot(key);
  if (slot < 0) return; // table full
  uint32_t now = millis();
  if (!table[slot].used) {
    table[slot].used = true;
    table[slot].key = key;
    table[slot].type = type;
    table[slot].count = 1;
    table[slot].suspicious = (table[slot].count >= SUSPICIOUS_THRESHOLD);
    table[slot].lastSeen = now;
    // lightweight serial log for new device
    {
      char macbuf[32];
      keyToMacStr(key, macbuf);
      char dbg[64];
      snprintf(dbg, sizeof(dbg), "New device %s type=%s", macbuf, (type==DEVICE_WIFI?"WiFi":"BLE"));
      Serial.println(dbg);
    }
    return;
  }
  // update existing
  table[slot].count++;
  table[slot].type |= type;
  table[slot].lastSeen = now;
  if (!table[slot].suspicious && table[slot].count >= SUSPICIOUS_THRESHOLD) {
    table[slot].suspicious = true;
    ledcWriteTone(0, 2000);
    delay(120);
    ledcWriteTone(0, 0);
  }
}

// Clear logs
void clearLogs()
{
  for (int i = 0; i < TABLE_SIZE; ++i) table[i] = {0,0,0,0,false,false};
}

// Wi‑Fi promiscuous callback
void wifiSnifferCallback(void* buf, wifi_promiscuous_pkt_type_t type)
{
  if (type != WIFI_PKT_MGMT) return; // focus on management frames (probe reqs)
  wifi_promiscuous_pkt_t *pkt = (wifi_promiscuous_pkt_t*)buf;
  const uint8_t *payload = pkt->payload;
  // need at least 16 bytes (to read addr2)
  if (pkt->rx_ctrl.sig_len < 16) return;
  // addr2 starts at offset 10 in 802.11 header
  const uint8_t *mac = payload + 10;
  addOrUpdateDeviceBytes(mac, DEVICE_WIFI);
}

// BLE advertised device callback
class MyAdvertisedDeviceCallbacks: public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice advertisedDevice) {
    // parse "AA:BB:CC:DD:EE:FF" string
    std::string s = advertisedDevice.getAddress().toString();
    uint8_t mac[6] = {0};
    int vals[6];
    if (sscanf(s.c_str(), "%x:%x:%x:%x:%x:%x",
               &vals[0], &vals[1], &vals[2], &vals[3], &vals[4], &vals[5]) == 6) {
      for (int i = 0; i < 6; ++i) mac[i] = (uint8_t)vals[i];
      addOrUpdateDeviceBytes(mac, DEVICE_BLE);
    }
  }
};

// UI helpers
void drawSummary()
{
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE);
  tft.setTextSize(2);
  tft.setCursor(4, 8);
  int total = 0;
  int suspicious = 0;
  for (int i = 0; i < TABLE_SIZE; ++i) {
    if (!table[i].used) continue;
    total++;
    if (table[i].suspicious) suspicious++;
  }
  tft.printf("Total: %d", total);
  tft.setCursor(4, 36);
  tft.printf("Suspicious: %d", suspicious);
  tft.setCursor(4, 64);
  tft.printf("Channel: %d", wifiChannel);
}

void drawList()
{
  tft.fillScreen(TFT_BLACK);
  tft.setTextSize(1);
  int y = 0;
  char buf[32];
  for (int i = 0; i < TABLE_SIZE && y < tft.height()-10; ++i) {
    if (!table[i].used) continue;
    if (table[i].suspicious) tft.setTextColor(TFT_RED);
    else tft.setTextColor(TFT_GREEN);
    keyToMacStr(table[i].key, buf);
    tft.setCursor(2, y);
    tft.printf("%s (%d)", buf, table[i].count);
    y += 12;
  }
}

void drawAlerts()
{
  tft.fillScreen(TFT_BLACK);
  tft.setTextSize(1);
  int y = 0;
  char buf[32];
  for (int i = 0; i < TABLE_SIZE; ++i) {
    if (!table[i].used || !table[i].suspicious) continue;
    tft.setTextColor(TFT_RED);
    keyToMacStr(table[i].key, buf);
    tft.setCursor(2, y);
    tft.printf("%s (%d)", buf, table[i].count);
    y += 12;
    if (y >= tft.height()-12) break;
  }
}

void drawSettings()
{
  tft.fillScreen(TFT_BLACK);
  tft.setTextColor(TFT_WHITE);
  tft.setTextSize(1);
  tft.setCursor(2, 2);
  tft.println("Settings:");
  tft.printf("1 Clear logs\n");
  tft.printf("2 Channel: %d\n", wifiChannel);
}

// Update NeoPixel and speaker state based on suspicious devices
void updateAlertsVisuals()
{
  bool anySuspicious = false;
  for (int i = 0; i < TABLE_SIZE; ++i) if (table[i].used && table[i].suspicious) { anySuspicious = true; break; }
  if (anySuspicious) {
    // red
    for (int i=0;i<NUM_PIXELS;i++) pixels.setPixelColor(i, pixels.Color(255,0,0));
  } else {
    for (int i=0;i<NUM_PIXELS;i++) pixels.setPixelColor(i, pixels.Color(0,40,0));
  }
  pixels.show();
}

// Encoder poll (simple)
void pollEncoder()
{
  int a = digitalRead(PIN_ENCODER_A);
  int b = digitalRead(PIN_ENCODER_B);
  static int lastA = LOW;
  if (a != lastA) {
    if (a == HIGH) {
      if (b == LOW) menuIndex--; else menuIndex++;
      if (menuIndex < 0) menuIndex = 3;
      if (menuIndex > 3) menuIndex = 0;
    }
    lastA = a;
    delay(2);
  }
  if (digitalRead(PIN_ENCODER_BTN) == LOW) {
    if (millis() - lastButtonPress > 250) {
      lastButtonPress = millis();
      // confirm: set screen
      currentScreen = (Screen)menuIndex;
    }
  }
}

// Change Wi‑Fi channel (1..13)
void setWifiChannel(int ch)
{
  if (ch < 1) ch = 1;
  if (ch > 13) ch = 13;
  wifiChannel = ch;
  esp_wifi_set_channel(wifiChannel, WIFI_SECOND_CHAN_NONE);
}

void setup()
{
  Serial.begin(115200);
  delay(500);
  // TFT init
  tft.init();
  tft.setRotation(1);
  tft.fillScreen(TFT_BLACK);

  // NeoPixel
  pixels.begin();
  pixels.show();

  // speaker using LEDC channel 0
  ledcSetup(0, 2000, 8);
  ledcAttachPin(PIN_SPEAKER, 0);

  // encoder pins
  pinMode(PIN_ENCODER_A, INPUT_PULLUP);
  pinMode(PIN_ENCODER_B, INPUT_PULLUP);
  pinMode(PIN_ENCODER_BTN, INPUT_PULLUP);

   // Wi‑Fi init for promiscuous sniffing
   WiFi.mode(WIFI_MODE_STA);
   WiFi.disconnect(true);
   esp_wifi_set_promiscuous_rx_cb(&wifiSnifferCallback);
   esp_wifi_set_promiscuous(true);
   setWifiChannel(wifiChannel);

  // BLE init
  BLEDevice::init("");
  pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new MyAdvertisedDeviceCallbacks());
  pBLEScan->setActiveScan(true);
  pBLEScan->setInterval(100);
  pBLEScan->setWindow(99);
  // initialize tracker and UI
  clearLogs();
  drawSummary();
}

unsigned long lastBLEScan = 0;
const unsigned long BLE_SCAN_INTERVAL_MS = 10000; // every 10s

void loop()
{
  pollEncoder();

  // periodic BLE scan (non-blocking pattern)
  if (millis() - lastBLEScan > BLE_SCAN_INTERVAL_MS) {
    lastBLEScan = millis();
    // run short scan in blocking mode but short
    pBLEScan->start(3, false);
    pBLEScan->clearResults();
  }

  // UI update when screen changes
  if (lastMenuIndex != menuIndex) {
    lastMenuIndex = menuIndex;
    // highlight menu selection briefly
  }

  // draw current screen
  switch (currentScreen) {
    case SCREEN_SUMMARY: drawSummary(); break;
    case SCREEN_LIST: drawList(); break;
    case SCREEN_ALERTS: drawAlerts(); break;
    case SCREEN_SETTINGS: drawSettings(); break;
  }

  updateAlertsVisuals();

  // check settings actions if on settings screen and button pressed
  if (currentScreen == SCREEN_SETTINGS && digitalRead(PIN_ENCODER_BTN) == LOW) {
    delay(50);
    // short press cycles actions: first press = clear, second = change channel
    static int settingsStep = 0;
    if (millis() - lastButtonPress > 300) {
      lastButtonPress = millis();
      if (settingsStep == 0) {
        clearLogs();
      } else if (settingsStep == 1) {
        setWifiChannel(wifiChannel % 13 + 1);
      }
      settingsStep = (settingsStep + 1) % 2;
    }
  }

  delay(200);
}

