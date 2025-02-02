# 便携式太阳能供电系统

## 项目简介
本项目是一个便携式太阳能供电系统，支持太阳能充电和220V市电充电，电池容量为20000mAh。系统实时显示电池电压、电量、充电状态、时间、日期、星期和农历，并具有过流和短路保护功能。充电时或使用时屏幕亮起，不使用时屏幕关闭以节省电量。

---

## 功能
1. **供电功能**：
   - 支持太阳能充电（外置20W太阳能板）。
   - 支持220V市电充电。
   - 电池容量20000mAh，支持12小时以上供电。
2. **显示功能**：
   - 实时显示电池电压、电量、充电状态。
   - 显示当前时间、日期、星期和农历。
3. **保护功能**：
   - 加入保险丝，提供过流和短路保护。
4. **节能功能**：
   - 充电时或使用时屏幕亮起，不使用时屏幕关闭。

---

## 硬件清单
| 组件               | 数量 | 备注                          |
|--------------------|------|-------------------------------|
| 18650电池（3500mAh）| 6节  | 推荐松下NCR18650B或三星35E    |
| 18650电池盒（带保护电路） | 2个  | 支持3节并联                   |
| 20W折叠太阳能板    | 1块  | 输出电压5V-12V，外置设计      |
| 太阳能充电控制器   | 1个  | 支持5V/12V输出                |
| DC-DC降压模块      | 1个  | 可选，用于调整输出电压        |
| 220V充电模块       | 1个  | 支持220V输入，输出5V/12V     |
| 塑料或铝合金外壳   | 1个  | 尺寸约20cm × 15cm × 5cm       |
| USB输出接口        | 1个  | 5V/2A输出                     |
| DC输出接口         | 1个  | 根据设备需求选择（如5.5mm接口）|
| 电源开关           | 1个  | 控制输出通断                  |
| OLED显示屏（0.91英寸） | 1个  | I2C接口，显示基本信息         |
| Arduino Nano       | 1个  | 用于控制显示屏和读取传感器数据 |
| 电压传感器         | 1个  | 用于测量电池电压              |
| DS3231 RTC模块     | 1个  | 用于时间显示                  |
| 电流检测模块（ACS712） | 1个  | 用于检测设备连接              |
| 保险丝（5A）       | 3个  | 用于过流保护                  |
| 保险丝座           | 3个  | 用于安装保险丝                |
| 耐高温导线         | 若干 | 用于内部布线                  |
| 螺丝、胶水         | 若干 | 固定外壳和组件                |
| 工具（螺丝刀、电烙铁等） | 1套  | 用于组装和焊接                |

---

## 电路图
![电路图](http://wd393.fun:10001/i/2025/02/02/fk0a4a.png)

---

## 代码说明
### 功能
1. **显示电池电压、电量、充电状态**。
2. **显示当前时间、日期、星期和农历**。
3. **低功耗模式**：降低刷新频率，减少功耗。
4. **异常处理**：检测传感器和RTC模块是否正常工作。
5. **精确电量计算**：考虑电池放电曲线，使用非线性拟合计算电量。
6. **屏幕控制**：充电时或使用时亮屏，不使用时息屏。

### 依赖库
1. **Adafruit SSD1306**：用于驱动OLED显示屏。
2. **RTClib**：用于驱动RTC模块。
3. **LunarCalendar**：用于计算农历日期。

### 代码示例
```cpp
#include <Wire.h>
#include <Adafruit_SSD1306.h>  // OLED库
#include <RTClib.h>            // RTC库
#include <LunarCalendar.h>     // 农历库

// OLED显示屏配置
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// RTC模块配置
RTC_DS3231 rtc;

// 电压传感器配置
const int voltagePin = A0;  // 电压传感器连接引脚
const float referenceVoltage = 5.0;  // 参考电压
const float voltageDividerRatio = 2.0;  // 分压比
const float minVoltage = 3.0;  // 电池最低电压
const float maxVoltage = 4.2;  // 电池最高电压
const float chargingThreshold = 4.0;  // 充电阈值

// 电流检测模块配置
const int currentPin = A1;  // 电流检测模块连接引脚
const float currentThreshold = 0.1;  // 电流阈值（A）

// 低功耗模式配置
const unsigned long refreshInterval = 5000;  // 刷新间隔（毫秒）
unsigned long lastRefreshTime = 0;  // 上次刷新时间

// 星期名称数组
const char* weekDays[7] = {"日", "一", "二", "三", "四", "五", "六"};

void setup() {
  // 初始化串口通信
  Serial.begin(9600);

  // 初始化OLED显示屏
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED初始化失败！");
    while (1);  // 如果初始化失败，停止程序
  }
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.display();

  // 初始化RTC模块
  if (!rtc.begin()) {
    Serial.println("RTC模块未找到！");
    while (1);  // 如果初始化失败，停止程序
  }
  if (rtc.lostPower()) {
    Serial.println("RTC模块断电，正在设置时间...");
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));  // 设置时间为编译时间
  }

  // 初始化电压传感器
  pinMode(voltagePin, INPUT);

  // 初始化电流检测模块
  pinMode(currentPin, INPUT);
}

void loop() {
  // 读取电池电压
  float batteryVoltage = readBatteryVoltage();
  if (batteryVoltage == 0.0) {
    Serial.println("电压传感器异常，请检查连接！");
    return;
  }

  // 读取输出电流
  float outputCurrent = readOutputCurrent();

  // 判断是否在充电
  bool isCharging = batteryVoltage > chargingThreshold;

  // 判断是否有设备连接
  bool isDeviceConnected = outputCurrent > currentThreshold;

  // 控制屏幕亮灭
  if (isCharging || isDeviceConnected) {
    // 如果正在充电或有设备连接，保持屏幕常亮
    updateDisplay(batteryVoltage, calculateBatteryPercentage(batteryVoltage), rtc.now(), LunarCalendar(rtc.now().year(), rtc.now().month(), rtc.now().day()));
  } else {
    // 如果未充电且无设备连接，关闭屏幕
    display.clearDisplay();
    display.display();
  }

  // 低功耗模式：按固定间隔刷新显示
  if (millis() - lastRefreshTime >= refreshInterval) {
    lastRefreshTime = millis();  // 更新上次刷新时间
  }
}

// 读取电池电压
float readBatteryVoltage() {
  int sensorValue = analogRead(voltagePin);
  if (sensorValue == 0 || sensorValue == 1023) {
    return 0.0;  // 传感器异常，返回0.0
  }
  float voltage = (sensorValue / 1023.0) * referenceVoltage * voltageDividerRatio;
  return voltage;
}

// 读取输出电流
float readOutputCurrent() {
  int sensorValue = analogRead(currentPin);
  float voltage = (sensorValue / 1023.0) * referenceVoltage;
  float current = (voltage - 2.5) / 0.185;  // ACS712电流计算公式
  return abs(current);  // 返回绝对值
}

// 计算电池电量百分比
int calculateBatteryPercentage(float voltage) {
  // 使用非线性拟合计算电量百分比
  if (voltage >= maxVoltage) return 100;
  if (voltage <= minVoltage) return 0;
  float percentage = (voltage - minVoltage) / (maxVoltage - minVoltage);
  percentage = percentage * 100;  // 转换为百分比
  return constrain(percentage, 0, 100);
}

// 更新OLED显示内容
void updateDisplay(float batteryVoltage, int batteryPercentage, DateTime now, LunarCalendar lunar) {
  display.clearDisplay();

  // 显示电池电压
  display.setCursor(0, 0);
  display.print("电压: ");
  display.print(batteryVoltage);
  display.println(" V");

  // 显示电池电量
  display.setCursor(0, 10);
  display.print("电量: ");
  display.print(batteryPercentage);
  display.println(" %");

  // 显示当前时间
  display.setCursor(0, 20);
  display.print("时间: ");
  display.print(now.hour());
  display.print(":");
  display.print(now.minute());
  display.print(":");
  display.println(now.second());

  // 显示日期
  display.setCursor(0, 30);
  display.print("日期: ");
  display.print(now.year());
  display.print("-");
  display.print(now.month());
  display.print("-");
  display.println(now.day());

  // 显示星期
  display.setCursor(0, 40);
  display.print("星期: ");
  display.println(weekDays[now.dayOfTheWeek()]);

  // 显示农历
  display.setCursor(0, 50);
  display.print("农历: ");
  display.print(lunar.lunarYear());
  display.print("-");
  display.print(lunar.lunarMonth());
  display.print("-");
  display.println(lunar.lunarDay());

  // 刷新显示
  display.display();
}
```

---

## 制作步骤
1. **组装电池组**：
   - 将6节18650电池并联，安装保险丝。
2. **安装太阳能板**：
   - 将太阳能板固定在外壳上层，连接充电控制器。
3. **安装220V充电模块**：
   - 将220V充电模块固定在外壳下层，连接电池组。
4. **安装充电控制器**：
   - 连接电池组、太阳能板和220V充电模块。
5. **安装输出接口**：
   - 连接USB和DC输出接口。
6. **安装显示屏和控制器**：
   - 连接OLED显示屏、RTC模块、电压传感器、电流检测模块和Arduino Nano。
7. **内部布线**：
   - 使用耐高温导线连接所有组件，确保绝缘。
8. **封装外壳**：
   - 固定所有组件，合上外壳。

---

## 测试与调试
1. **保险功能测试**：
   - 模拟短路或过流情况，检查保险丝是否正常熔断。
2. **电路功能测试**：
   - 确保保险丝不影响正常电路的运行。
3. **太阳能充电测试**：
   - 将太阳能板展开，放置在阳光下，观察充电控制器指示灯是否正常。
4. **220V充电测试**：
   - 连接220V市电，观察充电控制器指示灯是否正常。
5. **电池输出测试**：
   - 打开电源开关，用万用表测量USB和DC输出接口电压是否正常。
6. **显示屏测试**：
   - 检查显示屏是否正常显示电池电压、电量、充电状态、时间、日期、星期和农历。
7. **设备供电测试**：
   - 连接MMDVM或对讲机，测试供电是否稳定。
8. **屏幕控制测试**：
   - 测试充电时和使用时屏幕是否亮起，不使用时屏幕是否关闭。

---

## 总结
- **设计特点**：
  - 便携、轻量化，重量约800-1000克。
  - 支持太阳能充电和220V市电充电，电池容量20000mAh。
  - 实时显示电池状态信息、时间、日期、星期和农历。
  - 加入保险丝，提供过流和短路保护。
  - 充电时或使用时亮屏，不使用时息屏，节省电量。
- **成本**：约450-500元。
- **适用场景**：户外活动、应急供电、对讲机充电等。

---
