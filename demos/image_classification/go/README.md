# Image Classification Demo (Go) {#ovms_demo_image_classification_go}

This client demonstrates how to interact with OpenVINO Model Server prediction endpoints from a Go application. The example shows end-to-end workflow for running classification on JPEG/PNG images using a ResNet50 model. To simplify the environment setup, the demo is run inside a Docker container.

Clone the repository and enter directory:

```bash
git clone https://github.com/openvinotoolkit/model_server.git
cd model_server/demos/image_classification/go
```

## Get the model

To run end to end flow and get correct results, please download `resnet-50-tf` model and convert it to IR format by following [instructions available on the OpenVINO Model Zoo page](https://docs.openvino.ai/2022.1/omz_models_model_resnet_50_tf.html)

Place converted model files (XML and BIN) under the following path: `<PATH_TO_MODELS>/resnet-50-tf/1`

Where `PATH_TO_MODELS` is the path to the directory with models on the host filesystem.

For example:
```bash
mkdir models
docker run -u $(id -u):$(id -g) -v ${PWD}/models:/models openvino/ubuntu20_dev:latest omz_downloader --name resnet-50-tf --output_dir /models
docker run -u $(id -u):$(id -g) -v ${PWD}/models:/models:rw openvino/ubuntu20_dev:latest omz_converter --name resnet-50-tf --download_dir /models --output_dir /models --precisions FP32
mv ${PWD}/models/public/resnet-50-tf/FP32 ${PWD}/models/public/resnet-50-tf/1

tree models/public/resnet-50-tf
models/public/resnet-50-tf
├── 1
│   ├── resnet-50-tf.bin
│   ├── resnet-50-tf.mapping
│   └── resnet-50-tf.xml
└── resnet_v1-50.pb
```

## Build Go client docker image

Before building the image let's copy single zebra image, here so it's included in the docker build context. This way client container will already have a sample to run prediction on:

```bash
cp ../../common/static/images/zebra.jpeg .
``` 
Then build the docker image and tag it `ovmsclient`:
```bash
docker build . -t ovmsclient
```

## Start OpenVINO Model Server with ResNet model

Before running the client launch OVMS with prepared ResNet model. You can do that with a command similar to:

```bash
docker run -d --rm -p 9000:9000  -v ${PWD}/models/public/resnet-50-tf:/models/resnet openvino/model_server:latest --model_name resnet --model_path /models/resnet --port 9000
```

**Note** Layout for downloaded resnet model is NHWC. It ensures that the model will accept binary input generated by the client. See [binary inputs](../../../docs/binary_input.md) doc if you want to learn more about this feature.

## Run prediction with Go client

In order to run prediction on the model served by the OVMS using Go client run the following command:

```bash
docker run --net=host --rm ovmsclient --serving-address localhost:9000 zebra.jpeg
# exemplary output
2022/06/15 13:46:51 Request sent successfully
Predicted class: zebra
Classification confidence: 98.914299%
```

Command explained:
- `--net=host` option is required so the container with the client can access container with the model server via host network (localhost),
- `--serving-address` parameter defines the address of the model server gRPC endpoint,
- the last part in the command is a path to the image that will be send to OVMS for prediction. The image must be accessible from the inside of the container (could be mounted). Single zebra picture - `zebra.jpeg` - has been embedded in the docker image to simplify the example, so above command would work out of the box. If you wish to use other image you need to provide it to the container and change the path.

You can also choose if the image should be sent as binary input (raw JPG or PNG bytes) or should be converted on the client side to the data array accepted by the model.
To send raw bytes just add `--binary-input` flag like this:

```bash
docker run --net=host --rm ovmsclient --serving-address localhost:9000 --binary-input zebra.jpeg
# exemplary output
2022/06/15 13:46:53 Request sent successfully
Predicted class: zebra
Classification confidence: 98.914299%
```