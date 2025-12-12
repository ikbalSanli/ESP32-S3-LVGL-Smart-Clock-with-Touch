#include <TFT_eSPI.h>
#include <lvgl.h>
#include <Wire.h>
#include <WiFi.h>
#include <time.h>

#define TOUCH_SDA 6
#define TOUCH_SCL 7
#define TOUCH_INT 21
#define TOUCH_RST 13
#define I2C_FREQ 400000
#define CST816S_ADDRESS 0x15

TFT_eSPI tft = TFT_eSPI();
static lv_color_t buf[240 * 20];
static lv_disp_draw_buf_t draw_buf;

int16_t touch_x = 0;
int16_t touch_y = 0;
bool touch_pressed = false;

typedef struct _objects_t {
  lv_obj_t *main;
} objects_t;

enum ScreensEnum {
  SCREEN_ID_MAIN = 1,
};

objects_t objects;
static int16_t currentScreen = -1;

lv_obj_t *wifi_icon = nullptr;
lv_obj_t *label_time = nullptr;
lv_obj_t *arc_sec = nullptr;
bool wifiConnected = false;
unsigned long lastNTPSync = 0;

#define MANUAL_YEAR 2025
#define MANUAL_MONTH 12
#define MANUAL_DAY 12
#define MANUAL_HOUR 16
#define MANUAL_MIN 10
#define DISABLE_NTP false

void create_screen_main();
void tick_screen_main();
void create_screens();
void loadScreen(int screenId);
void ui_init();
void ui_tick();

void i2c_scan() {
  byte count = 0;
  for (byte i = 1; i < 127; i++) {
    Wire.beginTransmission(i);
    if (Wire.endTransmission() == 0) {
      count++;
    }
  }
}

void touch_init() {
  Wire.begin(TOUCH_SDA, TOUCH_SCL, I2C_FREQ);
  delay(50);
  pinMode(TOUCH_RST, OUTPUT);
  digitalWrite(TOUCH_RST, LOW);
  delay(20);
  digitalWrite(TOUCH_RST, HIGH);
  delay(100);
  pinMode(TOUCH_INT, INPUT_PULLUP);
  i2c_scan();
}

bool touch_read() {
  Wire.beginTransmission(CST816S_ADDRESS);
  Wire.write(0x01);
  if (Wire.endTransmission(false) != 0) return false;
  if (Wire.requestFrom(CST816S_ADDRESS, 5) != 5) return false;

  uint8_t points = Wire.read();
  uint8_t xHigh = Wire.read();
  uint8_t xLow = Wire.read();
  uint8_t yHigh = Wire.read();
  uint8_t yLow = Wire.read();

  if (points == 1) {
    touch_x = ((xHigh & 0x0F) << 8) | xLow;
    touch_y = ((yHigh & 0x0F) << 8) | yLow;
    if (touch_x >= 240) touch_x = 239;
    if (touch_y >= 240) touch_y = 239;
    touch_pressed = true;
    return true;
  }
  touch_pressed = false;
  return false;
}

void my_touchpad_read(lv_indev_drv_t *indev_driver, lv_indev_data_t *data) {
  if (touch_read()) {
    data->state = LV_INDEV_STATE_PRESSED;
    data->point.x = touch_x;
    data->point.y = touch_y;
  } else {
    data->state = LV_INDEV_STATE_RELEASED;
  }
}

void my_disp_flush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p) {
  uint32_t w = area->x2 - area->x1 + 1;
  uint32_t h = area->y2 - area->y1 + 1;

  tft.startWrite();
  tft.setAddrWindow(area->x1, area->y1, w, h);
  tft.pushColors((uint16_t *)&color_p->full, w * h, true);
  tft.endWrite();

  lv_disp_flush_ready(disp);
}

void setup() {
  Serial.begin(115200);
  WiFi.useStaticBuffers(true);
  WiFi.mode(WIFI_OFF);
  delay(100);
  WiFi.mode(WIFI_STA);
  delay(100);
  WiFi.disconnect(true);
  delay(500);

  WiFi.begin("cvw8-gqead", "qs5Hak&paD2t");

  int retry = 0;
  while (WiFi.status() != WL_CONNECTED && retry < 30) {
    delay(500);
    retry++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    wifiConnected = true;
    configTime(3 * 3600, 0, "tr.pool.ntp.org", "pool.ntp.org");
  } else {
    struct tm timeinfo;
    timeinfo.tm_year = MANUAL_YEAR - 1900;
    timeinfo.tm_mon = MANUAL_MONTH - 1;
    timeinfo.tm_mday = MANUAL_DAY;
    timeinfo.tm_hour = MANUAL_HOUR;
    timeinfo.tm_min = MANUAL_MIN;
    timeinfo.tm_sec = 0;
    time_t t = mktime(&timeinfo);
    struct timeval tv = { .tv_sec = t };
    settimeofday(&tv, NULL);
  }

  pinMode(2, OUTPUT);
  digitalWrite(2, HIGH);
  tft.init();
  tft.setRotation(0);
  tft.fillScreen(TFT_BLACK);

  lv_init();
  lv_disp_draw_buf_init(&draw_buf, buf, NULL, 240 * 20);

  static lv_disp_drv_t disp_drv;
  lv_disp_drv_init(&disp_drv);
  disp_drv.hor_res = 240;
  disp_drv.ver_res = 240;
  disp_drv.flush_cb = my_disp_flush;
  disp_drv.draw_buf = &draw_buf;
  lv_disp_drv_register(&disp_drv);

  touch_init();

  static lv_indev_drv_t indev_drv;
  lv_indev_drv_init(&indev_drv);
  indev_drv.type = LV_INDEV_TYPE_POINTER;
  indev_drv.read_cb = my_touchpad_read;
  lv_indev_drv_register(&indev_drv);

  ui_init();
}

void loop() {
  static unsigned long last_update = 0;

  if (millis() - last_update >= 1000) {
    last_update = millis();

    time_t now;
    time(&now);
    struct tm *tm_info = localtime(&now);

    if (label_time) {
      char buffer[16];
      sprintf(buffer, "%02d:%02d:%02d",
              tm_info->tm_hour,
              tm_info->tm_min,
              tm_info->tm_sec);
      lv_label_set_text(label_time, buffer);
    }

    if (arc_sec) {
      int arc_value = (tm_info->tm_sec * 100) / 60;
      lv_arc_set_value(arc_sec, arc_value);
    }
  }

  lv_timer_handler();
  delay(5);
}

void create_screen_main() {
  lv_obj_t *scr = lv_obj_create(NULL);
  objects.main = scr;
  lv_obj_set_size(scr, 240, 240);
  lv_obj_set_style_bg_color(scr, lv_color_hex(0x000000), LV_PART_MAIN);

  wifi_icon = lv_label_create(scr);
  lv_obj_set_style_text_font(wifi_icon, &lv_font_montserrat_12, LV_PART_MAIN);
  lv_label_set_text(wifi_icon, wifiConnected ? LV_SYMBOL_WIFI : LV_SYMBOL_CLOSE);
  lv_obj_align(wifi_icon, LV_ALIGN_TOP_LEFT, 10, 5);

  label_time = lv_label_create(scr);
  lv_obj_set_style_text_color(label_time, lv_color_hex(0x00E5FF), LV_PART_MAIN);
  lv_obj_set_style_text_font(label_time, &lv_font_montserrat_16, LV_PART_MAIN);
  lv_obj_align(label_time, LV_ALIGN_CENTER, 0, -20);

  arc_sec = lv_arc_create(scr);
  lv_obj_set_size(arc_sec, 200, 200);
  lv_arc_set_rotation(arc_sec, 270);
  lv_arc_set_bg_angles(arc_sec, 0, 360);
  lv_arc_set_value(arc_sec, 0);
  lv_obj_set_style_arc_width(arc_sec, 10, LV_PART_INDICATOR);
  lv_obj_set_style_arc_color(arc_sec, lv_color_hex(0xFF2ECC), LV_PART_INDICATOR);
  lv_obj_clear_flag(arc_sec, LV_OBJ_FLAG_CLICKABLE);
  lv_obj_align(arc_sec, LV_ALIGN_CENTER, 0, 0);

  lv_scr_load(scr);
}

void tick_screen_main() {}

void create_screens() {
  lv_disp_t *dispp = lv_disp_get_default();
  lv_theme_t *theme = lv_theme_default_init(
      dispp,
      lv_palette_main(LV_PALETTE_BLUE),
      lv_palette_main(LV_PALETTE_RED),
      false,
      LV_FONT_DEFAULT);
  lv_disp_set_theme(dispp, theme);
  create_screen_main();
}

void loadScreen(int screenId) {
  currentScreen = screenId - 1;
  lv_scr_load(objects.main);
}

void ui_init() {
  create_screens();
  loadScreen(SCREEN_ID_MAIN);
}

void ui_tick() {}
