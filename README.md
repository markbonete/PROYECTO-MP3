# Proyecto: MP3
Este programa reproduce archivos .mp3 desde una tarjeta SD y muestra su información en una pantalla. Permite controlar la reproducción de música mediante tres botones: play, next y previous.

## Componentes físicos utilizados
- Microcontrolador ESP32
- Tarjeta SD
- Módulo de audio
- Pantalla OLED

## Librerías necesarias
- SD.h: Para el acceso a la tarjeta SD.
- Audio.h: Para la reproducción de audio.
- Adafruit_GFX.h: Para la manipulación de gráficos en la pantalla OLED (opcional).
- Adafruit_SSD1306.h: Para controlar la pantalla OLED (opcional).

## Funcionamiento básico
El programa busca los archivos MP3 en la carpeta "playlist" en la tarjeta SD al inicio. Los botones físicos conectados al Arduino (play, next y previous) permiten controlar la reproducción de las canciones. Cuando se presiona el botón de reproducción, el programa reproduce la siguiente canción en la lista. Los botones de siguiente y anterior permiten avanzar y retroceder en la lista de reproducción, respectivamente. Se utiliza una pantalla OLED opcional para mostrar información sobre la canción que se está reproduciendo.

## Explicación del código por partes
### Librerías Utilizadas
- Arduino Core (Arduino.h): La base para la mayoría de los proyectos de Arduino.
- SD (SD.h): Para la gestión de la tarjeta SD.
- FS (FS.h): Sistema de archivos para gestionar archivos en la tarjeta SD.
- Audio (Audio.h, AudioGeneratorMP3.h, AudioOutputI2S.h, AudioFileSourceSD.h): Conjunto de librerías para reproducir audio en formato MP3 utilizando el protocolo I2S.
- Wire (Wire.h): Para la comunicación I2C.
- Adafruit GFX (Adafruit_GFX.h): Librería para gráficos en pantallas.
- Adafruit SSD1306 (Adafruit_SSD1306.h): Librería específica para manejar pantallas OLED basadas en el controlador SSD1306.
- I2S (driver/i2s.h): Controlador I2S para salida de audio.
- Vector (vector): Librería de C++ para manejo de vectores dinámicos.

### Definiciones de Pines
Estas líneas definen los pines del microcontrolador utilizados para la tarjeta SD, el bus SPI, la salida de audio I2S, la comunicación I2C y los botones de control.
```cpp
#define SD_CS         15
#define SPI_MOSI      12
#define SPI_MISO      13
#define SPI_SCK       14
#define I2S_DOUT      25
#define I2S_BCLK      27
#define I2S_LRC       26
#define I2C_SDA       21
#define I2C_SCL       22
#define BUTTON_PLAY   4
#define BUTTON_PREV   16
#define BUTTON_NEXT   2
```

### Declaraciones de Objetos y Variables Globales
#### Pantalla OLED:
```cpp
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32
#define OLED_RESET    -1
#define SCREEN_ADDRESS 0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
```
#### Audio:
```cpp
AudioFileSourceSD *audioFile;
AudioGeneratorMP3 *mp3;
AudioOutputI2S *audioOutput;
```
#### Lista de Reproducción:
```cpp
const int Nfiles=15;
String playlist[Nfiles];
std::vector<String> filenames;
int fileCount = 0;
int currentIndex = 0;
bool isPlaying = false;
```
#### Botones:
```cpp
bool prevButtonStateRaw = false;
bool playButtonStateRaw = false;
bool nextButtonStateRaw = false;
unsigned long lastDebounceTimePrev = 0;
unsigned long lastDebounceTimePlay = 0;
unsigned long lastDebounceTimeNext = 0;
```

### Funciones
`setup()`
Configura el entorno inicial:
1. Inicialización de la Comunicación Serie:
```cpp
   Serial.begin(115200);
```
2. Inicialización de la Tarjeta SD:
```cpp
SPI.begin(SPI_SCK, SPI_MISO, SPI_MOSI, SD_CS);
if (!SD.begin(SD_CS)) {
  Serial.println("Error al inicializar la tarjeta SD.");
  return;
}
```
3. Inicialización de la Comunicación I2C:
```cpp
   Wire.begin(I2C_SDA, I2C_SCL);
```
4. Inicialización de la Pantalla OLED:
```
if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
  Serial.println(F("Error en la inicialización de la pantalla OLED."));
  for (;;);
}
```
5. Configuración de Pines para Audio:
```cpp
audioOutput = new AudioOutputI2S();
audioOutput->SetPinout(I2S_BCLK, I2S_LRC, I2S_DOUT);
audioOutput->SetGain(0.125);
```
6. Configuración de Pines de Botones:
```cpp
pinMode(BUTTON_PLAY, INPUT_PULLUP);
pinMode(BUTTON_PREV, INPUT_PULLUP);
pinMode(BUTTON_NEXT, INPUT_PULLUP);
```
7. Listar Archivos en la Tarjeta SD:
```cpp
listFiles();
```
8. Mostrar Selección Inicial en la Pantalla:
```cpp
displayCurrentSelection();
```

`loop()`
Verifica el estado de los botones y la reproducción del MP3:
1. Leer el Estado de los Botones:
```cpp
readButtons();
```
Verificar la Reproducción en Curso:
```cpp
if (mp3 && mp3->isRunning()) {
  if (!mp3->loop()) {
    playNext();
  }
}
```

`playMP3(const char *filename)`
Reproduce un archivo MP3 específico:
1. Detener Reproducción Actual:
```cpp
if (isPlaying && mp3) {
  mp3->stop();
  isPlaying = false;
}
```
2. Iniciar Nueva Reproducción:
```cpp
audioFile = new AudioFileSourceSD(filename);
mp3 = new AudioGeneratorMP3();
if (mp3->begin(audioFile, audioOutput)) {
  isPlaying = true;
  displaySongInfo(filename);
} else {
  delete mp3;
  delete audioFile;
  mp3 = nullptr;
  audioFile = nullptr;
}
```

`displaySongInfo(const char *filename)`
Muestra la información de la canción en la pantalla OLED.
1. Formatear Nombre del Archivo:
```cpp
String fileStr = String(filename);
fileStr.replace("/playlist/", "");
fileStr.replace(".mp3", "");
```
2. Extraer Nombre de Canción y Artista:
```cpp
   int separatorIndex = fileStr.indexOf('_');
String songName, artistName;
if (separatorIndex != -1) {
  songName = fileStr.substring(0, separatorIndex);
  artistName = fileStr.substring(separatorIndex + 1);
} else {
  songName = fileStr;
  artistName = "Desconocido";
}
```
3. Mostrar Información en la Pantalla:
```cpp
display.clearDisplay();
display.setTextSize(1);
display.setTextColor(SSD1306_WHITE);
display.setCursor(0, 10);
display.println(songName);
display.setCursor(0, 20);
display.println(artistName);
display.display();
```

`listFiles()`
Lista los archivos disponibles en la carpeta "playlist" de la tarjeta SD:
1. Abrir Directorio:
```cpp
File dir = SD.open("/playlist");
```
2. Leer Archivos y Almacenar Nombres:
```cpp
fileCount = 0;
while (true) {
  File entry = dir.openNextFile();
  if (!entry) break;
  filenames.push_back(String("/playlist/") + entry.name());
}
dir.close();
```
3. Ordenar Nombres de Archivos:
```cpp
std::sort(filenames.begin(), filenames.end());
```
4. Copiar Nombres al Array Playlist:
```cpp
fileCount = filenames.size();
for (int i = 0; i < fileCount; ++i) {
  playlist[i] = filenames[i];
}
```
`playNext()`
Reproduce la siguiente canción en la lista:
1. Reproduce la siguiente canción en la lista:
```cpp
if (fileCount > 0) {
  currentIndex = (currentIndex + 1) % fileCount;
```
2. Reproducir Canción:
```cpp
if (currentIndex < fileCount) {
  playMP3(playlist[currentIndex].c_str());
} else {
  currentIndex = 0;
  playMP3(playlist[currentIndex].c_str());
}
```
`readButtons()`
Lee el estado de los botones y maneja la lógica de debounce:
1. Leer Estado Actual de los Botones:
```cpp
prevButtonStateRaw = digitalRead(BUTTON_PREV);
playButtonStateRaw = digitalRead(BUTTON_PLAY);
nextButtonStateRaw = digitalRead(BUTTON_NEXT);
```
2. Manejar Debounce:
```cpp
unsigned long currentTime = millis();
if (prevButtonStateRaw != prevButtonLastState) {
  lastDebounceTimePrev = currentTime;
}
if ((currentTime - lastDebounceTimePrev) > DEBOUNCE_DELAY) {
  if (prevButtonStateRaw != prevButtonState) {
    prevButtonState = prevButtonStateRaw;
    if (prevButtonState == LOW) {
      currentIndex = (currentIndex - 1 + fileCount) % fileCount;
      if (isPlaying) {
        playMP3(playlist[currentIndex].c_str());
      } else {
        displayCurrentSelection();
      }
    }
  }
}
prevButtonLastState = prevButtonStateRaw;

// Similar logic for PLAY and NEXT buttons
```

`displayCurrentSelection()`
Muestra la canción actualmente seleccionada en la pantalla OLED:
1. Formatear Nombre del Archivo:
```cpp
String fileStr = playlist[currentIndex];
fileStr.replace("/playlist/", "");
fileStr.replace(".mp3", "");
```
2. Extraer Nombre de Canción y Artista:
```cpp
int separatorIndex = fileStr.indexOf('_');
String songName, artistName;
if (separatorIndex != -1) {
  songName = fileStr.substring(0, separatorIndex);
  artistName = fileStr.substring(separatorIndex + 1);
} else {
  songName = fileStr;
  artistName = "Desconocido";
}
```
3. Mostrar Información en la Pantalla:
```cpp
display.clearDisplay();
display.setTextSize(1);
display.setTextColor(SSD1306_WHITE);
display.setCursor(0, 10);
display.println(songName);
display.setCursor(0, 20);
display.println(artistName);
display.display();
```
