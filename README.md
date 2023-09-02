# Esp32-Cam

### Codigo para tomar una foto
```c++
#include "esp_camera.h"
#include "FS.h"
#include "SD_MMC.h"

//Declaracion de pins para camara MODEL_AI_THINKER
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

//Numero de foto
int fotoNum = 1;

void setup()
{
  //Inicia el puerto serial
  Serial.begin(115200);

  //Configuracion de la camara
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  //Verifica si es compatible con PSRAM y elige la configuracion adecuada
  if (psramFound())
  {
    config.frame_size = FRAMESIZE_VGA; //Tamaño de la foto, se pueden elegir QVGA|CIF|VGA|SVGA|XGA|SXGA|UXGA, mientras mas pequeña mejor resolucion
    config.jpeg_quality = 10; //10-63 mientras menor el numero mejor calidad
    config.fb_count = 2;
  }

  else
  {
    config.frame_size = FRAMESIZE_VGA; //Tamaño de la foto, se pueden elegir QVGA|CIF|VGA|SVGA|XGA|SXGA|UXGA, mientras mas pequeña mejor resolucion
    config.jpeg_quality = 10; //10-63 mientras menor el numero mejor calidad
    config.fb_count = 2;
  }

  //Inicializar la camera con la configuracion
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK)
  {
    Serial.println("Camera no inicializada" + err);
    return;
  }

  //SD incializada correctamente
  if (!SD_MMC.begin())
  {
    Serial.println("SD Card Montada incorrectamente");
    return;
  }

  uint8_t cardType = SD_MMC.cardType();

  //En caso que no se detecte una tarjeta sd
  if (cardType == CARD_NONE)
  {
    Serial.println("No SD Card introducida");
    return;
  }

}

void loop()
{
  TomarFoto();
}

void TomarFoto()
{
  camera_fb_t * fb = NULL;

  //Toma la foto
  fb = esp_camera_fb_get();
  if (!fb)
  {
    Serial.println("La camara no pudo realizar la fotografia");
    return;
  }

  //Ruta de la tarjeta sd donde se guardara la foto
  String ruta = "/foto" + String(fotoNum) + ".jpg";

  fs::FS &fs = SD_MMC;
  Serial.println("Foto nombre de archivo: " + ruta);

  File archivo = fs.open(ruta, FILE_WRITE);
  if (!archivo)
  {
    Serial.println("Fallo en abrir el archivo para escribir");
  }

  else
  {
    //Escribe en la tarjeta sd la foto
    archivo.write(fb->buf, fb->len);
    Serial.println("Archivo guardado en la ruta: " + ruta);
  }

  //Cierra el archivo
  archivo.close();

  esp_camera_fb_return(fb);

  //Incrementa el valor
  fotoNum++;

  //Retardo de 2 segundos
  delay(2000);
}

```

### Codigo que cuenta la cantidad de archivos de la tarjeta sd
```c++
#include "FS.h"
#include "SD_MMC.h"

void setup()
{
  //Inicia el puerto serial
  Serial.begin(115200);

  //SD incializada correctamente
  if (!SD_MMC.begin())
  {
    Serial.println("SD Card Montada incorrectamente");
    return;
  }

  uint8_t cardType = SD_MMC.cardType();

  //En caso que no se detecte una tarjeta sd
  if (cardType == CARD_NONE)
  {
    Serial.println("No SD Card introducida");
    return;
  }

}

void loop()
{
  //La funcion para contar archivos que devuelve la cantidad de archivos
  int contarArchivos = contarFiles();

  //Imprime la cantidad de archivos de la sd
  Serial.print("Cantidad de archivos: ");
  Serial.println(contarArchivos);

  //Retraso de 5 segundos
  delay(5000);
}

int contarFiles()
{
  //Inicia en -1 porque en el bucle do-while se hace un paso adicional
  int numFiles = -1;

  //Abre la raiz de la sd
  File root = SD_MMC.open("/");

  //Crea la variable tipo File
  File entrar;

  //Realiza un ciclo do-while para no comprobar la primera condicion
  do {
    entrar = root.openNextFile(); //Se mueve al archivo siguiente
    numFiles++; //Aumenta el contador
    //Recordar que una carpeta se lo detecta como un archivo
    //Tambien recordar que los archivos ocultos del sistema los detecta
  }
  while (entrar == true); //Mientras es true se queda en bucle

  //Devuelve la cantidad de archivos
  return numFiles;
}
```

### Codigo completo, funcionando que al iniciar cuenta la cantidad de imagenes en la sd y captura una nueva imagen cada 2 segundos  

```c++
#include "esp_camera.h"
#include "FS.h"
#include "SD_MMC.h"

//Declaracion de pins para camara MODEL_AI_THINKER
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

//Numero de foto
int fotoNum;

void setup()
{
  //Inicia el puerto serial
  Serial.begin(115200);

  //SD incializada correctamente
  if (!SD_MMC.begin())
  {
    Serial.println("SD Card Montada incorrectamente");
    return;
  }

  uint8_t cardType = SD_MMC.cardType();

  //En caso que no se detecte una tarjeta sd
  if (cardType == CARD_NONE)
  {
    Serial.println("No SD Card introducida");
    return;
  }

  //Configuracion de la camara
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  //Verifica si es compatible con PSRAM y elige la configuracion adecuada
  if (psramFound())
  {
    config.frame_size = FRAMESIZE_VGA; //Tamaño de la foto, se pueden elegir QVGA|CIF|VGA|SVGA|XGA|SXGA|UXGA, mientras mas pequeña mejor resolucion
    config.jpeg_quality = 10; //10-63 mientras menor el numero mejor calidad
    config.fb_count = 2;
  }

  else
  {
    config.frame_size = FRAMESIZE_VGA; //Tamaño de la foto, se pueden elegir QVGA|CIF|VGA|SVGA|XGA|SXGA|UXGA, mientras mas pequeña mejor resolucion
    config.jpeg_quality = 10; //10-63 mientras menor el numero mejor calidad
    config.fb_count = 2;
  }

  //Inicializar la camera con la configuracion
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK)
  {
    Serial.println("Camera no inicializada" + err);
    return;
  }

  //SD incializada correctamente
  if (!SD_MMC.begin())
  {
    Serial.println("SD Card Montada incorrectamente");
    return;
  }

  //La funcion para contar archivos que devuelve la cantidad de archivos
  fotoNum = contarFiles();

}

void loop()
{
  TomarFoto();
}

void TomarFoto()
{
  camera_fb_t * fb = NULL;

  //Toma la foto
  fb = esp_camera_fb_get();
  if (!fb)
  {
    Serial.println("La camara no pudo realizar la fotografia");
    return;
  }

  //Ruta de la tarjeta sd donde se guardara la foto
  String ruta = "/foto" + String(fotoNum) + ".jpg";

  fs::FS &fs = SD_MMC;
  Serial.println("Foto nombre de archivo: " + ruta);

  File archivo = fs.open(ruta, FILE_WRITE);
  if (!archivo)
  {
    Serial.println("Fallo en abrir el archivo para escribir");
  }

  else
  {
    //Escribe en la tarjeta sd la foto
    archivo.write(fb->buf, fb->len);
    Serial.println("Archivo guardado en la ruta: " + ruta);
  }

  //Cierra el archivo
  archivo.close();

  esp_camera_fb_return(fb);

  //Incrementa el valor
  fotoNum++;

  //Retardo de 2 segundos
  delay(2000);
}

int contarFiles()
{
  //La definicion de esta variable con (-1) devuelve el total de archivo al definirlo con (0) devuelve el total de archivos +1  
  int numFiles = 0;

  //Abre la raiz de la sd
  File root = SD_MMC.open("/");

  //Crea la variable tipo File
  File entrar;

  //Realiza un ciclo do-while para no comprobar la primera condicion
  do {
    entrar = root.openNextFile(); //Se mueve al archivo siguiente
    numFiles++; //Aumenta el contador
    //Recordar que una carpeta se lo detecta como un archivo
    //Tambien recordar que los archivos ocultos del sistema los detecta
  }
  while (entrar == true); //Mientras es true se queda en bucle

  //Devuelve la cantidad de archivos
  return numFiles;
}
```



IDEA ORIGINALMENTE EN CODIGO UTILIZA LA MEMORIA EEPROM QUE ESTA OBSOLETA PERO INTERNAMENTE SE UTILIZA LA MEMORIA NVS, PORQUE UTILIZAR LA MEMORIA INTERNA DEL MICROCONTROLADOR Y NO UTILIZAR LA TARJETA SD QUE ES UNA MEJOR MEMORIA SIMPLEMENTE SE PUEDE CONTAR CUANTOS IMAGENES AHI EN LA CARPETA SACAR EL TOTAL Y EL CONTADOR SUMARLE +1 EN EL NOMBRE Y LISTO

FRAMESIZE_VGA mas pequeña pero con mejor resolucion

### Codigo aparte con display
* recordar que para funcione la librerias de adafruit se debe descargar la libreria Adafruit BusIO lo pide como dependencia
* mas informacion en https://github.com/adafruit/Adafruit_VEML6075/issues/5
* Adicionalmente se debe descargar gfx de adafruit https://github.com/adafruit/Adafruit-GFX-Library
* y ademas para el modulo en especifico https://github.com/adafruit/Adafruit-ST7735-Library

### Codigo que compila pero no probado
* mas informacion sobre el uso de esta pantalla https://www.instructables.com/Value-Your-Project-Use-Graphic-Display/
* mas informacion sobre jppg convertidor nose para que sirve pero dicen que funciona https://github.com/Bodmer/TJpg_Decoder/blob/master/examples/SD%20Card/SD_Jpg/SD_Jpg.ino#L33

```c++
#include <Adafruit_GFX.h> // Core graphics library
#include <Adafruit_ST7735.h> // Hardware-specific library for ST7735
#include <SPI.h>
 
#define TFT_DC 17 //A0
#define TFT_CS 21 //CS
#define TFT_MOSI 2 //SDA
#define TFT_CLK 23 //SCK
#define TFT_RST 0 
#define TFT_MISO 0
 
Adafruit_ST7735 tft = Adafruit_ST7735(TFT_CS, TFT_DC, TFT_MOSI, TFT_CLK, TFT_RST);

void setup() {
   tft.setCursor(0, 0);
  //tft.setTextColor(color);
  tft.setTextWrap(true);
  tft.print("hola mundo");

}

void loop() {
  // put your main code here, to run repeatedly:

}
```

### Codigo experimental no comprobado de que muestra una imagen en pantalla led
```c++
#include <SPI.h>

#include <FS.h>

//#include <SD.h>
#include "SD_MMC.h"

#include <TFT_eSPI.h>

// JPEG decoder library
#include <JPEGDecoder.h>

//#define TFT_DC 21       //already defined  User_setup.h
//#define TFT_CS 22
//#define TFT_MOSI 23
//#define TFT_SCLK 18
//#define TFT_MISO 19

//#define ILI9341_DRIVER

//#define sd_cs 32

TFT_eSPI tft = TFT_eSPI();

//####################################################################################################
// Setup
//####################################################################################################
void setup() {
  Serial.begin(115200);

  // Set all chip selects high to avoid bus contention during initialisation of each peripheral
  //digitalWrite(22, HIGH); // Touch controller chip select (if used)
  digitalWrite(22, HIGH); // TFT screen chip select
  //digitalWrite(sd_cs, HIGH); // SD card chips select, must use GPIO 5 (ESP32 SS)

  tft.begin();
  tft.setRotation(1);
  tft.fillScreen(TFT_WHITE);

 //SD incializada correctamente
  if (!SD_MMC.begin())
  {
    Serial.println("SD Card Montada incorrectamente");
    return;
  }

  uint8_t cardType = SD_MMC.cardType();

  //En caso que no se detecte una tarjeta sd
  if (cardType == CARD_NONE)
  {
    Serial.println("No SD Card introducida");
    return;
  }
  
}

//####################################################################################################
// Main loop
//####################################################################################################
void loop() {

  tft.setRotation(2);  // portrait
  tft.fillScreen(random(0xFFFF));

  // The image is 300 x 300 pixels so we do some sums to position image in the middle of the screen!
  // Doing this by reading the image width and height from the jpeg info is left as an exercise!
  int x = (tft.width()  - 300) / 2 - 1;
  int y = (tft.height() - 300) / 2 - 1;

  drawSdJpeg("/EagleEye.jpg", x, y);     // This draws a jpeg pulled off the SD Card
  delay(2000);

  tft.setRotation(2);  // portrait
  tft.fillScreen(random(0xFFFF));
  drawSdJpeg("/Baboon40.jpg", 0, 0);     // This draws a jpeg pulled off the SD Card
  delay(2000);

  tft.setRotation(2);  // portrait
  tft.fillScreen(random(0xFFFF));
  drawSdJpeg("/lena20k.jpg", 0, 0);     // This draws a jpeg pulled off the SD Card
  delay(2000);

  tft.setRotation(1);  // landscape
  tft.fillScreen(random(0xFFFF));
  drawSdJpeg("/Mouse480.jpg", 0, 0);     // This draws a jpeg pulled off the SD Card

  delay(2000);

  while(1); // Wait here
}

//####################################################################################################
// Draw a JPEG on the TFT pulled from SD Card
//####################################################################################################
// xpos, ypos is top left corner of plotted image
void drawSdJpeg(const char *filename, int xpos, int ypos) {

  // Open the named file (the Jpeg decoder library will close it)
  File jpegFile = SD_MMC.open( filename, FILE_READ);  // or, file handle reference for SD library
 
  if ( !jpegFile ) {
    Serial.print("ERROR: File \""); Serial.print(filename); Serial.println ("\" not found!");
    return;
  }

  Serial.println("===========================");
  Serial.print("Drawing file: "); Serial.println(filename);
  Serial.println("===========================");

  // Use one of the following methods to initialise the decoder:
  bool decoded = JpegDec.decodeSdFile(jpegFile);  // Pass the SD file handle to the decoder,
  //bool decoded = JpegDec.decodeSdFile(filename);  // or pass the filename (String or character array)

  if (decoded) {
    // print information about the image to the serial port
    jpegInfo();
    // render the image onto the screen at given coordinates
    jpegRender(xpos, ypos);
  }
  else {
    Serial.println("Jpeg file format not supported!");
  }
}

//####################################################################################################
// Draw a JPEG on the TFT, images will be cropped on the right/bottom sides if they do not fit
//####################################################################################################
// This function assumes xpos,ypos is a valid screen coordinate. For convenience images that do not
// fit totally on the screen are cropped to the nearest MCU size and may leave right/bottom borders.
void jpegRender(int xpos, int ypos) {

  //jpegInfo(); // Print information from the JPEG file (could comment this line out)

  uint16_t *pImg;
  uint16_t mcu_w = JpegDec.MCUWidth;
  uint16_t mcu_h = JpegDec.MCUHeight;
  uint32_t max_x = JpegDec.width;
  uint32_t max_y = JpegDec.height;

  bool swapBytes = tft.getSwapBytes();
  tft.setSwapBytes(true);
  
  // Jpeg images are draw as a set of image block (tiles) called Minimum Coding Units (MCUs)
  // Typically these MCUs are 16x16 pixel blocks
  // Determine the width and height of the right and bottom edge image blocks
  uint32_t min_w = jpg_min(mcu_w, max_x % mcu_w);
  uint32_t min_h = jpg_min(mcu_h, max_y % mcu_h);

  // save the current image block size
  uint32_t win_w = mcu_w;
  uint32_t win_h = mcu_h;

  // record the current time so we can measure how long it takes to draw an image
  uint32_t drawTime = millis();

  // save the coordinate of the right and bottom edges to assist image cropping
  // to the screen size
  max_x += xpos;
  max_y += ypos;

  // Fetch data from the file, decode and display
  while (JpegDec.read()) {    // While there is more data in the file
    pImg = JpegDec.pImage ;   // Decode a MCU (Minimum Coding Unit, typically a 8x8 or 16x16 pixel block)

    // Calculate coordinates of top left corner of current MCU
    int mcu_x = JpegDec.MCUx * mcu_w + xpos;
    int mcu_y = JpegDec.MCUy * mcu_h + ypos;

    // check if the image block size needs to be changed for the right edge
    if (mcu_x + mcu_w <= max_x) win_w = mcu_w;
    else win_w = min_w;

    // check if the image block size needs to be changed for the bottom edge
    if (mcu_y + mcu_h <= max_y) win_h = mcu_h;
    else win_h = min_h;

    // copy pixels into a contiguous block
    if (win_w != mcu_w)
    {
      uint16_t *cImg;
      int p = 0;
      cImg = pImg + win_w;
      for (int h = 1; h < win_h; h++)
      {
        p += mcu_w;
        for (int w = 0; w < win_w; w++)
        {
          *cImg = *(pImg + w + p);
          cImg++;
        }
      }
    }

    // calculate how many pixels must be drawn
    uint32_t mcu_pixels = win_w * win_h;

    // draw image MCU block only if it will fit on the screen
    if (( mcu_x + win_w ) <= tft.width() && ( mcu_y + win_h ) <= tft.height())
      tft.pushImage(mcu_x, mcu_y, win_w, win_h, pImg);
    else if ( (mcu_y + win_h) >= tft.height())
      JpegDec.abort(); // Image has run off bottom of screen so abort decoding
  }

  tft.setSwapBytes(swapBytes);

  showTime(millis() - drawTime); // These lines are for sketch testing only
}

//####################################################################################################
// Print image information to the serial port (optional)
//####################################################################################################
// JpegDec.decodeFile(...) or JpegDec.decodeArray(...) must be called before this info is available!
void jpegInfo() {

  // Print information extracted from the JPEG file
  Serial.println("JPEG image info");
  Serial.println("===============");
  Serial.print("Width      :");
  Serial.println(JpegDec.width);
  Serial.print("Height     :");
  Serial.println(JpegDec.height);
  Serial.print("Components :");
  Serial.println(JpegDec.comps);
  Serial.print("MCU / row  :");
  Serial.println(JpegDec.MCUSPerRow);
  Serial.print("MCU / col  :");
  Serial.println(JpegDec.MCUSPerCol);
  Serial.print("Scan type  :");
  Serial.println(JpegDec.scanType);
  Serial.print("MCU width  :");
  Serial.println(JpegDec.MCUWidth);
  Serial.print("MCU height :");
  Serial.println(JpegDec.MCUHeight);
  Serial.println("===============");
  Serial.println("");
}

//####################################################################################################
// Show the execution time (optional)
//####################################################################################################
// WARNING: for UNO/AVR legacy reasons printing text to the screen with the Mega might not work for
// sketch sizes greater than ~70KBytes because 16 bit address pointers are used in some libraries.

// The Due will work fine with the HX8357_Due library.

void showTime(uint32_t msTime) {
  //tft.setCursor(0, 0);
  //tft.setTextFont(1);
  //tft.setTextSize(2);
  //tft.setTextColor(TFT_WHITE, TFT_BLACK);
  //tft.print(F(" JPEG drawn in "));
  //tft.print(msTime);
  //tft.println(F(" ms "));
  Serial.print(F(" JPEG drawn in "));
  Serial.print(msTime);
  Serial.println(F(" ms "));
}
```
