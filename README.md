OpenCL Notes
============

Notes taken while reading OpenCL in Action, by Matthew Scarpino.

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

> Note: this function is deprecated as of OpenCL 2.0, where it was replaced by `clCreateCommandQueueWithProperties`.

It needs a context, a device and a structure of properties. The error code is returned in `err`. The device must be associated with the context. The available properties are:
- `CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE`
- `CL_QUEUE_PROFILING_ENABLE`

As with contexts, the reference count of the returned command queue can be increased calling `clRetainCommandQueue` and decreased calling `clReleaseCommandQueue`.

In order to *use* queues there are some functions whose names start with `clEnqueue`, as

```C
cl_int clEnqueueTask(cl_command_queue queue, cl_kernel kernel, cl_uint num_events, const cl_event *wait_list, cl_event *event)
```

This function enqueues a kernel into a command queue. The `num_events` and `wait_list` parameters specify a list of events that must be finished before executing this kernel. The `event` parameter, if not `null`, will be populated with a unique event that can be used to identify this particular kernel execution.

# Transferring Data from Host to Device

In order to set the arguments of a kernel, one needs to set them one by one by calling the next function:

```
cl_int clSetKernelArg(cl_kernel kernel, cl_uint index, size_t size, const void *value)
```

The `index` is the index of the argument in the kernel, while the `value` is a pointer to the object that will be used by the device.

This pointer can point to:
- primitive data
- memory objects
- sampler objects
- it can be null, meaning that the device will just allocate memory.

## Memory Objects

Memory objects are represented by `cl_mem` structures. They can be buffer or image objects.

### Buffer Objects

In order to create a new buffer, use the following function: 

```C
cl_mem clCreateBuffer(cl_context context, cl_mem_flags options, size_t size, void *host_ptr, cl_int *error)
```

The `cl_mem` returned is a wrapper around the host memory pointed by `host_ptr`. The options parameter should be a combination of flags, with one flag describing how the device will access the memory (one of the mutually exclusive `CL_MEM_READ_WRITE`, `CL_MEM_WRITE_ONLY` and `CL_MEM_READ_ONLY`) and a combination of flags describing how the object is allocated in the host (possible flags are `CL_MEM_USE_HOST_PTR` `CL_MEM_COPY_HOST_PTR` and `CL_MEM_ALLOC_HOST_PTR`).

The host does not allocate memory for a write only buffer, so the `host_ptr` may be `null` in those cases.

#### Subbuffers

A subbuffer is just a view of an existing buffer, and can be created with the function:

```C
clCreateSubBuffer(cl_mem buffer, cl_mem_flags flags, cl_buffer_create_type type, const void *info, cl_int *error)
```

This creates a subbuffer of the provided `buffer`, whose beginning position and size is specified by the `info` argument, which expects a pointer to a struct of type

```C
typedef struct _cl_buffer_region {
	size_t origin;
	size_t size;
} cl_buffer_region;
```

The available flags may be the same as the ones used when creating a buffer and the type should be always equal to `CL_BUFFER_CREATE_TYPE_REGION`.

We can retrieve information about buffer objects by using

```C
cl_int clGetMemObjectInfo(cl_mem object, cl_mem_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
```

The possible parameters to retrieve are:
- `CL_MEM_TYPE`
- `CL_MEM_FLAGS`
- `CL_MEM_HOST_PTR`
- `CL_MEM_SIZE`
- `CL_MEM_CONTEXT`
- `CL_MEM_ASSOCIATED_MEMOBJECT`
- `CL_MEM_OFFSET`
- `CL_MEM_REFERENCE_COUNT`
- `CL_MEM_D3D10_RESOURCE_KHR`

### Image Objects

Image objects are `cl_mem` structures created very much in the same way that buffers are created, but using the following functions:

```C
cl_mem clCreateImage2D (cl_context context, cl_mem_flags opts, const cl_image_format *format, size_t width, size_t height, size_t row_pitch, void *data, cl_int *error)
cl_mem clCreateImage3D (cl_context context, cl_mem_flags opts, const cl_image_format *format, size_t width, size_t height, size_t depth, size_t row_pitch, size_t slice_pitch, void *data, cl_int *error)
```

We can allocate 2D and 3D images, which are just 2D or 3D structures of data. The arguments `context` and `opts` behave in the same way than in the case of buffers. The `width`, `height` and `depth` arguments set the structure of the data (in pixels), and the `*_pitch` arguments set the pitch of the image; i.e., the number of bytes per row (and per slice in the 3D case). The pitch arguments will usualy be 0, which tells OpenCL that the number of bytes per row can be computed as the number of pixels in a row multiplied by the number of bytes per pixel. This may not be the case, e.g., when the rows are padded to align the memory.

Regarding the third argument, `format`, it is a pointer to a structure like

```C
typedef struct _cl_image_format {
	cl_channel_order image_channel_order;
	cl_channel_type image_channel_data_type;
} cl_image_format;
```

It sets the order in which the channels RGBA (if present) are stored (the usual RGBA is `CL_RGBA`) and the type and number of bits per channel (the usual 8-bit image is `CL_UNSIGNED_INT8`).

As before, one can get information about an image object by using the function

```C
cl_int clGetImageInfo(cl_mem image,	cl_image_info param_name, size_t param_value_size, void *param_value, size_t *param_value_size_ret)
```

The possible parameters to retrieve are:
- `CL_IMAGE_FORMAT`
- `CL_IMAGE_ELEMENT_SIZE`
- `CL_IMAGE_ROW_PITCH`
- `CL_IMAGE_SLICE_PITCH`
- `CL_IMAGE_WIDTH`
- `CL_IMAGE_HEIGHT`
- `CL_IMAGE_DEPTH`
- `CL_IMAGE_D3D10_SUBRESOURCE_KHR`

### Transferring Data

We know how to transfer data from the device to the host using memory/image objects and setting the kernel parameters (when this transfer is done depends on the specific implementation, but the standard ensures that the data is accesible by the device when the kernel is executed, so the copy usually happens as soon as the kernel is set).

We need to enqueue reads and writes using the functions `clEnqueue[X][Y]` where `[X]` may be `Read` or `Write` and `[Y]` is one of `Buffer`, `Image` or `BufferRect`. All of them have a `blocking` parameter that sets the blocking status of the function: if it is set to `CL_TRUE`, the function will block until the operation is done; if it is `CL_FALSE`, it will not. We use `origin[3]` and `region[3]` in the `Image` functions to specicfy the origin pixel and the dimensions of the square (or cube) that we are interested in. If we want to do the same thing with a non-image object, we can use the `BufferRect` functions, which use `buffer_origin[3]`, `host_origin[3]` and `region[3]` in the same way as before.

### Mapping data

As in C, one can map buffer data to the host memory address space. The main idea is to map buffer/image objects to the host and use common memory primitives to read/write into those objects instead of using the `clEnqueue[X][Y]` functions. In order to map and unmap regions of buffers and images, the functions to use are:

```C
void* clEnqueueMapBuffer(cl_command_queue queue, cl_mem buffer, cl_bool blocking, cl_map_flags map_flags, size_t offset, size_t data_size, cl_uint num_events, const cl_event *wait_list, cl_event *event, cl_int *errcode_ret)
void* clEnqueueMapImage(cl_command_queue queue, cl_mem image, cl_bool blocking, cl_map_flags map_flags, const size_t origin[3], const size_t region[3], size_t *row_pitch, size_t *slice_pitch, cl_uint num_events, const cl_event *wait_list, cl_event *event, cl_int *errcode_ret)
int clEnqueueUnmapMemObject(cl_command_queue queue, cl_mem memobj, void *mapped_ptr, cl_uint num_events, const cl_event *wait_list, cl_event *event)
```

The `void*` pointer returned by the map functions identify the start of the memory that maps to the object and is the parameter that the `unmap` function expects. The flags in the `map` functions can be any combination of the flags `CL_MAP_READ` and `CL_MAP_WRITE`.

The process to use mapped memory is:
1. map the memory
2. transfer the data using some primitive like `memcpy`
3. unmap the memory

### Copying Data between Memory Objects

We now know how to copy memory from host to device and from device to host, using explicit read/write functions or memory mappings. But we can also copy data between memory objects. This is accomplished with the functions `clEnqueueCopy[X]` where `[X]` may be `Buffer` or `Image` to copy data between objects of the same type, `BufferToImage` or `ImageToBuffer` to copy data between objects of different type, or `BufferRect` to copy rectangular regions of data between buffers.
