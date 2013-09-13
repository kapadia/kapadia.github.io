---
layout: post
title:  "Emscripten: Pointers and Pointers"
date:   2013-09-13
categories: emscripten
---

[Emscripten](http://emscripten.org) is a [Mozilla Research](http://www.mozilla.org/research/) project that compiles LLVM bytecode to Javascript. Any language (e.g. C and C++) that compiles to LLVM as an intermediary may be ported to Javascript for use in the browser.  This is a brief write up on porting functions to JS and exposing their functionality.

### C to Javascript

Porting functions from C/C++ to Javascript is straight-forward. As an example, the C function below multiplies two numbers.

    extern "C" {
      int float_multiply(float x, float y) {
        return x * y;
      }
    }

Running the following on the command line will port this function to Javascript outputting the code to `multiply.js`.

    emcc multiply.cpp -o multiply.js -s EXPORTED_FUNCTIONS="['_float_multiply']"

Note that the `EXPORTED_FUNCTIONS` flag specifies an array of functions to export, each with a leading underscore.  If the function is not explicitly listed then Emscripten will consider it dead code, and strip it from the output.

After including `multiply.js` in a project, the function may be accessed by calling `cwrap` from `Module`, an object that Emscripten defines in its output. This function exposes `float_multiply` by specifying the function name, the return type, and an array of argument types.

    var float_multiply = Module.cwrap('float_multiply', 'number', ['number', 'number']);
    var result = float_multiply(5.2, 4.5);

Using `cwrap` (or `ccall`) is simple when functions use only primitive types as arguments, however many interesting functions use pointers as a means to operate over an array.

### Pointers

Using pointers with a ported function needs a bit more machinery. For example, the C function below is ported using `emcc` in the same way as `float_multiply`, but it's slightly more complex to use from Javascript.

    int float_multiply_array(float factor, float *arr, int length) {
      for (int i = 0; i <  length; i++) {
        arr[i] = factor * arr[i];
      }
      return 0;
    }

Emscripten's `cwrap` and `ccall` functions use only primitives to define the arguments of the ported function. Since an array (or typed array) is not a primitive type, it must be passed to the function by another mechanism – it must be passed as a number pointing to a block of memory internally managed by Emscripten.

    // Import function from Emscripten generated file
    float_multiply_array = Module.cwrap(
      'float_multiply_array', 'number', ['number', 'number', 'number']
    );
    
    // Create example data to test float_multiply_array
    var data = new Float32Array([1, 2, 3, 4, 5]);
    
    // Get data byte size, allocate memory on Emscripten heap, and get pointer
    var nDataBytes = data.length * data.BYTES_PER_ELEMENT;
    var dataPtr = Module._malloc(nDataBytes);
    
    // Copy data to Emscripten heap (directly accessed from Module.HEAPU8)
    var dataHeap = new Uint8Array(Module.HEAPU8.buffer, ptr, nDataBytes);
    dataHeap.set(new Uint8Array(data.buffer));
    
    // Call function and get result
    float_multiply_array(2, dataHeap.byteOffset, data.length);
    var result = new Float32Array(dataHeap.buffer, dataHeap.byteOffset, data.length);
    
    // Free memory
    Module._free(dataHeap.byteOffset);

First the data must be copied to Emscripten's memory heap. A number representing the data's byte offset on the heap is passed as an argument to `float_multiply_array`.

With these techniques, many C/C++ algorithms may be ported to Javascript, but thus far only pointers have been addressed. Many algorithms use pointers of pointers for multi-dimensional arrays (e.g. to represent an image or volume).

### Pointers of Pointers

Much of the above is described in the [Emscripten Wiki](https://github.com/kripken/emscripten/wiki/Interacting-with-code), but there isn't much discussion on pointers of pointers. Extrapolating from the above examples, consider this function, and note the use of the double pointer, `float **arr`.

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

Porting the function is done with `emcc` in the same way as the previous two functions, but using this function in Javascript requires slightly more code.

    // Import function from Emscripten generated file
    var float_multiply_matrix = Module.cwrap(
      'float_multiply_matrix', 'number', ['number', 'number', 'number']
    );
    
    // Create example data to test float_multiply_matrix
    var width = 10;
    var height = 5;
    var data = new Float32Array(width * height);
    for (var i = 0; i < width * height; i++) {
      data[i] = i;
    }
    
    // Get data byte size, allocate memory on Emscripten heap, and get pointer
    var nDataBytes = data.length * data.BYTES_PER_ELEMENT;
    var dataPtr = Module._malloc(nDataBytes);
    
    // Copy data to Emscripten heap
    var dataHeap = new Uint8Array(Module.HEAPU8.buffer, dataPtr, nDataBytes);
    dataHeap.set( new Uint8Array(data.buffer) );
    
    // Create array of pointers that reference each row in the data
    // Note the use of Uint32Array. The pointer is limited to 2147483648 bytes
    // or only 2GB of memory :(
    var pointers = new Uint32Array(height);
    for (var i = 0; i < pointers.length; i++) {
      pointers[i] = dataPtr + i * data.BYTES_PER_ELEMENT * width;
    }
    
    // Allocate bytes needed for the array of pointers
    var nPointerBytes = pointers.length * pointers.BYTES_PER_ELEMENT
    var pointerPtr = Module._malloc(nPointerBytes);
    
    // Copy array of pointers to Emscripten heap
    var pointerHeap = new Uint8Array(Module.HEAPU8.buffer, pointerPtr, nPointerBytes);
    pointerHeap.set( new Uint8Array(pointers.buffer) );
    
    // Call the function by passing a number pointing to the byte location of 
    // the array of pointers on the Emscripten heap.  Emscripten knows what to do!
    float_multiply_matrix(pointerHeap.byteOffset, height, width);
    
    var result = new Float32Array(dataHeap.buffer, dataHeap.byteOffset, data.length);
    
    // Free memory
    Module._free(pointerHeap.byteOffset);
    Module._free(dataHeap.byteOffset);

The technique in the above example requires a careful preparation of data, and an understanding of byte offsets on Emscripten's heap. Emscripten offers a lot in ways of memory management making it fun to all `malloc` and `free` from a Javascript.

The intent of using this tool is to port scientific algorithms to JS for use in web applications.  Next week is [.astronomy](http://dotastronomy.com/events/five/) – a conference bringing together astronomers and developers to share and implement unique ideas. One  idea is to port an astronomical source extraction algorithm to Javascript.  More on that soon.


[]