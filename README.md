# Esp32-Cam-Foto

### Codigo que funciona para una tomar una foto
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

### Codigo de pruebas que cuenta las cantidad de archivos en la tarjeta sd funciona
```c++
#include "FS.h"                // SD Card ESP32
#include "SD_MMC.h"            // SD Card ESP32

File root;
int fileCountOnSD = 0; // for counting files


void setup() {
  delay(2000);
  
  Serial.begin(115200);
  
  if(!SD_MMC.begin()) 
  {
    Serial.println("SD Card Mount Failed");
    return;
  }

  uint8_t cardType = SD_MMC.cardType();
  if (cardType == CARD_NONE) 
  {
    Serial.println("No SD Card attached");
    return;
  }

 // const char* dir = ("/");
  root = SD_MMC.open("/");
  
  //Cuenta todos los archivo del root las subcarpetas que tiene archivo no cuenta sus archvios
  //pero cuenta una carpeta como un archivo
  printDirectory(root);

  // Now print the total files count
  Serial.println(F("fileCountOnSD: "));
  Serial.println(fileCountOnSD);

  Serial.println("done!");

  
}

void loop() {
  // put your main code here, to run repeatedly:

}

void printDirectory(File dir) {
  while (true) 
  {
    File entry =  dir.openNextFile();
    if (!entry) 
    {
      // no more files
      break;
    }
    
    //for (uint8_t i = 0; i < numTabs; i++) {
     // Serial.print('\t');
    //}
    
    Serial.print(entry.name());
    // for each file count it
    fileCountOnSD++;

/*
    if (entry.isDirectory()) {
      Serial.println("/");
      printDirectory(entry, numTabs + 1);
    } else {
      // files have sizes, directories do not
      Serial.print("\t\t");
      Serial.println(entry.size(), DEC);
    }
    */
    entry.close();
  }
}
```

### Codigo experimental que cuenta archivo sd
```c++
#include "FS.h"
#include "SD_MMC.h"

int numFiles = 0;

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

}

void contarFiles()
{
  File root = SD_MMC.open("/");

  File entrar = null;
  
  while(entrar == false)
  {
    entrar = root.openNextFile();
    numFiles++;
  }

}


```

IDEA ORIGINALMENTE EN CODIGO UTILIZA LA MEMORIA EEPROM QUE ESTA OBSOLETA PERO INTERNAMENTE SE UTILIZA LA MEMORIA NVS, PORQUE UTILIZAR LA MEMORIA INTERNA DEL MICROCONTROLADOR Y NO UTILIZAR LA TARJETA SD QUE ES UNA MEJOR MEMORIA SIMPLEMENTE SE PUEDE CONTAR CUANTOS IMAGENES AHI EN LA CARPETA SACAR EL TOTAL Y EL CONTADOR SUMARLE +1 EN EL NOMBRE Y LISTO

FRAMESIZE_VGA mas pequeña pero con mejor resolucion
