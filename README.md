# FFMPEG SuperResolution with OpenVINO

This is a work in progress trying to demonstrate how to use OpenVINO
to run SR models in FFMPEG. It doesn't currently work.

## OpenVINO

Used latest release, OpenVINO 2022.1.0.

Build a minimal compilation to get a CPU only OpenVINO build that can
be used in ffmpeg using the following cmake

```
mkdir build && cd build
cmake .. \
    -DCMAKE_BUILD_TYPE=Release" \
    -DCMAKE_INSTALL_PREFIX=/usr/local/openvino \
	-DENABLE_OV_PADDLE_FRONTEND=OFF \
	-DENABLE_OV_ONNX_FRONTEND=OFF \
	-DENABLE_OV_TF_FRONTEND=OFF \
	-DENABLE_CPPLINT=OFF \
	-DENABLE_SAMPLES=OFF \
	-DENABLE_TESTS=OFF \
	-DENABLE_PROFILING_ITT=OFF \
	-DENABLE_CLDNN=OFF \
	-DENABLE_HETERO=OFF \
	-DENABLE_INTEL_CPU=ON \
	-DENABLE_INTEL_GNA=OFF \
	-DENABLE_INTEL_GPU=OFF \
	-DENABLE_INTEL_MYRIAD=OFF \
	-DENABLE_INTEL_MYRIAD_COMMON=OFF \
	-DENABLE_MULTI=OFF \
	-DENABLE_ONEDNN_FOR_GPU=OFF \
	-DENABLE_OPENCL_LAYERS=OFF \
	-DENABLE_VPU=OFF \
	-DENABLE_OPENCV=OFF \
	-DENABLE_PYTHON=OFF \
	-DENABLE_AUTO=OFF \
	-DENABLE_REQUIREMENTS_INSTALL=OFF" \
	-DTBB_INCLUDE_DIRS=/usr/local/tbb/include \
	-DTBB_LIBRARIES_RELEASE=/usr/local/tbb/lib/libtbb.a \
```

## Ffmpeg Patch

Latest ffmpeg doesn't build against libraries from OpenVINO 2022.1.0,
and it's actually not clear what version is required. THe following
patch changes ffmpeg to link against the new library names.

In ffmpeg `./configure`

```
-enabled libopenvino       && require libopenvino c_api/ie_c_api.h ie_c_api_version -linference_engine_c_api
+enabled libopenvino       && require libopenvino c_api/ie_c_api.h ie_c_api_version "-lopenvino_c -lopenvino"
```

Actually only `-lopevino_c` is needed for current DNN backend, but add
in `-lopenvino` in case we need newer APIs.

Then build ffmpeg with `--enable-libopenvino`

## TF Model

Ffmpeg SR filter currently links to example models for ESPCN and SRCNN. Trying with ESPCN first. Download TensorFlow model from https://github.com/XueweiMeng/sr

There is code in here to freeze the model, but it doesn't work with TF 2.x, so instead replace

```
-import tensorflow as tf
+import tensorflow.compat.v1 as tf
```

This generates the `espcn.pb` model included in this repo.

## Convert to OpenVINO

A full build of OpenVINO is needed to get Model Optimizer, including
python, not the limited build to link
against ffmpeg.

Convert the TF model to OpenVINO IR format.

```
 mo --input_model espcn.pb
```

This creates the `espcn.xml` and `espcn.bin` files included in this
repo.

## Try to run through `dnn_processing` filter

The SR filter in ffmpeg doesn't expose an OpenVINO backend. So instead
attempt to use the full `dnn_processing` filter.

```
ffmpeg -i input.mp4 -vf "dnn_processing=dnn_backend=openvino:model=espcn.xml:input=x:output=espcn/prediction:backend_configs='device=CPU\&input_resizable=1',scale=-1:1080:flags=lanczos"  -y output.mp4
```

This then shows an error

```
[Parsed_dnn_processing_0] the frame's format yuv420p does not match the model input channel 0
```

This message is somewhat confusing but looking ffmpeg source it shows
that the problem is that it's expecting the input to have a channel
count of 1 (given a yuv420p input) but it read a channel count of 0.

https://github.com/FFmpeg/FFmpeg/blob/d0a999a0ab8313fd1b5e9cb09e35fb769fb3e51c/libavfilter/vf_dnn_processing.c#L121
