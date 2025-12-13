# LCD Display States

## Display Messages

### State 1: IDLE

┌────────────────────┐
│   SYSTEM READY     │
│   Scan QR Here     │
└────────────────────┘
**Trigger**: System startup, after dispensing complete  
**Duration**: Indefinite until QR detected

### State 2: VALIDATING

┌────────────────────┐
│   Validating...    │
│  Please wait       │
└────────────────────┘

**Trigger**: QR code detected by GM65  
**Duration**: 0.5-2 seconds

### State 3: DISPENSING

┌────────────────────┐
│   Dispensing...    │
│     5 seconds      │
└────────────────────┘
**Trigger**: QR code validation is true
**Duration**: 5 seconds

### State 4: ERROR

┌────────────────────┐
│      Error!        │
│  Invalid/Expired   │
└────────────────────┘
**Trigger**: QR code validation is false
**Duration**: 3 seconds until back to idle


## Code Implementation
```cpp
void displayIdle() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("  SYSTEM READY");
  lcd.setCursor(0, 1);
  lcd.print("  Scan QR Here");
}

void displayValidating() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("   VALIDATING...");
}

void displayDispensing() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("  DISPENSING...");
  lcd.setCursor(0, 1);
  lcd.print("   Please Wait");
}

void displayError() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("    ERROR ❌");
  lcd.setCursor(0, 1);
  lcd.print("Check system/logs");
}
```