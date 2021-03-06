TensorRT Optimization
====================================

This part of the repository is based on jkjung's [TensorRT](https://github.com/jkjung-avt/tensorrt_demos) repository.

TensorRT is an SDK for high-peformance deep learning inference. The workflow follows the picture below. Neural Network training is commonly done on the host, a high-performance PC, and afterwards the model is optimized for deployment on the target platform.



![TensorRT_workflow](docs/tensor_rt_workflow.jpeg)



It is important to mention that TensorRT performs platform specific optimizations to guarantee the highest performance during inference. **This means that a serialized engine optimized on the Jetson Xavier AGX is not portable between platforms (e.g. Jetson Nano).** 

Furthermore, TensorRT has a lot of dependencies with the software in which the Neural Network is trained. TensorRT optimization is not compatible with models trained in some Tensorflow versions. This is mainly due to the Tensorflow API, which renames nodes upon releases. This ultimately results in TensorRT not recognizing some of them during the optimization phase. This can be circumvented by using graphsurgeon to remove/rename nodes. Nevertheless, for simplicity, this project provides a fully contained environment with the right versions to guarantee a seamless process.

*Note: At the time of writing, UFF and ONNX parsers are supported by TensorRT. Nevertheless, at this point we'll only work with the UFF parser.*

## Setup

From here onwards we'll assume the [training phase](../tensorflow_training) is done using our provided environment with Tensorflow-gpu 1.12. Otherwise, successful optimization is not guaranteed. Furthermore, we'll assume the target platform is the [NVIDIA Xavier AGX](https://developer.nvidia.com/embedded/jetson-agx-xavier-developer-kit). 

Before proceeding with the optimization, you must ensure the latest Jetpack version is installed in the target platform. At the time of writing, the system was tested using Jetpack 4.3, which comes with all the necessary dependencies to get up and running in no time (Cuda, CuDNN, OpenCV). The steps for installation can be found on [NVIDIA's website](https://developer.nvidia.com/embedded/jetpack).

Once Jetpack is installed, run the install script located in this folder. This script will add extra paths, dependencies and patches required to run the optimization process.

```
$ cd tensor_rt
$ ./install.sh
```



## Running the Optimization

The first step to perform TensorRT optimization is to convert frozen graph files (.pb) provided by software such as Tensorflow to [UFF](https://docs.nvidia.com/deeplearning/sdk/tensorrt-api/python_api/uff/uff.html) format. This format will later be parsed by TensorRT  to create an 'Engine' which will be later used by the TensorRT runtime to perform inference.

The *build_engine.py* loads the frozen graph file (.pb), and saves the uff model along a *.pbtxt file for debugging. Afterwards, the TensorRT engine will be created, and saved a .bin file. 

To modify the build_engine to your requirements, the only need that must be done is to include your model to MODEL_SPECS. The number of classes must match the number of classes of the given model, plus an additional one (for the background). Furthermore min_size and max_size are parameters which come from the config file of the model (look for ssd_anchor_generator min_scale and max_scale parameters ). 

```
MODEL_SPECS = {

    'ssd_mobilenet_v2_coco': {
        'input_pb':   os.path.abspath(os.path.join(
                          DIR_NAME, 'ssd_mobilenet_v2_coco.pb')),
        'tmp_uff':    os.path.abspath(os.path.join(
                          DIR_NAME, 'ssd_mobilenet_v2_coco.uff')),
        'output_bin': os.path.abspath(os.path.join(
                          DIR_NAME, 'TRT_ssd_mobilenet_v2_coco.bin')),
        'num_classes': 91, #NUM_CLASSES + 1
        'min_size': 0.2, #from config file
        'max_size': 0.95,#from config file
        'input_order': [0, 2, 1],  # order of loc_data, conf_data, priorbox_data
    },
}
```



Finally, for *input_order*, we advice you to follow the [0, 2, 1] pattern. If it fails, verify the generated *.pbtxt* and look for the following lines. 

```
graphs {
	id: "main"
	nodes {
		id: "NMS"
		inputs: "squeeze"
		inputs: "concat_priorbox"
		inputs: "concat_box_conf"
		operation: "_NMS_TRT"
		fields {
		key: "backgroundLabelId_u_int"
		value {
			i: 1
.
.
.
```

*input_order* lists the order in which *loc_data*, *conf_data* and *priorbox_data* appear in the *.pbtxt* file. *loc_data* appears first (squeeze), therefore, it is on index 0. *conf_data* appears last , therefore it is on index 2. Finally priorbox_data is index 1, thus, creating [0, 2, 1].



Finally, we optimize the model by executing the build_engine

```
$ python build_engine.py ssd_mobilenet_v2_coco
```

*Note: This process takes a while.*

If everything went smoothly, you should see no errors on screen. The engine is now the *.bin file.