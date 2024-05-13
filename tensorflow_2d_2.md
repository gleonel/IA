# Predição de dados 2D  -  Carregando dados csv e utilizando node

Antes de iniciar vamos entender um pouco mais sobre o modelo multicamadas das redes neurais:

[Redes neurais: estrutura](https://developers.google.com/machine-learning/crash-course/introduction-to-neural-networks/anatomy?hl=pt-br)

e sobre as  [Funções de ativação](https://developers.google.com/machine-learning/glossary?hl=pt-br#activation-function)

As funções de ativação são um componente crucial em redes neurais artificiais que introduzem não linearidade nos modelos de aprendizado de máquina. Sem elas, as redes neurais seriam apenas modelos lineares simples, incapazes de aprender e modelar relações complexas que existem em dados do mundo real.

Imagine uma rede neural sem nenhuma função de ativação. Os dados fluiriam através da rede, camada por camada, sofrendo apenas transformações lineares. Matematicamente, isso significa que a saída de cada neurônio seria uma combinação linear de suas entradas.

Assim temos os exemplos mais comuns de funções de ativação:

- A função binária classifica entradas em duas categorias (por exemplo, spam ou não spam).
- A função sigmóide normaliza a saída para um intervalo entre 0 e 1, útil para tarefas como regressão logística.
- A função ReLU permite que os neurônios se concentrem apenas em entradas positivas, ignorando entradas negativas.

## Aplicativo Node.js.

Instale o Node.js e o npm. 

Crie um diretório chamado `./baseball` para nosso aplicativo `Node.js`. Copie o [`package.json`](/code/baseball/package.json) e [`webpack.config.js`](/code/baseball/webpack.config.js) vinculados neste diretório para configurar as dependências do pacote npm (incluindo o pacote npm `@tensorflow/tfjs-node`). 

Em seguida, execute `npm install` para instalar as dependências.


<br>

``` bash

$ cd baseball
$ ls
package.json  webpack.config.js
$ npm install
...
$ ls
node_modules  package.json  package-lock.json  webpack.config.js

```
<br>



## Configure os dados de treinamento e teste

Você usará os dados de treinamento e teste como arquivos CSV nos links abaixo. Baixe e explore os dados nestes arquivos:

[pitch_type_training_data.csv](https://storage.googleapis.com/mlb-pitch-data/pitch_type_training_data.csv)

[pitch_type_test_data.csv](https://storage.googleapis.com/mlb-pitch-data/pitch_type_test_data.csv)

Vejamos alguns exemplos de dados de treinamento:

<br>

``` bash

vx0,vy0,vz0,ax,ay,az,start_speed,left_handed_pitcher,pitch_code
7.69914900671662,-132.225686405648,-6.58357157666866,-22.5082591074995,28.3119270826735,-16.5850095967027,91.1,0,0
6.68052308575228,-134.215511616881,-6.35565979491619,-19.6602769147989,26.7031848314466,-14.3430602022656,92.4,0,0
2.56546504690782,-135.398673977074,-2.91657310799559,-14.7849950586111,27.8083916890792,-21.5737737390901,93.1,0,0

```
<br>


Existem oito recursos de entrada - que descrevem os dados do sensor de inclinação:


- velocidade da bola (vx0, vy0, vz0)
- aceleração da bola (ax, ay, az)
- velocidade inicial de arremesso
- se o arremessador é canhoto ou não
  

e um rótulo de saída, pitch_code que significa um dos [sete tipos de pitch](https://www.mlb.com/glossary/pitch-types):

- Fastball (2-seam)
- Fastball (4-seam)
- Fastball (sinker)
- Fastball (cutter)
- Slider, 
- Changeup, 
- Curveball

O objetivo é construir um modelo que seja capaz de prever o tipo de pitch com base nos dados do sensor de pitch.

Antes de criar o modelo, você precisa preparar os dados de treinamento e teste. 

Crie o arquivo [`pitch_type.js` html] no diretório [`baseball/` html] e crie o seguinte código nele. 

Este código carrega dados de treinamento e teste usando a API [`tf.data.csv` html]. Também normaliza os dados (o que é sempre recomendado) usando uma escala de normalização min-max.

<br>

``` js

const tf = require('@tensorflow/tfjs');

// util function to normalize a value between a given range.

function normalize(value, min, max) {
  if (min === undefined || max === undefined) {
    return value;
  }
  return (value - min) / (max - min);
}

// data can be loaded from URLs or local file paths when running in Node.js.

const TRAIN_DATA_PATH =
'https://storage.googleapis.com/mlb-pitch-data/pitch_type_training_data.csv';

const TEST_DATA_PATH =    'https://storage.googleapis.com/mlb-pitch-data/pitch_type_test_data.csv';

// Constants from training data
const VX0_MIN = -18.885;
const VX0_MAX = 18.065;
const VY0_MIN = -152.463;
const VY0_MAX = -86.374;
const VZ0_MIN = -15.5146078412997;
const VZ0_MAX = 9.974;
const AX_MIN = -48.0287647107959;
const AX_MAX = 30.592;
const AY_MIN = 9.397;
const AY_MAX = 49.18;
const AZ_MIN = -49.339;
const AZ_MAX = 2.95522851438373;
const START_SPEED_MIN = 59;
const START_SPEED_MAX = 104.4;

const NUM_PITCH_CLASSES = 7;
const TRAINING_DATA_LENGTH = 7000;
const TEST_DATA_LENGTH = 700;

// Converts a row from the CSV into features and labels.
// Each feature field is normalized within training data constants
const csvTransform =
    ({xs, ys}) => {
      const values = [
        normalize(xs.vx0, VX0_MIN, VX0_MAX),
        normalize(xs.vy0, VY0_MIN, VY0_MAX),
        normalize(xs.vz0, VZ0_MIN, VZ0_MAX), normalize(xs.ax, AX_MIN, AX_MAX),
        normalize(xs.ay, AY_MIN, AY_MAX), normalize(xs.az, AZ_MIN, AZ_MAX),
        normalize(xs.start_speed, START_SPEED_MIN, START_SPEED_MAX),
        xs.left_handed_pitcher
      ];
      return {xs: values, ys: ys.pitch_code};
    }

const trainingData =
    tf.data.csv(TRAIN_DATA_PATH, {columnConfigs: {pitch_code: {isLabel: true}}})
        .map(csvTransform)
        .shuffle(TRAINING_DATA_LENGTH)
        .batch(100);

// Load all training data in one batch to use for evaluation
const trainingValidationData =
    tf.data.csv(TRAIN_DATA_PATH, {columnConfigs: {pitch_code: {isLabel: true}}})
        .map(csvTransform)
        .batch(TRAINING_DATA_LENGTH);

// Load all test data in one batch to use for evaluation
const testValidationData =
    tf.data.csv(TEST_DATA_PATH, {columnConfigs: {pitch_code: {isLabel: true}}})
        .map(csvTransform)
        .batch(TEST_DATA_LENGTH);

```
<br>

## Crie um modelo para classificar os tipos de pitch

Agora você está pronto para construir o modelo. Use a API `tf.layers` para conectar as entradas (formato de [8] valores do sensor de pitch) a 3 camadas ocultas totalmente conectadas que consistem em unidades de ativação `ReLU`, seguidas por uma camada de saída `softmax` que consiste em 7 unidades, cada uma representando uma das saídas (tipos de pitch), a soma das saídas da softmax é sempre 1.

Treine o modelo com o otimizador Adam e a função de perda sparseCategoricalCrossentropy.

Adicione o seguinte código ao final de [`pitch_type.js` html]:

<br>

``` js

const model = tf.sequential();
model.add(tf.layers.dense({units: 250, activation: 'relu', inputShape: [8]}));
model.add(tf.layers.dense({units: 175, activation: 'relu'}));
model.add(tf.layers.dense({units: 150, activation: 'relu'}));
model.add(tf.layers.dense({units: NUM_PITCH_CLASSES, activation: 'softmax'}));

model.compile({
  optimizer: tf.train.adam(),
  loss: 'sparseCategoricalCrossentropy',
  metrics: ['accuracy']
});

```
<br>



Para completar o módulo `pitch_type.js`, vamos escrever uma função para avaliar o conjunto de dados de validação e teste, prever um tipo de pitch para uma única amostra e calcular métricas de precisão. 

Anexe este código ao final de [`pitch_type.js` html]:

<br>

``` js

// Returns pitch class evaluation percentages for training data
// with an option to include test data
async function evaluate(useTestData) {
  let results = {};
  await trainingValidationData.forEachAsync(pitchTypeBatch => {
    const values = model.predict(pitchTypeBatch.xs).dataSync();
    const classSize = TRAINING_DATA_LENGTH / NUM_PITCH_CLASSES;
    for (let i = 0; i < NUM_PITCH_CLASSES; i++) {
      results[pitchFromClassNum(i)] = {
        training: calcPitchClassEval(i, classSize, values)
      };
    }
  });

  if (useTestData) {
    await testValidationData.forEachAsync(pitchTypeBatch => {
      const values = model.predict(pitchTypeBatch.xs).dataSync();
      const classSize = TEST_DATA_LENGTH / NUM_PITCH_CLASSES;
      for (let i = 0; i < NUM_PITCH_CLASSES; i++) {
        results[pitchFromClassNum(i)].validation =
            calcPitchClassEval(i, classSize, values);
      }
    });
  }
  return results;
}

async function predictSample(sample) {
  let result = model.predict(tf.tensor(sample, [1,sample.length])).arraySync();
  var maxValue = 0;
  var predictedPitch = 7;
  for (var i = 0; i < NUM_PITCH_CLASSES; i++) {
    if (result[0][i] > maxValue) {
      predictedPitch = i;
      maxValue = result[0][i];
    }
  }
  return pitchFromClassNum(predictedPitch);
}

// Determines accuracy evaluation for a given pitch class by index
function calcPitchClassEval(pitchIndex, classSize, values) {
  // Output has 7 different class values for each pitch, offset based on
  // which pitch class (ordered by i)
  let index = (pitchIndex * classSize * NUM_PITCH_CLASSES) + pitchIndex;
  let total = 0;
  for (let i = 0; i < classSize; i++) {
    total += values[index];
    index += NUM_PITCH_CLASSES;
  }
  return total / classSize;
}

// Returns the string value for Baseball pitch labels
function pitchFromClassNum(classNum) {
  switch (classNum) {
    case 0:
      return 'Fastball (2-seam)';
    case 1:
      return 'Fastball (4-seam)';
    case 2:
      return 'Fastball (sinker)';
    case 3:
      return 'Fastball (cutter)';
    case 4:
      return 'Slider';
    case 5:
      return 'Changeup';
    case 6:
      return 'Curveball';
    default:
      return 'Unknown';
  }
}

module.exports = {
  evaluate,
  model,
  pitchFromClassNum,
  predictSample,
  testValidationData,
  trainingData,
  TEST_DATA_LENGTH
}

```
<br>

## Treine o modelo no servidor

Escreva o código do servidor para realizar o treinamento e avaliação do modelo em um novo arquivo chamado server.js. 

Primeiro, crie um servidor HTTP e abra uma conexão de soquete bidirecional usando a API socket.io. 

Em seguida, execute o treinamento do modelo usando o [`model.fitDataset API` html] e avalie a precisão do modelo usando o método [`pitch_type.evaluate()` html] que você escreveu anteriormente. 

Treine e avalie 10 iterações, imprimindo métricas no console.

Crie o código abaixo no arquivo `server.js`:

<br>

``` js

require('@tensorflow/tfjs-node');

const http = require('http');
const socketio = require('socket.io');
const pitch_type = require('./pitch_type');

const TIMEOUT_BETWEEN_EPOCHS_MS = 500;
const PORT = 8001;

// util function to sleep for a given ms
function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// Main function to start server, perform model training, and emit stats via the socket connection
async function run() {
  const port = process.env.PORT || PORT;
  const server = http.createServer();
  const io = socketio(server);

  server.listen(port, () => {
    console.log(`  > Running socket on port: ${port}`);
  });

  io.on('connection', (socket) => {
    socket.on('predictSample', async (sample) => {
      io.emit('predictResult', await pitch_type.predictSample(sample));
    });
  });

  let numTrainingIterations = 10;
  for (var i = 0; i < numTrainingIterations; i++) {
    console.log(`Training iteration : ${i+1} / ${numTrainingIterations}`);
    await pitch_type.model.fitDataset(pitch_type.trainingData, {epochs: 1});
    console.log('accuracyPerClass', await pitch_type.evaluate(true));
    await sleep(TIMEOUT_BETWEEN_EPOCHS_MS);
  }

  io.emit('trainingComplete', true);
}

run();


```
<br>


Neste ponto, você está pronto para executar e testar o servidor! 

Você deverá ver algo assim, com o servidor treinando uma época em cada iteração (você também pode usar o [`model.fitDataset API` html] para treinar múltiplas épocas com uma chamada). 

Se você encontrar algum erro neste ponto, verifique a instalação do seu Node e npm.

<br>

``` bash

$ npm run start-server
...
  > Running socket on port: 8001
Epoch 1 / 1
eta=0.0 ========================================================================================================>
2432ms 34741us/step - acc=0.429 loss=1.49

```
<br>

## Crie a página do cliente e exiba o código

Agora que o servidor está pronto, o próximo passo é escrever o código do cliente e rodar no navegador. 

Crie uma página simples para invocar a previsão do modelo no servidor e exibir o resultado. 

Isso usa `socket.io` para comunicação cliente/servidor.

Primeiro, crie `index.html` na pasta` baseball/`:

<br>

``` html

<!doctype html>
<html>
  <head>
    <title>Pitch Training Accuracy</title>
  </head>
  <body>
    <h3 id="waiting-msg">Waiting for server...</h3>
    <p>
    <span style="font-size:16px" id="trainingStatus"></span>
    <p>
    <div id="predictContainer" style="font-size:16px;display:none">
      Sensor data: <span id="predictSample"></span>
      <button style="font-size:18px;padding:5px;margin-right:10px" id="predict-button">Predict Pitch</button><p>
      Predicted Pitch Type: <span style="font-weight:bold" id="predictResult"></span>
    </div>
    <script src="dist/bundle.js"></script>
    <style>
      html,
      body {
        font-family: Roboto, sans-serif;
        color: #5f6368;
      }
      body {
        background-color: rgb(248, 249, 250);
      }
    </style>
  </body>
</html>

```
<br>


Em seguida, crie um novo arquivo `client.js` na pasta `baseball/` com o código abaixo:


<br>

``` js

import io from 'socket.io-client';
const predictContainer = document.getElementById('predictContainer');
const predictButton = document.getElementById('predict-button');

const socket =
    io('http://localhost:8001',
       {reconnectionDelay: 300, reconnectionDelayMax: 300});

const testSample = [2.668,-114.333,-1.908,4.786,25.707,-45.21,78,0]; // Curveball

predictButton.onclick = () => {
  predictButton.disabled = true;
  socket.emit('predictSample', testSample);
};

// functions to handle socket events
socket.on('connect', () => {
    document.getElementById('waiting-msg').style.display = 'none';
    document.getElementById('trainingStatus').innerHTML = 'Training in Progress';
});

socket.on('trainingComplete', () => {
  document.getElementById('trainingStatus').innerHTML = 'Training Complete';
  document.getElementById('predictSample').innerHTML = '[' + testSample.join(', ') + ']';
  predictContainer.style.display = 'block';
});

socket.on('predictResult', (result) => {
  plotPredictResult(result);
});

socket.on('disconnect', () => {
  document.getElementById('trainingStatus').innerHTML = '';
  predictContainer.style.display = 'none';
  document.getElementById('waiting-msg').style.display = 'block';
});

function plotPredictResult(result) {
  predictButton.disabled = false;
  document.getElementById('predictResult').innerHTML = result;
  console.log(result);
}

```
<br>

O cliente lida com a mensagem do soquete `trainingComplete` para exibir um botão de previsão. 

Quando este botão é clicado, o cliente envia uma mensagem de soquete com dados do sensor de amostra. 

Ao receber uma mensagem `predictResult`, exibe a previsão na página.

## Execute o aplicativo

Execute o servidor e o cliente para ver o aplicativo completo em ação:

<br>

``` bash

[In one terminal, run this first]
$ npm run start-client

[In another terminal, run this next]
$ npm run start-server

```
<br>

Abra a página do cliente em seu navegador ( `http://localhost:8080`). Quando o treinamento do modelo terminar, clique no botão Prever amostra. 

Você deverá ver um resultado de previsão exibido no navegador. 

Sinta-se à vontade para modificar os dados do sensor de amostra com alguns exemplos do arquivo CSV de teste e ver a precisão com que o modelo prevê.