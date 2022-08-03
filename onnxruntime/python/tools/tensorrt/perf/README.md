# TensorRT EP Performance Test Script
This script mainly focus on benchmarking ORT TensorRT EP performance compared with CUDA EP and standalone TensorRT. The metrics includes TensorRT EP performance gain, percentage of model operators and execution time that run on TensorRT EP.

## Usage with Azure Pipelines

### Linux GPU TensorRT Perf CI Pipeline
- [x] **Build ORT** Build ORT from source. Specify _branch_ variable if not master.
- [x] **Post to Dashboard** Post to ONNX Runtime EP Dashboard (No Docker).
- [ ] **Run in Docker (CUDA 11.0)** Check to run in CUDA 11.0 vs 10.2 (default). 
- [ ] **Configure EPs** Choose which EPs to run against. Specify _epList_ variable.
- **ModelGroups**: Select which model groups to run. (i.e. selected-models, specify _selected-models_ variable)

#### Variables (under Advanced Options) 
- **branch**: (*default: master*) Specified branch to run against.
- **epList**: List of EPs to run separated by spaces [from available options](https://github.com/microsoft/onnxruntime/blob/master/onnxruntime/python/tools/tensorrt/perf/benchmark.py#L26) _i.e. "CPUExecutionProvider TensorrtExecutionProvider"_
- **selected-models**: Specified path to model json file or model folder if selected-models in ModelGroups. 

## Usage Locally 
You can use following command to test whether models can be run using TensorRT and run benchmark:
```
./perf.sh
```
If you only want to run benchmark or use randomly generated input data instead of input data from ONNX model zoo, please use following command:
```
python3 benchmark_wrapper.py -r benchmark -i random -t 100
```
or
```
python3 benchmark.py -r benchmark -i random -t 100
```
Please note that benchmark_wrapper.py creates one process to execute benchmark.py for every model and every ep, therefore, when process runs into segmentation fault and is forced to exit, the wrapper can catch the error.
However, benchmark.py creates only one process to run all the model inferences on all eps, once the process triggers segmentation fault, the whole process is forced to exit and can't successfully capture the error and testing results.

### Options
- **-r, --running_mode**: (*defaul: benchmark*) There are two types of running mode, *validate* and *benchmark*. For validation, this test script records any runtime error as well as validates the accuracy of prediction result using *np.testing.assert_almost_equal()* and exposes result that doesn't meet accuracy requirement. For benchmark, it simply runs model inference assuming model is correct and get the performance metrics. (Note: If you run validation first and then benchmark, test script knows which model has issue and will skip benchmarking of this particular model.)
- **-m, --model_source**: (*default: model_list.json*) There are two ways to specify list of models to test. (1) Explicitly specify model list file which contains model information. (2) Specify directory which has following layout:
```
    --Directory
      --ModelName1
          --test_data_set_0
              --input0.pb
          --test_data_set_2
              --input0.pb
          --model.onnx
      --ModelName2
          --test_data_set_0
              --input0.pb
          --test_data_set_2
              --input0.pb
          --model.onnx
```
- **-i, --input_data**: (*default: random*) Where is the input data coming from. The value are *zoo* or *random*. The input data can be from ONNX model zoo or it can be randomly generated by test script.
- **-t, --test_times**: (*default: 1*) Number of inference run when in 'benchmark' running mode.
- **-w, --workspace**: (*default: ./*) Root directory of perf dir. Tensorrt_home should be discoverable from this path.
- **-o, --perf_result_path**: (*default: result*) Directory for perf result..
- **--fp16**: (*default: True*) Enable TensorRT/CUDA FP16 and include the performance of this floating point optimization.
- **--trtexec**: Path of standalone TensorRT executable, for example: trtexec.
- **--track_memory**: Track memory usage of CUDA and TensorRT execution providers.
- **--ep_list**: Optional argument. List of eps to run against, surround with quotes and separate with spaces.
- **--trt_ep_options**: Optional argument. Comma-separated key/value pairs denoting TensorRT EP options. Ex: `--trt_ep_options trt_engine_cache_enable=True,trt_max_workspace_size=4294967296`. Refer to [TensorRT Execution Provider Options](https://onnxruntime.ai/docs/execution-providers/TensorRT-ExecutionProvider.html#execution-provider-options) for a complete list of options.
- **--cuda_ep_options**: Optional argument. Comma-separated key/value pairs denoting CUDA EP options. Ex: `--cuda_ep_options device_id=0,arena_extend_strategy=kNextPowerOfTwo`. Refer to [CUDA Execution Provider Options](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#configuration-options) for a complete list of options.


### Validation Configuration 
- **--percent_mismatch**: The allowed percentage of values to be incorrect when comparing given outputs to ORT outputs. 
- **--rtol**: The relative tolerance for validating ORT outputs.
- **--atol**: The absolute tolerance for validating ORT outputs.

### Results
After running validation and benchmark. The metrics are written into five different csv files in 'result' directory or the directory you specified with -o argument.
- **benchmark_fail_xxxx.csv**: Lists all the models that fail to be inferenced by TensorRT/CUDA.
- **benchmark_success_xxxx.csv**: Lists all the models that can be successfully inferenced by TensorRT/CUDA, as well as other related metrics.
- **benchmark_latency_xxxx.csv**: Lists all the models with inference latecy of TensorRT/CUDA and TensorRT Float32/Float16 performance gain compared with CUDA.
- **benchmark_metrics_xxxx.csv**: List how much and percentage of model operators that are run by TensorRT and what percentage of execution time is running on TensorRT.
- **benchmark_status_xxxx.csv**: List of all the models and the status as pass or fail for each execution provider.
- **benchmark_system_info_xxxx.csv**: includes CUDA version, TensorRT version and CPU information.

Thoese metrics will be shown on the standard output as well.

The output of running validation:
```
Total time for running/profiling all models: 0:20:30.761618
['bert-squad', 'faster-rcnn', 'mask-rcnn', 'ssd', 'tiny-yolov2', 'resnet152v1']

============================================
========== Failing Models/EPs ==============
============================================
{'faster-rcnn': ['CUDAExecutionProvider_fp16'], 'mask-rcnn': ['CUDAExecutionProvider_fp16']}

========================================
========== Models/EPs metrics ==========
========================================
{   'BERT-Squad': {   'ratio_of_execution_time_in_trt': 0.9980344366695495,
                      'ratio_of_ops_in_trt': 0.9989451476793249,
                      'total_execution_time': 12719,
                      'total_ops': 948,
                      'total_ops_in_trt': 947,
                      'total_trt_execution_time': 12694}}

```

The output of running benchmark:
```

=========================================
=========== Models/EPs latency  ===========
=========================================
{   'ResNet101-DUC-7': {   'CPUExecutionProvider': {   'average_latency_ms': '10664.04',
                                                       'latency_90_percentile': '10923.44'},
                           'CUDAExecutionProvider': {   'average_latency_ms': '406.66',
                                                        'latency_90_percentile': '412.55',
                                                        'memory': 13259},
                           'CUDAExecutionProvider_fp16': {   'average_latency_ms': '129.93',
                                                             'latency_90_percentile': '130.80',
                                                             'memory': 13003},
                           'Standalone_TRT': {   'average_latency_ms': '365.334 ',
                                                 'latency_90_percentile': '369.046 ',
                                                 'memory': 1443},
                           'Standalone_TRT_fp16': {   'average_latency_ms': '105.71 ',
                                                      'latency_90_percentile': '107.01 ',
                                                      'memory': 1071},
                           'TRT_CUDA_fp16_gain(%)': '51.54 %',
                           'TRT_CUDA_gain(%)': '14.07 %',
                           'TRT_Standalone_fp16_gain(%)': '40.44 %',
                           'TRT_Standalone_gain(%)': '4.35 %',
                           'TensorrtExecutionProvider': {   'average_latency_ms': '349.43',
                                                            'latency_90_percentile': '352.53',
                                                            'memory': 1941},
                           'TensorrtExecutionProvider_fp16': {   'average_latency_ms': '62.96',
                                                                 'latency_90_percentile': '63.96',
                                                                 'memory': 1257}},
}

```

```
=========================================
=========== CUDA/TRT Status =============
=========================================
{   'BERT-Squad': {   'CUDAExecutionProvider': 'Pass',
                      'CUDAExecutionProvider_fp16': 'Pass',
                      'TensorrtExecutionProvider': 'Pass',
                      'TensorrtExecutionProvider_fp16': 'Fail'}
}
```

#### Comparing Runs
```
python comparison_script.py -p "prev" -c "current" -o "output.csv"
```
- **compare_latency.py**: creates a csv file with any regressions in average latencies 
- **new_failures.py**: creates a csv file with any new failures

## Others

### Setting Up Perf Models
- setup_onnx_zoo.py: Create a text file 'links.txt' with download links from onnx zoo models, or setup in the same folder structure. Extracts the models and creates the json file perf script will be run with.
- setup_many_models.sh: ./setup_many_models "wget_link_to_models" to extract all the models.

### Building ORT Env
build_images.sh: This script should be run before running run_perf_docker.sh to make sure the docker images are up to date.
- **-p, --perf_dockerfile_path**: Path to EP Perf Docker File.
- **-i, --image**: What the perf docker image will be named.
- **-b, --branch**: ORT branch name you are perf testing on.

ort_build_latest.py: This script should be run before running run_perf_machine.sh or benchmark.py to make sure the latest ORT wheel file is being used.
- **-o, --ort_master_path**: ORT master repo.
- **-t, --tensorrt_home**: TensorRT home directory.
- **-c, --cuda_home**: CUDA home directory.
- **-b, --branch**: (*default: master*) ORT branch name you are perf testing on.

### Running Perf Script 
run_perf_docker.sh: Runs the perf script in docker environment. 
- **-d, --docker_image**: Name of perf docker image.
- **-o, --option**: Name of which models you want to run {i.e. selected-models}
- **-p, --perf_dir**: Path to perf directory.
- **-m, --model_path**: Model path relative to workspace (/). If option is selected-models, include path to models either json or folder.

run_perf_machine.sh: Runs the perf script directly. 
- **-o, --option**: Name of which models you want to run {i.e. selected-models}
- **-m, --model_path**: Model path relative to workspace (~/). If option is selected-models, include path to models either json or folder.

## Dependencies
- When inferencing model using CUDA float16, this script following script to convert nodes in model graph from float32 to float16. It also modifies the converting script a little bit to better cover more model graph conversion.
https://github.com/microsoft/onnxconverter-common/blob/master/onnxconverter_common/float16.py
- For dynamic input shape models, the script runs symbolic shape inference on the model. https://github.com/microsoft/onnxruntime/blob/master/onnxruntime/python/tools/symbolic_shape_infer.py