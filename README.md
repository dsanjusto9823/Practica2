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
