# Esp32 Cam

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

### Codigo que inicia contando la cantidad de imagenes dentro de la sd y toma una fotografia cada 2 segundos  
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

### Esp32 Cam Funcionando
<img src="https://github.com/IDiegoUlises/Esp32-Cam-Foto-En-SD/blob/main/Imagenes/IMG_20230912_220816.jpg" />

### Fotos de la Camara
<img src="https://github.com/IDiegoUlises/Esp32-Cam-Foto-En-SD/blob/main/Imagenes/Foto-de-la-camara.png" width="1000" height="500" />

