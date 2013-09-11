---
layout: post
title:  "Emscripten: Pointers and Pointers"
date:   2013-09-13
categories: emscripten
---

[Emscripten](http://emscripten.org) is a Mozilla Research project that compiles LLVM byte code to Javascript.  This means that any code that is compiled using LLVM as an intermediary may be compiled to Javascript (e.g. C and C++).  This is a great project because so many algorithms written in C and C++ may be exposed to Javascript and the browser.

### C to Javascript

Porting functions from C/C++ to Javascript is straight-forward.  The C function below multiplies two numbers.

    extern "C" {
      int float_multiply(float x, float y) {
        return x * y;
      }
    }

The following will port this function to Javascript outputting the code to `multiply.js`.

    emcc multiply.cpp -o multiply.js -s EXPORTED_FUNCTIONS="['_float_multiply']"

Note that the `EXPORTED_FUNCTIONS` flag specifies an array of functions to export, each with a leading underscore.  If functions are not explicitly specified then Emscripten will consider the functions to be dead code, and strip them from the output.

After including `multiply.js` in a project, the function may be accessed by calling `cwrap` from `Module`, an object that Emscripten defines in the output. This function exposes `float_multiply` by specifying the function name, the return type, and an array of argument types.

    var float_multiply = Module.cwrap('float_multiply', 'number', ['number', 'number']);
    var result = float_multiply(5, 4);

### Pointers

Working with pointers requires a bit more machinery. A C function with the following definition is slightly more complex to port.

    int float_multiply_array(float factor, float *arr, int length) {
      for (int i = 0; i <  length; i++) {
        arr[i] = factor * arr[i];
      }
      return 0;
    }

Emscripten's `cwrap` or `ccall` function uses only primitives to define the arguments of the ported function. Since an array (or typed array) is not a primitive type, it must be passed to the ported function by another mechanism – it must be passed as a pointer to a block of memory managed by Emscripten.

Porting the function above is done is the same way as `float_multiply`, but the functionality is exposed differently from Javascript.

    // Import function from Emscripten generated file
    float_multiply_array = Module.cwrap(
      'float_multiply_array', 'number', ['number', 'number', 'number']
    );
    
    // Prepare data to operate on
    var data = new Float32Array([1, 2, 3, 4, 5]);
    
    // Get data byte size, allocate on Emscripten heap, and get pointer
    var dataBytes = data.length * data.BYTES_PER_ELEMENT;
    var dataPtr = Module._malloc(dataBytes);
    
    // Copy data to Emscripten heap
    var dataHeap = new Uint8Array(Module.HEAPU8.buffer, ptr, dataBytes);
    dataHeap.set(new Uint8Array(data.buffer));
    
    // Call function and save result
    float_multiply_array(2, dataHeap.byteOffset, data.length);
    var result = new Float32Array(dataHeap.buffer, dataHeap.byteOffset, data.length);
    
    // Free memory
    Module._free(dataHeap.byteOffset);

Emscripten requires that the data be copied to its memory heap, and the pointer is passed to `float_multiply_array`. At this point, any function with a pointer as an argument may easily be ported to Javascript, however, many of the interesting algorithms utilize pointers of pointers. Whoa.

### Pointers of Pointers

Much of the above is described in the [Emscripten Wiki](https://github.com/kripken/emscripten/wiki/Interacting-with-code), but there isn't much discussion on pointers of pointers. Extrapolating the above examples further, consider this function:

    int float_multiply_matrix(float **arr, int ilength, int jlength) {
      float *row;
      for (int i = 0; i < ilength; i++) {
        row = arr[i];
        for (int j = 0; j < jlength; j++) {
          row[j] = 2.0 * row[j];
        }
      }
      return 0;
    }

This function is ported exactly as the previous two examples, but requires extra work when calling it from Javascript.


    // Import function from Emscripten generated file
    var float_multiply_matrix = Module.cwrap(
      'float_multiply_matrix', 'number', ['number', 'number', 'number']
    );
    
    // Create test data
    var width = 10;
    var height = 5;
    var data = new Float32Array(width * height);
    for (var i = 0; i < width * height; i++) {
      data[i] = i;
    }
    
    // Allocate bytes needed for data
    var dataBytes = data.length * data.BYTES_PER_ELEMENT;
    var dataPtr = Module._malloc(dataBytes);
    
    // Copy data to Emscripten memory
    var dataHeap = new Uint8Array(Module.HEAPU8.buffer, dataPtr, dataBytes);
    dataHeap.set( new Uint8Array(data.buffer) );
    
    // Create array of pointers that reference each row in the data
    var pointers = new Uint32Array(height);
    for (var i = 0; i < pointers.length; i++) {
      pointers[i] = dataPtr + i * data.BYTES_PER_ELEMENT * width;
    }
    
    // Allocate bytes needed for array of pointers
    var nPointerBytes = pointers.length * pointers.BYTES_PER_ELEMENT
    var pointerPtr = Module._malloc(nPointerBytes);
    
    // Copy array of pointers to Emscripten memory
    var pointerHeap = new Uint8Array(Module.HEAPU8.buffer, pointerPtr, nPointerBytes);
    pointerHeap.set( new Uint8Array(pointers.buffer) );
    
    // Call the function by passing a pointer to an array of pointers
    float_multiply_matrix(pointerHeap.byteOffset, height, width);
    
    var result = new Float32Array(dataHeap.buffer, dataHeap.byteOffset, data.length);