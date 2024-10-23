# Projeto SALVA: Sistema de Auxílio e Localização de Vítimas em Áreas Alagadas

## Contexto

O **Projeto SALVA** tem como principal objetivo auxiliar equipes de resgate durante situações de inundações. Utilizando técnicas de visão computacional, o sistema é capaz de reconhecer telhados e pessoas presas em áreas alagadas, facilitando a identificação e localização dessas vítimas para que possam ser resgatadas com maior rapidez e eficiência.

Este sistema foi desenvolvido com o intuito de fornecer suporte à identificação automática de estruturas como telhados, que frequentemente são o único ponto visível durante enchentes, e também detectar pessoas que estejam em situação de emergência. Ao melhorar a precisão e a velocidade do reconhecimento dessas imagens, o projeto contribui significativamente para as operações de resgate e salvamento em situações críticas.

## Introdução

Neste projeto, iremos desenvolver um detector de objetos utilizando o modelo YOLOv5 em conjunto com a ESP32-CAM. Vamos criar um banco de imagens para treinar o modelo e, em seguida, realizar o treinamento no Google Colab, utilizando imagens capturadas em diferentes condições para melhorar a precisão da detecção.

## Como usar o sistema

Para usar o sistema, siga os passos abaixo:

### 1. Criar uma pasta no Google Drive
Crie uma pasta no Google Drive. O nome dessa pasta pode ser de sua escolha e que tenha estes arquivos:
```bash
images/
   train/   # 48 imagens para treinamento
   val/     # 12 imagens para validação
labels/
   train/   # Labels correspondentes às imagens de treinamento
   val/     # Labels correspondentes às imagens de validação
test/
telhado.yml/
```
Todos estes arquivos estão disponíveis no repositório

### 2. Organizar os arquivos
Dentro dessa pasta criada no Google Drive, você deve incluir os 4 arquivos disponíveis no repositório, organizados nas seguintes pastas e ordem:

- **pasta/images**: Contém as imagens de treino e teste que serão utilizadas para treinar o modelo.
- **pasta/labels**: Contém arquivos em formato `.txt` com as labels associadas às imagens. Cada arquivo de label consiste em várias matrizes de números que representam as coordenadas das caixas delimitadoras (bounding boxes) dos objetos nas imagens.
- **pasta/tests**: Contém os dados para testar o modelo após o treinamento estar concluído.
- **pasta/telhado.yml**: Este arquivo contém as variáveis de configuração e classes que o modelo deve prever. No caso deste projeto, o modelo está configurado para identificar "telhado" e "boneco" (representando pessoas).

### 3. Criando seu Próprio Modelo
Caso queira montar o seu próprio modelo, você deve seguir os seguintes passos:

#### 3.1 Busca de Imagens
- Procure e baixe imagens contendo os objetos que você deseja detectar.

- Renomeie as imagens para facilitar a organização. Exemplo de nome: telhado_01.

- Separe 80% das imagens para treino e 20% para validação.

- Coloque essas imagens nas respectivas pastas train e val dentro da pasta images.

#### 3.2 Criação de Rótulos
Agora que temos as imagens separadas, precisamos criar os rótulos de cada objeto. Para isso, vamos usar o site makesense.ai.

- Acesse o site e clique em Get started.

- Faça o upload das imagens da pasta train.

- Selecione Object Detection e crie os rótulos para seus objetos (exemplo: telhado).

- Crie caixas delimitadoras para cada objeto na imagem (detalhe: use o retângulo (rect), pois o modelo YOLOv5 não aceita outros tipos.)

- Exporte os rótulos no formato YOLOv5 e salve na pasta correspondente (labels/train).

Repita o processo para a pasta val.

### 4. Como Treinar o Modelo
- Caso utilize o Colab, É necessário a montagem do Google Drive:
   ```bash
    from google.colab import drive
    drive.mount('/content/drive')
    ``` 
- Clonagem do Repositório YOLOv5
   ```bash
   git clone https://github.com/seu-usuario/projeto-SALVA.git
   ``` 
- Instalação de Dependências
   ```bash
   !pip install -r yolov5/requirements.txt
    ``` 
- Treinamento do Modelo
   ```bash
  !python yolov5/train.py --data telhado.yaml --weights yolov5s.pt --img 640 --epochs 40
   ``` 
- Testando o Modelo Treinado
```bash
    import os
    import subprocess

    def get_latest_train_run_folder():
        subfolders = [f.path for f in os.scandir('yolov5/runs/train') if f.is_dir()]
        latest_folder = max(subfolders, key=os.path.getctime, default=None)
        return latest_folder

    latest_run = get_latest_train_run_folder()
    result = subprocess.run(f'python yolov5/detect.py --weights {latest_run}/weights/best.pt --img 640 --source drive/MyDrive/PROJETOS/DRONE/2024/test/ --data yolov5/data/telhado.yaml', shell=True, capture_output=True, text=True)

    if latest_run:
        print(result.stdout)
        print(result.stderr)
    else:
        print("Não foi possível encontrar a pasta de treinamento mais recente.")
```

#### Código Completo:
```bash
from google.colab import drive
drive.mount('/content/drive')
!git clone https://github.com/ultralytics/yolov5.git
!pip install -r yolov5/requirements.txt
!python yolov5/train.py --data telhado.yaml --weights yolov5s.pt --img 640 --epochs 40

import os
import subprocess

def get_latest_train_run_folder():
    subfolders = [f.path for f in os.scandir('yolov5/runs/train') if f.is_dir()]
    latest_folder = max(subfolders, key=os.path.getctime, default=None)
    return latest_folder

latest_run = get_latest_train_run_folder()
result = subprocess.run(f'python yolov5/detect.py --weights {latest_run}/weights/best.pt --img 640 --source drive/MyDrive/PROJETOS/DRONE/2024/test/ --data yolov5/data/telhado.yaml', shell=True, capture_output=True, text=True)

if latest_run:
    print(result.stdout)
    print(result.stderr)
else:
    print("Não foi possível encontrar a pasta de treinamento mais recente.")

``` 
   
### 5. Integração com ESP32-CAM
Após o treinamento, o modelo pode ser usado na ESP32-CAM. O código abaixo configura a ESP32-CAM para capturar imagens, no Arduino.

```bash
#include <WebServer.h>
#include <WiFi.h>
#include <esp32cam.h>

// Configurações de rede WiFi
const char* WIFI_SSID = "sua_rede_wifi";
const char* WIFI_PASS = "sua_senha_wifi";
int8_t nivelBrilho = 100;
#define FLASH_GPIO_NUM 4

WebServer server(80);

static auto hiRes = esp32cam::Resolution::find(800, 600);

void setup(){
  Serial.begin(115200);
  
  // Configuração da câmera
  using namespace esp32cam;
  Config cfg;
  cfg.setPins(pins::AiThinker);
  cfg.setResolution(hiRes);
  cfg.setJpeg(80);
  
  bool ok = Camera.begin(cfg);
  Serial.println(ok ? "CAMERA OK" : "CAMERA FAIL");

  WiFi.begin(WIFI_SSID, WIFI_PASS);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
  
  server.on("/cam-hi.jpg", []() {
    auto frame = esp32cam::capture();
    server.send(200, "image/jpeg", frame->size(), frame->data());
  });
  
  server.begin();
  pinMode(FLASH_GPIO_NUM, OUTPUT);
}

void loop() {
  server.handleClient();
  analogWrite(FLASH_GPIO_NUM, nivelBrilho);
}

``` 


### 6. Execução do sistema no VsCode
Crie um arquivo Python, cole esse código e salve-o na mesma pasta em que está o .ino e o best.pt:

- Utilize o seguinte código:

```bash
import cv2
import torch
import numpy as np
import urllib
import pathlib
from pathlib import Path

pathlib.PosixPath = pathlib.WindowsPath

path = 'best.pt' # TROQUE PELO CAMINHO DO SEU PESO CASO QUEIRA (best.pt que foi gerado no treinamento) ex: yolov5/runs/train/exp9/weights/best.pt
image_url = 'http://192.168.10.250/cam-hi.jpg' # TROQUE PELO LINK GERADO NO MONITOR SERIAL

model = torch.hub.load('ultralytics/yolov5', 'custom', path, force_reload=True)
model.conf = 0.6 #essa variável determina a acurácia do seu modelo. Faça testes para encontrar o melhor resultado.

print(path)

while True:
    img_resp=urllib.request.urlopen(url=image_url)
    imgnp=np.array(bytearray(img_resp.read()),dtype=np.uint8)
    im = cv2.imdecode(imgnp,-1)
    results = model(im)
    print(results)
    frame = np.squeeze(results.render())
    cv2.imshow('Deteccao', frame)
    key=cv2.waitKey(5)
    if key==ord('q'):
        break

cv2.destroyAllWindows()
``` 

- Só lembrando que o seu arquivo best.pt fica num path parecido como esse: ```yolov5/runs/train/exp9/weights/best.pt``` do seu Google Drive. Esse exp9 é o número de vezes que você treinou. Portanto, ele vai incrementar a cada treinamento.

## Solução de Problemas

Essa seção é para apontar principais falhas e suas soluções. Por enquanto, não houve falhas, mas à medida que forem surgindo, iremos atualizar essa seção.

Contudo, não custa lembrar de algumas dicas:

**A)** Caso dê erro nas bibliotecas, significa que você não tem todas as instâncias instaladas. Para cada erro de biblioteca que der, execute o respectivo ```pip install```:

**A1)** ```pip install torch torchvision torchaudio```

**A2)** ```pip.exe install opencv-contrib-python```

**A3)** ```pip.exe install pandas```

**A4)** ```pip install ultralytics --user```


**B)** Seu ESP32-CAM tem que estar na mesma rede WiFi que o seu PC. Verifique se ambos estão na mesma rede;

**C)** WiFi do celular Apple não é compatível com o ESP32;

**D)** WiFi local de empresas possuem proxy que podem impedir o bom funcionamento da rede. Sugiro roteador o WiFi do seu celular;

**E)** Se você ficou perdido no seu Colab quanto à sua pasta, porque minimizou os path, busque a pasta **content** que é o começo de tudo. Dentro de **content** temos as seguintes subpastas usadas no seu projeto:

* drive: aqui fica a sua estrutura de pastas e subpastas do treinamento e teste
* sample_data (pode ignorar)
* yolov5: aqui fica o **best.pt**, dentro da pasta **runs/train/exp/weights**
* yolov5s.pt: esse arquivo contém os pesos pré-treinados do modelo YOLOv5 de tamanho "small" (s). É uma versão compacta do modelo YOLOv5, que oferece um equilíbrio entre velocidade e precisão. Durante o treinamento, esse arquivo é usado como um ponto de partida para treinar o modelo no seu conjunto de dados específico.

**F)** Caso tenha esse erro abaixo, significa que seu ESP32 não está rodando. Para resolver, abra o seu navegador e digite o IP no campo da URL para testar a detecção de imagens.

```
urllib.error.URLError: <urlopen error [WinError 10060] Uma tentativa de conexão falhou porque o componente conectado não respondeu
corretamente após um período de tempo ou a conexão estabelecida falhou porque o host conectado não respondeu>
```

## Demonstração

Dentro do repositório, criamos uma pasta chamada ```demonstração``` onde você encontrará imagens e vídeos demonstrativos dos resultados do modelo em ação. Essa pasta será atualizada constantemente para refletir os avanços do projeto.

## Considerações Finais

O Projeto **SALVA** visa fornecer uma ferramenta robusta e eficiente para reconhecimento de objetos em áreas alagadas, com foco em salvar vidas durante desastres naturais. Com um sistema de detecção automática de telhados e pessoas, esperamos contribuir para que as operações de resgate sejam mais rápidas e precisas, facilitando o trabalho das equipes de salvamento.

Este repositório está em constante desenvolvimento e aprimoramento. Caso tenha dúvidas ou sugestões, sinta-se à vontade para abrir uma issue ou entrar em contato.

Agradecemos por contribuir para esse esforço de salvar vidas em situações de emergência!