# Practica2: Interrupciones
## Objetivo
El objetivo es comprender como funcionan las interrupciones. Para esto, controlaremos dos LEDS de forma periódica y una entrada, el uso de esta provocará un cambio de frecuencia en las oscilaciones, solo en un LED.
## Apartado A: Interrupcion por GPIO
En este apartado, configuraremos un botón para que cada vez que lo presionemos genere una interrupcion, incrementando el contador. 
Los materiales usados en este apartado son:
1.Protoboard
2.ESP32-S2
3.Boton
Nuestro circuito montado quedo:
![image](https://github.com/user-attachments/assets/ba77938f-dc02-4db4-87c9-d504d80a7943) 
## Codigo usado:
```
struct Button {
  const uint8_t PIN;
  uint32_t numberKeyPresses;
   bool pressed;
};


Button button1 = {18, 0, false};


void IRAM_ATTR isr() {
  button1.numberKeyPresses += 1;
  button1.pressed = true;
}


void setup() {
  Serial.begin(115200);
  pinMode(button1.PIN, INPUT_PULLUP);
  attachInterrupt(button1.PIN, isr, FALLING);


  // Configurar LEDs
  pinMode(LED1, OUTPUT);
  pinMode(LED2, OUTPUT);
}


void loop() {
  if (button1.pressed) {
    Serial.printf("Button 1 has been pressed %u times\n", button1.numberKeyPresses);
    button1.pressed = false;
   
    // Encender LEDs por 500ms
    digitalWrite(LED1, HIGH);
    digitalWrite(LED2, HIGH);
    delay(500);
    digitalWrite(LED1, LOW);
    digitalWrite(LED2, LOW);
  }


  // Detach Interrupt after 1 Minute
  static uint32_t lastMillis = 0;
  if (millis() - lastMillis > 60000) {
    lastMillis = millis();
    detachInterrupt(button1.PIN);
    Serial.println("Interrupt Detached!");
  }
}


```
### Funcionamiento del código:
1. Se configura un botón en un pin GPIO con `INPUT_PULLUP`.
2. Se adjunta una interrupción en modo `FALLING`.
3. En la ISR, se incrementa un contador y se cambia un flag.
4. En `loop()`, se verifica el flag y se imprime la cantidad de veces que el botón fue presionado.
5. Después de un minuto, la interrupción se desactiva.

### Salidas:
En el puerto serie se muestra este mensaje:
```
Button 1 has been pressed X times
Interrupt Detached!
```
---
![Imagen de WhatsApp 2025-04-28 a las 12 20 00_8edc2969](https://github.com/user-attachments/assets/a08899fa-fba1-4ce6-947f-6404e801790b)



## Apartado B: Interrupcion por Timer
En este apartado, es lo mismo que el anterior pero en vez de GPIO, con un timer.
Los materiales usados son los mismos que en anterior apartado también.
El circuito montado también es el mismo
## Código usado:
```
volatile int interruptCounter = 0;
int totalInterruptCounter = 0;
bool ledState = false;  // Estado de los LEDs


const int LED1 = 16;
const int LED2 = 5;


hw_timer_t *timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;


void IRAM_ATTR onTimer() {
  portENTER_CRITICAL_ISR(&timerMux);
  interruptCounter++;
  portEXIT_CRITICAL_ISR(&timerMux);
}


void setup() {
  Serial.begin(115200);
  while (!Serial);  // Espera conexión serial


  pinMode(LED1, OUTPUT);
  pinMode(LED2, OUTPUT);


  // Configuración del temporizador
  timer = timerBegin(0, 80, true);       // Timer 0, divisor 80 -> 1 tick = 1us
  timerAttachInterrupt(timer, &onTimer, true);
  timerAlarmWrite(timer, 1000000, true); // Interrupción cada 1s (1,000,000 us)
  timerAlarmEnable(timer);


  Serial.println("Timer iniciado...");
}


void loop() {
  if (interruptCounter > 0) {
    portENTER_CRITICAL(&timerMux);
    interruptCounter--;
    portEXIT_CRITICAL(&timerMux);


    totalInterruptCounter++;


    Serial.print("An interrupt has occurred. Total number: ");
    Serial.println(totalInterruptCounter);


    // Alternar estado de los LEDs
    ledState = !ledState;
    digitalWrite(LED1, ledState);
    digitalWrite(LED2, ledState);
  }
}
```

###Funcionamiento del código:
1. Se configura un temporizador con `timerBegin()`.
2. Se adjunta una interrupción con `timerAttachInterrupt()`.
3. Se establece una alarma con `timerAlarmWrite()`.
4. Se habilita la alarma con `timerAlarmEnable()`.
5. En cada interrupción, se incrementa un contador.
6. En `loop()`, si hay una interrupción, se muestra el total de interrupciones en el puerto serie.

### Salidas Obtenidas
El puerto serie imprimirá:
```
An interrupt has occurred. Total number: X
```
---
![image](https://github.com/user-attachments/assets/8044fc35-ebed-41ea-a701-56118b4fe148)  


## Ejercicio para subir nota
En este ultimo apartado se nos pide que a partir de dos pulsadores que controlen un led alterar la frecuencia de parpadeo. Por lo tanto el led parpadeara a partir de la frecuencia inicial,
tendra interrupciones a partir del timer y cuando pulses un boton esta disminuira y si pulsas el otro aumentara. El código incluye un filtrado de rebotes para evitar lecturas erroneas. Fotos del montaje:
![image](https://github.com/user-attachments/assets/f77f834e-cfd7-4a2c-8f2d-60f8a80b97ec)
![image](https://github.com/user-attachments/assets/44f0f516-8f1f-4ed2-a2d6-4da587bae786)


## Código usado:
```
#include <Arduino.h>

#define LED_PIN 4 // Pin del LED
#define BTN_UP 18 // Botón para aumentar frecuencia
#define BTN_DOWN 12 // Botón para disminuir frecuencia

hw_timer_t *timer = NULL;
portMUX_TYPE timerMux = portMUX_INITIALIZER_UNLOCKED;

volatile int timerInterval = 500000; // Intervalo inicial (500ms)
volatile bool ledState = false;

// Variables para el filtrado de rebotes
volatile unsigned long lastPressUp = 0;
volatile unsigned long lastPressDown = 0;
const int debounceTime = 200; // Tiempo de filtrado de rebotes en ms

// Interrupción del Timer
void IRAM_ATTR onTimer() {
portENTER_CRITICAL_ISR(&timerMux);
ledState = !ledState;
digitalWrite(LED_PIN, ledState);
timerAlarmWrite(timer, timerInterval, true);
portEXIT_CRITICAL_ISR(&timerMux);
}

// ISR para el botón de aumentar frecuencia
void IRAM_ATTR isrButtonUp() {
unsigned long currentMillis = millis();
if (currentMillis - lastPressUp > debounceTime) {
portENTER_CRITICAL_ISR(&timerMux);
if (timerInterval > 100000) timerInterval -= 100000; // Reduce 100ms
portEXIT_CRITICAL_ISR(&timerMux);
lastPressUp = currentMillis;
}
}

// ISR para el botón de disminuir frecuencia
void IRAM_ATTR isrButtonDown() {
unsigned long currentMillis = millis();
if (currentMillis - lastPressDown > debounceTime) {
portENTER_CRITICAL_ISR(&timerMux);
if (timerInterval < 1000000) timerInterval += 100000; // Aumenta 100ms
portEXIT_CRITICAL_ISR(&timerMux);
lastPressDown = currentMillis;
}
}

void setup() {
Serial.begin(115200);
pinMode(LED_PIN, OUTPUT);
pinMode(BTN_UP, INPUT_PULLUP);
pinMode(BTN_DOWN, INPUT_PULLUP);

// Configurar interrupciones para los botones
attachInterrupt(BTN_UP, isrButtonUp, FALLING);
attachInterrupt(BTN_DOWN, isrButtonDown, FALLING);

// Configurar el temporizador
timer = timerBegin(0, 80, true); // Temporizador 0, divisor de 80 (1us por tick)
timerAttachInterrupt(timer, &onTimer, true);
timerAlarmWrite(timer, timerInterval, true);
timerAlarmEnable(timer);
}

void loop() {
// No es necesario código en loop(), todo funciona con interrupciones
}
```

### Funcionamiento del código
El botón que aumenta la frecuencia es el 18 y el que disminuye esta es el 12. El led lo tenemos en el pin 4.El led parpadea a 1Hz (500ms) de frecuencia inicial y sale por pantalla el mensaje de que se ha iniciado el sistema. Entonces pulsamos el botón del GPIO18 disminuye de 50 en 50 el tiempo entre parpadeos causando que la frecuencia sea mayor con un mínimo de 100ms, y cuando se presione el GPIO12 habra la misma proporcion de disminucion que de aumento en el GPIO18 solo que en el caso de la bajada el parpadeo tendra el limite en 2000ms.



