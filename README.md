OpenCL Notes
============

# Host Data Structures

- There are six data structures that need to be configured by the host:
	- Platforms: an OpenCL implementation. In my case, I have the `Intel(R) OpenCL HD Graphics` platform.
	- Devices: the devices that the platforms make use of. In my case, I have an `Intel(R) Gen9 HD Graphics NEO` GPU.
	- Contexts: a subset of selected devices controlled by the *same* platform. Contexts make it possible to create command queues, the structures that allow hosts to send kernels to devices.
	- Programs: basically a set of kernels. They can be compiled and used later to define the kernels.
	- Kernels: functions in programs that can be deployed to a device.
	- Command queues: a one-way communication between a host and a single device: the host queues operations (kernels, data transfers, data copies...) for the device to execute.

### Platforms

1. Allocate memory to create a `cl_platform_id` structure.
2. Call `clGetPlatformIDs` to populate the previous structure. The signature of this function is:
```C
cl_int clGetPlatformIDs(cl_uint num_entries, cl_platform_id* platforms, cl_uint *num_platforms)
```
meaning that you want to detect a maximum of `num_entries` platforms, which will be placed in the memory pointed by `platforms`. The final number of platforms installed will be returned in the memory pointed by `num_platforms`. So I guess that the number of platforms returned is `min(num_entries, num_platforms)`. Or maybe `num_platforms` is always upper bounded by `num_entries`. I need to check this.
One cool thing to do here is to call `clGetPlatformIDs` with `platforms = null`, so we can just know the number of platforms available. With this number, we can allocate as much `cl_platform_id` structures as we need, calling the function again to populate these structures.
3. If one needs to know anything else about the platform, one can use the function `clGetPlatformInfo`, whose signature is
```C
cl_int clGetPlatformInfo(cl_platform_id platform, cl_platform_info parameter, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
```
This means that I want to know the value of the parameter `parameter`: it will be returned in the memory pointed by `param_value`, and the size of the returned char array will be stored in `param_value_size_ret`. Every parameter value returned here is an array of chars.
As before, if you want to know in advance the length of the returned value, call `clGetPlatformInfo` with the name of the parameter and `param_value = null` so you just retrieve the length in `param_value_size_ret`. Then, allocate as many memory as told by this returned size and call the function again with your newly allocated `param_value`.

### Devices

1. Allocate memory to create a `cl_device_id` structure.
2. Call `clGetDeviceIDs` to populate the previous structure. The signature of the function is:

```C
cl_int clGetDeviceIDs(cl_platform_id platform, cl_device_type device_type, 	cl_uint num_entries, cl_device_id* devices, cl_uint *num_devices)
```

The meaning of the parameters is as before, but now it also expects the platform ID populated before and the type of the device (CPU, GPU, accelerator...). As before, one can leave any of `devices` or `num_devices` null to just retrieve the desired information.
3. As before, more information can be queried using `clGetDeviceInfo`:

```C
cl_int clGetDeviceInfo(cl_devide_id device, cl_device_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
```

This works as before, but not every returned value is a char array, and the number of different parameters is huge.

### Contexts

To create contexts, there are two alternatives:
- specify the devices that the context should control.
- specify the type of devices that the context should control: i.e., the context will pick the devices whose type complaints with the type passed.

The signature of both functions is similar:

```C
cl_context clCreateContext(const cl_context_properties *properties,  cl_uint num_devices, const cl_device_id *devices, (void CL_CALLBACK *notify_func)(...), void *user_data, cl_int *error)
cl_context clCreateContextFromType(const cl_context_properties *properties,  cl_device_type *device_type, (void CL_CALLBACK *notify_func)(...), void *user_data, cl_int *error)
```

As you can guess, you don't need to allocate memory for the context structure, as it is directly returned by the function.

The common parameters are:
- `properties`: an array of names and values of properties; the last element should be 0. Something like `cl_context_properties context_props[] = {CL_CONTEXT_PLATFORM, (cl_context_properties)platforms[0], CL_GL_CONTEXT_KHR, (cl_context_properties)glXGetCurrentContext(), CL_GLX_DISPLAY_KHR, (cl_context_properties)glXGetCurrentDisplay(), 0};`
- `notify_func`: a callback function that will be called whenever an error occurs.
- `user_data`: a set of user-defined data with whichever shape the user decides. This data is accessed by the callback function when it is called.
- `error`: an integer that will be populated with the return code of the function when it finishes. If the context is created successfully, then the error will be 0.

The `num_devices` and `devices` parameters of the first function define the specific devices the context will be built for.
The `device_type` of the second function specifies the type of devices that will be selected for this context.

I guess that the `properties` must hold at least the platform to which this context is bound, but I'm not that sure. I need to check this.

As before, one can retrieve more information about a context by using

```C
cl_int clGetContextInfo(cl_context context, cl_context_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
```

As before, set the `param_value` pointer to `null` if you just want to know the size you need to allocate for it.

Contexts have reference counts; i.e., the structure is only deallocated when its reference count reaches zero. To increase the counter, use `clRetainContext`; to decrease it, use `clReleaseContext`. The scope in which a context is created should always call `clReleaseContext` before the scope finishes. Any function that uses a pre-existing context must use it preceded by a call to `clRetainContext` and must finish with a call to `clReleaseContext`.

### Programs

#### Create

A program cannot be read directly from a file handle, but must be read into a buffer first. There are two alternatives to load a program:

```C
cl_program clCreateProgramWithSource(cl_context context, cl_uint src_num, const char **src_strings, const size_t *src_sizes, cl_int *err_code)
cl_program clCreateProgramWithBinary(cl_context context, cl_uint num_devices, const cl_device_id *devices, const size_t *bin_sizes, const unsigned char **bins, cl_int *bin_sattus, cl_int *err_code)
```

`clCreateProgramWithSource` expects the code in text form, with `src_num` files passed in `src_strings`, each one of them with the corresponding length stored in `src_sizes`.

I don't know much about the second option yet.

#### Compile

In order to compile the program, one may use the following function:

```C
clBuildProgram(cl_program program, cl_uint num_devices, const cl_device_id *devices, const char *options, (void CL_CALLBACK *notify_func)(...), void *user_data)
```

The `program` parameter is the structure returned by `clCreateProgramWithWhatever`. The `num_devices` and `devices` specify the devices for which the program will be built. The `options` parameter specifies the compilation options. As usual, the `notify_func` is a callback function that will be called whenever the build finishes: if it is not null, `clBuildProgram` will return immediately; if it is null, it will block until the build finishes. And `user_data` is the data passed to `notify_func`.

This function does not give much information about the build process. If we need to know something about it, we can use the following functions:

```C
cl_int clGetContextInfo(cl_program program, cl_program_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
cl_int clGetProgramBuildInfo(cl_program program, cl_device_id device, cl_program_build_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)

```

The first one works exactly as the `clGetWhateverInfo` that we have already seen. The second one only adds an identifier of the device for which the program was built. The possible values for `cl_program_info param_name` are `CL_PROGRAM_BUILD_STATUS`, `CL_PROGRAM_BUILD_OPTIONS` and `CL_PROGRAM_BUILD_LOG`, which are self-explanatory. As usual, we can set `param_value` to `null` so that we retrieve the size of the returned value, allocate the needed memory and call the function again with the new pointer.

### Kernels

Once a program has been created and built, the kernels can be retrieved: either all of the kernels defined in a program or a specific one identified by its name:

```C
cl_int clCreateKernelsInProgram(cl_program program, cl_uint num_kernels, cl_kernel *kernels, cl_uint *num_kernels_ret);
cl_kernel clCreateKernel(cl_program program, const char *kernel_name, cl_int *error)
```

Both need to receive a `cl_program` structure. The first retrieves all the kernels, that are stored in `kernels` (the return value is an error code); the second identifies the kernel by its name and is returned by the function (the error code is returned using the pointer `error`).

As with previous structures, if you want to retrieve information about the kernel, you can use the function

```C
clGetKernelInfo(cl_kernel kernel, cl_kernel_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
```

The possible values for `param_name` are:
- `CL_KERNEL_FUNCTION_NAME`
- `CL_KERNEL_NUM_ARGS`
- `CL_KERNEL_REFERENCE_COUNT`
- `CL_KERNEL_CONTEXT`
- `CL_KERNEL_PROGRAM`

### Command queues

No retrieval of information, no alternatives to create queues in different ways. Just *one* function:

```C
cl_command_queue clCreateCommandQueue(cl_context context, cl_device_id device, cl_command_queue_properties properties, cl_int *err)
```

It needs a context, a device and a structure of properties. The error code is returned in `err`. The device must be associated with the context. The available properties are:
- `CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE`
- `CL_QUEUE_PROFILING_ENABLE`

As with contexts, the reference count of the returned command queue can be increased calling `clRetainCommandQueue` and decreased calling `clReleaseCommandQueue`.

In order to *use* queues there are some functions whose names start with `clEnqueue`, as

```C
cl_int clEnqueueTask(cl_command_queue queue, cl_kernel kernel, cl_uint num_events, const cl_event *wait_list, cl_event *event)
```

This function enqueues a kernel into a command queue. The `num_events` and `wait_list` parameters specify a list of events that must be finished before executing this kernel. The `event` parameter, if not `null`, will be populated with a unique event that can be used to identify this particular kernel execution.
