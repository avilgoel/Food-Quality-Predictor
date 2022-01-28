# How to integrate Neuton model into your firmware project 

##	Extract archive

Copy two folders to your firmware project: `neuton` and `model`.

The `neuton` folder contains C source files: 
*	`neuton.c` – runtime that can load model and make predictions;
*	`calculator.c` – helper code that init/deinit `NeuralNet` structure, normalize input data, makes predictions, and calls user callback functions.

The `model` folder contains model file generated by platform as a C-array.
The `user_app.c` file is an example how to use calculator and contains all required callback functions.

## Load model

Create an instance of `NeuralNet`structure and call `CalculatorInit` function, which fill structure with zeroes and store context associated with 
a model in `neuralNet.data` field (can be `NULL`).
``` C
NeuralNet neuralNet;
CalculatorInit(&neuralNet, NULL);
```
To load model from Flash/ROM you should call `CalculatorLoadFromMemory` function in callback function `CalculatorOnInit`:
``` C
extern const unsigned char model_bin[];
extern const unsigned int model_bin_len;

Err CalculatorOnInit(NeuralNet* neuralNet)
{
  return CalculatorLoadFromMemory(neuralNet, model_bin, model_bin_len, 0);
}
```
Here `model_bin` and `model_bin_len` symbols should be resolved if you include `model.c` to project build. 
A third parameter `copy=0` tells runtime that we are used model from memory and don’t want to copy data to RAM. 

After successful model loading `CalculatorOnLoad` callback will be called. In this callback you can implement additional 
logic specific for your application and return `ERR_NO_ERROR` code.

``` C
Err CalculatorOnLoad(NeuralNet* neuralNet)
{
  ...
  return ERR_NO_ERROR;
}
```

## Make inference

To make prediction you should call `CalculatorRunInference` function. Function needs two parameters: `NeuralNet` structure and a `buffer`. Buffer is a `float` array of `NeuralNet.inputsDim` elements and contains your features plus one `BIAS` value at the end (should be `1.0`). It is important that features in the same order as when model learning.

When we call `CalculatorRunInference` function, three functions will be called back:
* `CalculatorOnInferenceStart` – after features normalization, before inference;
* `CalculatorOnInferenceEnd` – after inference, before denormalization;
* `CalculatorOnInferenceResult` – after inference and output denormalization.

For example, you can start timer in `CalculatorOnInferenceStart` and stop it in `CalculatorOnInferenceEnd` to measure inference time or leave functions empty.

`CalculatorOnInferenceResult` will be called with result array of `NeuralNet.outputsDim` elements. As example, for regression task there will be one element – target value, for classification task – two elements: probabilities of classes.

``` C
float sample[7] = {
  846,    // PT08.S3(NOx)
  1638,   // PT08.S4(NO2)
  991,    // PT08.S5(O3)
  11.3,   // T
  78.1,   // RH
  1.0411, // AH
  1.0,    // BIAS
};
	
float* result = CalculatorRunInference(&neuralNet, sample);
void CalculatorOnInferenceResult(NeuralNet* neuralNet, float* result)
{
  if (neuralNet->taskType == TASK_BINARY_CLASSIFICATION && neuralNet->outputsDim == 2)
  {
    const float* value = result[0] >= result[1] ? &result[0] : &result[1];
    if (*value > 0.5)
    {
      if (value == &result[0])
      {
        // 0
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_SET);
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_RESET);
      }
      else
      {
        // 1
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_SET);
      }
    }
    else
    {
      // unknown
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, GPIO_PIN_RESET);
      HAL_GPIO_WritePin(GPIOA, GPIO_PIN_2, GPIO_PIN_RESET);
    }
  }
}
```

If you don’t want to use callback function `CalculatorOnInferenceResult` - leave it empty and use return value from `CalculatorRunInference`.

## Free resources

To release `NeuralNet` structure, call `CalculatorFree` function. It will free resources associated with model and call `CalculatorOnFree` callback with user context parameter.
