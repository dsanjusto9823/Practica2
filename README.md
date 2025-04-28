# Practica2: Interrupciones
## Objetivo
El objetivo es comprender como funcionan las interrupciones. Para esto, controlaremos dos LEDS de forma periódica y una entrada, el uso de esta provocará un cambio de frecuencia en las oscilaciones, solo en un LED.
## Apartado A: Interrupcion por GPIO
En este apartado, configuraremos un botón para que cada vez que lo presionemos genere una interrupcion, incrementando el contador. 
Los materiales usados en este apartado son:
-Protoboard
-ESP32-S2
-Boton
Nuestro circuito montado quedo:
![image](https://github.com/user-attachments/assets/ba77938f-dc02-4db4-87c9-d504d80a7943) 
## Codigo usado:
```
struct Button {
  const uint8_t PIN;
  volatile uint32_t numberKeyPresses;
  volatile bool pressed;
};


Button button1 = {16, 0, false};


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
![image](https://github.com/user-attachments/assets/8044fc35-ebed-41ea-a701-56118b4fe148)  

