# Copyright (c) 2016 PaddlePaddle Authors. All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

if(APPLE)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=pessimizing-move")
endif(APPLE)


set(inference_deps paddle_inference_api paddle_fluid_api analysis pass ir_pass_manager naive_executor analysis_predictor ${GLOB_PASS_LIB})

if(WITH_GPU AND TENSORRT_FOUND)
    set(inference_deps ${inference_deps} tensorrt_engine tensorrt_converter)
endif()

cc_library(reset_tensor_array SRCS details/reset_tensor_array.cc DEPS lod_tensor scope)
cc_library(analysis_config SRCS analysis_config.cc DEPS lod_tensor paddle_pass_builder)
cc_library(paddle_pass_builder SRCS paddle_pass_builder.cc)
cc_library(analysis_predictor SRCS analysis_predictor.cc DEPS paddle_inference_api analysis naive_executor zero_copy_tensor reset_tensor_array analysis_config paddle_pass_builder ir_pass_manager)
cc_library(zero_copy_tensor SRCS details/zero_copy_tensor.cc DEPS scope lod_tensor enforce)
cc_library(zero_copy_tensor_dummy SRCS details/zero_copy_tensor_dummy.cc)
cc_library(paddle_inference_api SRCS api.cc api_impl.cc helper.cc DEPS
           lod_tensor scope paddle_pass_builder reset_tensor_array analysis_config
           analysis_config paddle_pass_builder zero_copy_tensor reset_tensor_array)

cc_test(test_paddle_inference_api
        SRCS api_tester.cc
        DEPS paddle_inference_api)

if(WITH_TESTING)
  inference_base_test(test_api_impl SRCS api_impl_tester.cc DEPS ${inference_deps}
                      ARGS --word2vec_dirname=${WORD2VEC_MODEL_DIR} --book_dirname=${PYTHON_TESTS_DIR}/book)
  set_tests_properties(test_api_impl PROPERTIES DEPENDS test_image_classification)
endif()
cc_test(test_analysis_predictor SRCS analysis_predictor_tester.cc DEPS analysis_predictor ${inference_deps}
        ARGS --dirname=${WORD2VEC_MODEL_DIR})

if (WITH_ANAKIN AND WITH_MKL) # only needed in CI
    # compile the libinference_anakin_api.a and anakin.so.
    cc_library(inference_anakin_api SRCS api.cc api_anakin_engine.cc DEPS anakin_shared anakin_saber mklml zero_copy_tensor_dummy)
    cc_library(inference_anakin_api_shared SHARED SRCS api.cc api_anakin_engine.cc DEPS anakin_shared anakin_saber zero_copy_tensor_dummy)
    function(anakin_target target_name)
      target_compile_options(${target_name} BEFORE PUBLIC ${ANAKIN_COMPILE_EXTRA_FLAGS})
    endfunction()
    anakin_target(inference_anakin_api)
    anakin_target(inference_anakin_api_shared)
endif()
