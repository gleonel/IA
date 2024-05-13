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

## Set up a Node.js app
Install Node.js and npm. For supported platforms and dependencies, please see the tfjs-node installation guide.

Create a directory called ./baseball for our Node.js app. Copy the linked package.json and webpack.config.js into this directory to configure the npm package dependencies (including the @tensorflow/tfjs-node npm package). Then run npm install to install the dependencies.


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


Now you are ready to write some code and train a model!

## Setup the training and test data
You will use the training and test data as CSV files from the links below. Download and explore the data in these files:

pitch_type_training_data.csv

pitch_type_test_data.csv

Let's look at some sample training data:


<br>

``` bash

vx0,vy0,vz0,ax,ay,az,start_speed,left_handed_pitcher,pitch_code
7.69914900671662,-132.225686405648,-6.58357157666866,-22.5082591074995,28.3119270826735,-16.5850095967027,91.1,0,0
6.68052308575228,-134.215511616881,-6.35565979491619,-19.6602769147989,26.7031848314466,-14.3430602022656,92.4,0,0
2.56546504690782,-135.398673977074,-2.91657310799559,-14.7849950586111,27.8083916890792,-21.5737737390901,93.1,0,0

```
<br>


There are eight input features - describing pitch sensor data:

- ball velocity (vx0, vy0, vz0)
- ball acceleration (ax, ay, az)
- starting speed of pitch
- whether pitcher is left handed or not
  

and one output label,pitch_code that signifies one of seven pitch types: 
- Fastball (2-seam), 
- Fastball (4-seam), 
- Fastball (sinker), 
- Fastball (cutter), 
- Slider, 
- Changeup, 
- Curveball

The goal is to build a model that is able to predict the pitch type given pitch sensor data.

Before creating the model, you need to prepare the training and test data. Create the file [`pitch_type.js` html] in the [`baseball/` html] dir, and create the following code into it. This code loads training and test data using the [`tf.data.csv` html] API. It also normalizes the data (which is always recommended) using a min-max normalization scale.

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

## Create model to classify pitch types
Now you are ready to build the model. Use the tf.layers API to connect the inputs (shape of [8] pitch sensor values) to 3 hidden fully-connected layers consisting of ReLU activation units, followed by one softmax output layer consisting of 7 units, each representing one of the output pitch types.

Train the model with the adam optimizer and the sparseCategoricalCrossentropy loss function.

Add the following code to the end of [`pitch_type.js` html]:

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


Trigger the training from the main server code that you will write later.

To complete the pitch_type.js module, let's write a function to evaluate the validation and test data set, predict a pitch type for a single sample, and compute accuracy metrics. Append this code to the end of [`pitch_type.js` html]:

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

## Train model on the server

Write the server code to perform model training and evaluation in a new file called server.js. First, create an HTTP server and open a bidirectional socket connection using the socket.io API. Then execute model training using the [`model.fitDataset API` html], and evaluate model accuracy using the [`pitch_type.evaluate()` html] method you wrote earlier. Train and evaluate for 10 iterations, printing metrics to the console.

Copy the below code to server.js:

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


At this point, you are ready to run and test the server! You should see something like this, with the server training one epoch in each iteration (you could also use the [`model.fitDataset API` html] to train multiple epochs with one call). If you encounter any errors at this point, please check your node and npm installation.

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

## Create client page and display code

Now that the server is ready, the next step is to write the client code and that runs in the browser. Create a simple page to invoke model prediction on the server and display the result. This uses `socket.io` for client/server communication.

First, create `index.html` in the` baseball/` folder:

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


Then create a new file `client.js` in the `baseball/` folder with the below code:

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

The client handles the `trainingComplete` socket message to display a prediction button. When this button is clicked, the client sends a socket message with sample sensor data. Upon receiving a `predictResult` message, it displays the prediction on the page.

## Run the app
Run both the server and client to see the full app in action:

<br>

``` bash

[In one terminal, run this first]
$ npm run start-client

[In another terminal, run this next]
$ npm run start-server

```
<br>

Open the client page in your browser ( `http://localhost:8080`). When model training finishes, click on the Predict Sample button. You should see a prediction result displayed in the browser. Feel free to modify the sample sensor data with some examples from the test CSV file and see how accurately the model predicts.