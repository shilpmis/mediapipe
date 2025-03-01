# Copyright 2019 The MediaPipe Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

load("@bazel_skylib//lib:selects.bzl", "selects")
load("//mediapipe/framework/port:build_config.bzl", "mediapipe_proto_library")
load(
    "//mediapipe/framework/tool:mediapipe_graph.bzl",
    "mediapipe_binary_graph",
)
load("//mediapipe/framework:mediapipe_cc_test.bzl", "mediapipe_cc_test")
load("//mediapipe/framework:encode_binary_proto.bzl", "encode_binary_proto")

licenses(["notice"])

package(default_visibility = ["//visibility:private"])

selects.config_setting_group(
    name = "compute_shader_unavailable",
    match_any = [
        "//mediapipe/gpu:disable_gpu",
    ],
)

mediapipe_proto_library(
    name = "inference_calculator_proto",
    srcs = ["inference_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
    ],
)

cc_library(
    name = "inference_calculator_interface",
    srcs = ["inference_calculator.cc"],
    hdrs = ["inference_calculator.h"],
    copts = select({
        # TODO: fix tensor.h not to require this, if possible
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    deps = [
        ":inference_calculator_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/api2:node",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:ret_check",
        "//mediapipe/framework/stream_handler:fixed_size_input_stream_handler",
        "//mediapipe/framework/tool:subgraph_expansion",
        "//mediapipe/util/tflite:config",
        "//mediapipe/util/tflite:tflite_model_loader",
        "@com_google_absl//absl/memory",
        "@com_google_absl//absl/strings",
        "@org_tensorflow//tensorflow/lite:framework",
        "@org_tensorflow//tensorflow/lite/kernels:builtin_ops",
    ],
    alwayslink = 1,
)

cc_library(
    name = "inference_calculator_gl",
    srcs = ["inference_calculator_gl.cc"],
    tags = ["nomac"],  # config problem with cpuinfo via TF
    deps = [
        "inference_calculator_interface",
        "//mediapipe/gpu:gl_calculator_helper",
        "//mediapipe/gpu:gpu_buffer",
        "//mediapipe/util/tflite:tflite_gpu_runner",
        "@org_tensorflow//tensorflow/lite/delegates/gpu:gl_delegate",
        "@org_tensorflow//tensorflow/lite/delegates/gpu/common:shape",
        "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_buffer",
        "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_program",
        "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_shader",
    ],
    alwayslink = 1,
)

cc_library(
    name = "inference_calculator_metal",
    srcs = ["inference_calculator_metal.cc"],
    copts = [
        "-x objective-c++",
        "-fobjc-arc",  # enable reference-counting
    ],
    linkopts = [
        "-framework CoreVideo",
        "-framework MetalKit",
    ],
    tags = ["ios"],
    deps = [
        "inference_calculator_interface",
        "//mediapipe/gpu:MPPMetalHelper",
        "//mediapipe/gpu:MPPMetalUtil",
        "//mediapipe/gpu:gpu_buffer",
        "//mediapipe/objc:mediapipe_framework_ios",
        "//mediapipe/util/tflite:config",
        "@com_google_absl//absl/memory",
        "@org_tensorflow//tensorflow/lite/delegates/gpu:metal_delegate",
        "@org_tensorflow//tensorflow/lite/delegates/gpu:metal_delegate_internal",
        "@org_tensorflow//tensorflow/lite/delegates/gpu/common:shape",
        "@org_tensorflow//tensorflow/lite/delegates/gpu/metal:buffer_convert",
    ],
    alwayslink = 1,
)

cc_library(
    name = "inference_calculator_cpu",
    srcs = [
        "inference_calculator_cpu.cc",
    ],
    copts = select({
        # TODO: fix tensor.h not to require this, if possible
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    deps = [
        ":inference_calculator_interface",
        "@com_google_absl//absl/memory",
        "@org_tensorflow//tensorflow/lite/delegates/xnnpack:xnnpack_delegate",
    ] + select({
        "//conditions:default": [
            "//mediapipe/util:cpu_util",
        ],
    }) + select({
        "//conditions:default": [],
        "//mediapipe:android": ["@org_tensorflow//tensorflow/lite/delegates/nnapi:nnapi_delegate"],
    }),
    alwayslink = 1,
)

cc_library(
    name = "inference_calculator_gl_if_compute_shader_available",
    deps = select({
        ":compute_shader_unavailable": [],
        "//conditions:default": [":inference_calculator_gl"],
    }),
)

cc_library(
    name = "inference_calculator",
    visibility = ["//visibility:public"],
    deps = [
        ":inference_calculator_interface",
        ":inference_calculator_cpu",
    ] + select({
        "//conditions:default": [":inference_calculator_gl_if_compute_shader_available"],
        "//mediapipe:ios": [":inference_calculator_metal"],
    }),
    alwayslink = 1,
)

mediapipe_proto_library(
    name = "tensor_converter_calculator_proto",
    srcs = ["tensor_converter_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
    ],
)

cc_library(
    name = "tensor_converter_calculator",
    srcs = ["tensor_converter_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    features = ["-layering_check"],  # allow depending on tensor_converter_calculator_gpu_deps
    linkopts = select({
        "//mediapipe:apple": [
            "-framework CoreVideo",
            "-framework MetalKit",
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":tensor_converter_calculator_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/formats:image_frame",
        "//mediapipe/framework/formats:matrix",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:ret_check",
        "//mediapipe/framework:port",
        "//mediapipe/util:resource_util",
    ] + select({
        "//mediapipe/gpu:disable_gpu": [],
        "//conditions:default": ["tensor_converter_calculator_gpu_deps"],
    }),
    alwayslink = 1,
)

cc_library(
    name = "tensor_converter_calculator_gpu_deps",
    deps = select({
        "//mediapipe:android": [
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:gpu_buffer",
        ],
        "//mediapipe:ios": [
            "//mediapipe/gpu:MPPMetalUtil",
            "//mediapipe/gpu:MPPMetalHelper",
            "//mediapipe/objc:mediapipe_framework_ios",
        ],
        "//mediapipe:macos": [],
        "//conditions:default": [
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:gl_simple_shaders",
            "//mediapipe/gpu:shader_util",
            "//mediapipe/gpu:gpu_buffer",
        ],
    }),
)

cc_test(
    name = "tensor_converter_calculator_test",
    srcs = ["tensor_converter_calculator_test.cc"],
    deps = [
        ":tensor_converter_calculator",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:calculator_runner",
        "//mediapipe/framework/formats:image_format_cc_proto",
        "//mediapipe/framework/formats:image_frame",
        "//mediapipe/framework/formats:image_frame_opencv",
        "//mediapipe/framework/formats:matrix",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:gtest_main",
        "//mediapipe/framework/port:integral_types",
        "//mediapipe/framework/port:parse_text_proto",
        "//mediapipe/framework/tool:validate_type",
        "@com_google_absl//absl/memory",
        "@com_google_absl//absl/strings",
    ],
)

mediapipe_proto_library(
    name = "tensors_to_detections_calculator_proto",
    srcs = ["tensors_to_detections_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
    ],
)

cc_library(
    name = "tensors_to_detections_calculator",
    srcs = ["tensors_to_detections_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    features = ["-layering_check"],  # allow depending on tensors_to_detections_calculator_gpu_deps
    linkopts = select({
        "//mediapipe:apple": [
            "-framework CoreVideo",
            "-framework MetalKit",
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":tensors_to_detections_calculator_cc_proto",
        "//mediapipe/framework/formats:detection_cc_proto",
        "@com_google_absl//absl/strings:str_format",
        "@com_google_absl//absl/types:span",
        "//mediapipe/framework/api2:node",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:port",
        "//mediapipe/framework/deps:file_path",
        "//mediapipe/framework/formats:location",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/formats/object_detection:anchor_cc_proto",
        "//mediapipe/framework/port:ret_check",
    ] + select({
        ":compute_shader_unavailable": [],
        "//conditions:default": [":tensors_to_detections_calculator_gpu_deps"],
    }),
    alwayslink = 1,
)

cc_library(
    name = "tensors_to_detections_calculator_gpu_deps",
    deps = select({
        "//mediapipe:ios": [
            "//mediapipe/gpu:MPPMetalUtil",
            "//mediapipe/gpu:MPPMetalHelper",
        ],
        "//mediapipe:macos": [],
        "//conditions:default": [
            "//mediapipe/gpu:gl_calculator_helper",
        ],
    }),
)

mediapipe_proto_library(
    name = "tensors_to_landmarks_calculator_proto",
    srcs = ["tensors_to_landmarks_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
    ],
)

cc_library(
    name = "tensors_to_landmarks_calculator",
    srcs = ["tensors_to_landmarks_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":tensors_to_landmarks_calculator_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/api2:node",
        "//mediapipe/framework/formats:landmark_cc_proto",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:ret_check",
    ],
    alwayslink = 1,
)

mediapipe_proto_library(
    name = "tensors_to_floats_calculator_proto",
    srcs = ["tensors_to_floats_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
    ],
)

cc_library(
    name = "tensors_to_floats_calculator",
    srcs = ["tensors_to_floats_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":tensors_to_floats_calculator_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/api2:node",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:ret_check",
    ],
    alwayslink = 1,
)

cc_test(
    name = "tensors_to_floats_calculator_test",
    srcs = ["tensors_to_floats_calculator_test.cc"],
    deps = [
        ":tensors_to_floats_calculator",
        ":tensors_to_floats_calculator_cc_proto",
        "//mediapipe/framework:calculator_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:calculator_runner",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:gtest_main",
        "//mediapipe/framework/port:parse_text_proto",
        "@com_google_absl//absl/memory",
        "@com_google_googletest//:gtest_main",
    ],
)

cc_library(
    name = "tensors_to_classification_calculator",
    srcs = ["tensors_to_classification_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":tensors_to_classification_calculator_cc_proto",
        "@com_google_absl//absl/container:node_hash_map",
        "@com_google_absl//absl/strings:str_format",
        "@com_google_absl//absl/types:span",
        "//mediapipe/framework/api2:node",
        "//mediapipe/framework/formats:classification_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/formats:location",
        "//mediapipe/framework/port:ret_check",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/util:resource_util",
    ] + select({
        "//mediapipe:android": [
            "//mediapipe/util/android/file/base",
        ],
        "//mediapipe:ios": [
            "//mediapipe/util/android/file/base",
        ],
        "//conditions:default": [
            "//mediapipe/framework/port:file_helpers",
        ],
    }),
    alwayslink = 1,
)

mediapipe_proto_library(
    name = "tensors_to_classification_calculator_proto",
    srcs = ["tensors_to_classification_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
    ],
)

cc_test(
    name = "tensors_to_classification_calculator_test",
    srcs = ["tensors_to_classification_calculator_test.cc"],
    data = ["testdata/labelmap.txt"],
    deps = [
        ":tensors_to_classification_calculator",
        ":tensors_to_classification_calculator_cc_proto",
        "//mediapipe/framework:calculator_cc_proto",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:calculator_runner",
        "//mediapipe/framework/formats:classification_cc_proto",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:gtest_main",
        "//mediapipe/framework/port:parse_text_proto",
        "@com_google_absl//absl/memory",
        "@com_google_googletest//:gtest_main",
    ],
)

cc_library(
    name = "image_to_tensor_calculator",
    srcs = ["image_to_tensor_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    features = ["-layering_check"],  # allow depending on image_to_tensor_calculator_gpu_deps
    visibility = ["//visibility:public"],
    deps = [
        ":image_to_tensor_calculator_cc_proto",
        ":image_to_tensor_converter",
        ":image_to_tensor_utils",
        "//mediapipe/framework/api2:node",
        "//mediapipe/framework/formats:image",
        "//mediapipe/framework/formats:image_frame",
        "//mediapipe/framework/formats:rect_cc_proto",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:ret_check",
        "//mediapipe/framework/port:status",
        "//mediapipe/framework/port:statusor",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:port",
        "//mediapipe/gpu:gpu_origin_cc_proto",
    ] + select({
        "//mediapipe/gpu:disable_gpu": [],
        "//conditions:default": [":image_to_tensor_calculator_gpu_deps"],
    }) + select({
        "//mediapipe/framework/port:disable_opencv": [],
        "//conditions:default": [":image_to_tensor_converter_opencv"],
    }),
    alwayslink = 1,
)

cc_library(
    name = "image_to_tensor_calculator_gpu_deps",
    deps = select({
        "//mediapipe:android": [
            ":image_to_tensor_converter_gl_buffer",
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:gpu_buffer",
        ],
        "//mediapipe:apple": [
            ":image_to_tensor_converter_metal",
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:MPPMetalHelper",
            "//mediapipe/gpu:gpu_buffer",
        ],
        "//conditions:default": [
            ":image_to_tensor_converter_gl_buffer",
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:gpu_buffer",
        ],
    }),
)

mediapipe_proto_library(
    name = "image_to_tensor_calculator_proto",
    srcs = ["image_to_tensor_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
        "//mediapipe/gpu:gpu_origin_proto",
    ],
)

cc_test(
    name = "image_to_tensor_calculator_test",
    srcs = ["image_to_tensor_calculator_test.cc"],
    data = [
        "testdata/image_to_tensor/input.jpg",
        "testdata/image_to_tensor/large_sub_rect.png",
        "testdata/image_to_tensor/large_sub_rect_border_zero.png",
        "testdata/image_to_tensor/large_sub_rect_keep_aspect.png",
        "testdata/image_to_tensor/large_sub_rect_keep_aspect_border_zero.png",
        "testdata/image_to_tensor/large_sub_rect_keep_aspect_with_rotation.png",
        "testdata/image_to_tensor/large_sub_rect_keep_aspect_with_rotation_border_zero.png",
        "testdata/image_to_tensor/medium_sub_rect_keep_aspect.png",
        "testdata/image_to_tensor/medium_sub_rect_keep_aspect_border_zero.png",
        "testdata/image_to_tensor/medium_sub_rect_keep_aspect_with_rotation.png",
        "testdata/image_to_tensor/medium_sub_rect_keep_aspect_with_rotation_border_zero.png",
        "testdata/image_to_tensor/medium_sub_rect_with_rotation.png",
        "testdata/image_to_tensor/medium_sub_rect_with_rotation_border_zero.png",
        "testdata/image_to_tensor/noop_except_range.png",
    ],
    deps = [
        ":image_to_tensor_calculator",
        ":image_to_tensor_converter",
        ":image_to_tensor_utils",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:calculator_runner",
        "//mediapipe/framework/deps:file_path",
        "//mediapipe/framework/formats:image",
        "//mediapipe/framework/formats:image_format_cc_proto",
        "//mediapipe/framework/formats:image_frame",
        "//mediapipe/framework/formats:image_frame_opencv",
        "//mediapipe/framework/formats:rect_cc_proto",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:gtest_main",
        "//mediapipe/framework/port:integral_types",
        "//mediapipe/framework/port:opencv_core",
        "//mediapipe/framework/port:opencv_imgcodecs",
        "//mediapipe/framework/port:opencv_imgproc",
        "//mediapipe/framework/port:parse_text_proto",
        "@com_google_absl//absl/flags:flag",
        "@com_google_absl//absl/memory",
        "@com_google_absl//absl/strings",
    ],
)

cc_library(
    name = "image_to_tensor_converter",
    hdrs = ["image_to_tensor_converter.h"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":image_to_tensor_utils",
        "//mediapipe/framework/formats:image",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:statusor",
    ],
)

cc_library(
    name = "image_to_tensor_converter_opencv",
    srcs = ["image_to_tensor_converter_opencv.cc"],
    hdrs = ["image_to_tensor_converter_opencv.h"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    deps = [
        ":image_to_tensor_converter",
        ":image_to_tensor_utils",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/formats:image",
        "//mediapipe/framework/formats:image_format_cc_proto",
        "//mediapipe/framework/formats:image_opencv",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:opencv_core",
        "//mediapipe/framework/port:opencv_imgproc",
        "//mediapipe/framework/port:status",
        "//mediapipe/framework/port:statusor",
    ],
)

cc_library(
    name = "image_to_tensor_converter_gl_buffer",
    srcs = ["image_to_tensor_converter_gl_buffer.cc"],
    hdrs = ["image_to_tensor_converter_gl_buffer.h"],
    deps = ["//mediapipe/framework:port"] + select({
        "//mediapipe:apple": [],
        "//conditions:default": [
            ":image_to_tensor_converter",
            ":image_to_tensor_converter_gl_utils",
            ":image_to_tensor_utils",
            "@com_google_absl//absl/strings",
            "//mediapipe/framework:calculator_framework",
            "//mediapipe/framework/formats:rect_cc_proto",
            "//mediapipe/framework/formats:tensor",
            "//mediapipe/framework/port:ret_check",
            "//mediapipe/framework/port:status",
            "//mediapipe/framework/port:statusor",
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/framework/formats:image",
            "//mediapipe/gpu:gpu_buffer_format",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/common:shape",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/common:types",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:command_queue",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_buffer",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_call",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_texture",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:request_gpu_info",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:variable",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl/converters:util",
        ],
    }),
)

cc_library(
    name = "image_to_tensor_converter_gl_texture",
    srcs = ["image_to_tensor_converter_gl_texture.cc"],
    hdrs = ["image_to_tensor_converter_gl_texture.h"],
    deps = ["//mediapipe/framework:port"] + select({
        "//mediapipe/gpu:disable_gpu": [],
        "//conditions:default": [
            ":image_to_tensor_converter",
            ":image_to_tensor_converter_gl_utils",
            ":image_to_tensor_utils",
            "@com_google_absl//absl/strings",
            "//mediapipe/framework:calculator_framework",
            "//mediapipe/framework/formats:tensor",
            "//mediapipe/framework/port:ret_check",
            "//mediapipe/framework/port:status",
            "//mediapipe/framework/port:statusor",
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:gl_simple_shaders",
            "//mediapipe/framework/formats:image",
            "//mediapipe/gpu:shader_util",
        ],
    }),
)

cc_library(
    name = "image_to_tensor_converter_gl_utils",
    srcs = ["image_to_tensor_converter_gl_utils.cc"],
    hdrs = ["image_to_tensor_converter_gl_utils.h"],
    deps = ["//mediapipe/framework:port"] + select({
        "//mediapipe/gpu:disable_gpu": [],
        "//conditions:default": [
            "//mediapipe/gpu:gl_base",
            "//mediapipe/gpu:gl_context",
            "//mediapipe/framework/port:status",
            "//mediapipe/framework/port:statusor",
        ],
    }),
)

cc_library(
    name = "image_to_tensor_converter_metal",
    srcs = ["image_to_tensor_converter_metal.cc"],
    hdrs = ["image_to_tensor_converter_metal.h"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    linkopts = select({
        "//mediapipe:apple": [
            "-framework CoreVideo",
            "-framework MetalKit",
        ],
        "//conditions:default": [],
    }),
    deps = ["//mediapipe/framework:port"] + select({
        "//mediapipe:apple": [
            ":image_to_tensor_converter",
            ":image_to_tensor_utils",
            "//mediapipe/gpu:MPPMetalHelper",
            "@com_google_absl//absl/strings",
            "//mediapipe/framework:calculator_framework",
            "//mediapipe/framework/formats:rect_cc_proto",
            "//mediapipe/framework/formats:tensor",
            "//mediapipe/framework/port:ret_check",
            "//mediapipe/framework/port:status",
            "//mediapipe/framework/port:statusor",
            "//mediapipe/framework/formats:image",
            "//mediapipe/gpu:gpu_buffer_format",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/common:shape",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/common:types",
        ],
        "//conditions:default": [],
    }),
)

cc_library(
    name = "image_to_tensor_utils",
    srcs = ["image_to_tensor_utils.cc"],
    hdrs = ["image_to_tensor_utils.h"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework/formats:rect_cc_proto",
        "//mediapipe/framework/port:ret_check",
        "//mediapipe/framework/port:statusor",
        "@com_google_absl//absl/types:optional",
    ],
)

cc_test(
    name = "image_to_tensor_utils_test",
    srcs = ["image_to_tensor_utils_test.cc"],
    deps = [
        ":image_to_tensor_utils",
        "//mediapipe/framework/formats:rect_cc_proto",
        "//mediapipe/framework/port:gtest_main",
    ],
)

# Copied from /mediapipe/calculators/tflite/BUILD
selects.config_setting_group(
    name = "gpu_inference_disabled",
    match_any = [
        "//mediapipe/gpu:disable_gpu",
    ],
)

mediapipe_proto_library(
    name = "tensors_to_segmentation_calculator_proto",
    srcs = ["tensors_to_segmentation_calculator.proto"],
    visibility = ["//visibility:public"],
    deps = [
        "//mediapipe/framework:calculator_options_proto",
        "//mediapipe/framework:calculator_proto",
        "//mediapipe/gpu:gpu_origin_proto",
    ],
)

cc_library(
    name = "tensors_to_segmentation_calculator",
    srcs = ["tensors_to_segmentation_calculator.cc"],
    copts = select({
        "//mediapipe:apple": [
            "-x objective-c++",
            "-fobjc-arc",  # enable reference-counting
        ],
        "//conditions:default": [],
    }),
    visibility = ["//visibility:public"],
    deps = [
        ":tensors_to_segmentation_calculator_cc_proto",
        "@com_google_absl//absl/strings:str_format",
        "@com_google_absl//absl/strings",
        "@com_google_absl//absl/types:span",
        "//mediapipe/framework/formats:image",
        "//mediapipe/framework/formats:image_frame",
        "//mediapipe/framework/formats:image_opencv",
        "//mediapipe/framework/formats:tensor",
        "//mediapipe/framework/port:opencv_imgproc",
        "//mediapipe/framework/port:ret_check",
        "//mediapipe/framework:calculator_context",
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework:port",
        "//mediapipe/util:resource_util",
        "@org_tensorflow//tensorflow/lite:framework",
        "//mediapipe/gpu:gpu_origin_cc_proto",
        "//mediapipe/framework/port:statusor",
    ] + selects.with_or({
        "//mediapipe/gpu:disable_gpu": [],
        "//conditions:default": [
            "//mediapipe/gpu:gl_calculator_helper",
            "//mediapipe/gpu:gl_simple_shaders",
            "//mediapipe/gpu:gpu_buffer",
            "//mediapipe/gpu:shader_util",
        ],
    }) + selects.with_or({
        ":gpu_inference_disabled": [],
        "//mediapipe:ios": [
            "//mediapipe/gpu:MPPMetalUtil",
            "//mediapipe/gpu:MPPMetalHelper",
        ],
        "//conditions:default": [
            "@org_tensorflow//tensorflow/lite/delegates/gpu:gl_delegate",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_program",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_shader",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl:gl_texture",
            "@org_tensorflow//tensorflow/lite/delegates/gpu/gl/converters:util",
        ],
    }),
    alwayslink = 1,
)
