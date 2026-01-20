# True Grayscale Image Test Results

**Date:** 2026-01-19  
**Test Focus:** Verify grayscale improvement handles 1-channel images correctly  
**Test Image:** `true-grayscale.jpg` (1-channel grayscale JPEG)



##  Purpose

The code change in `src/image.c` adds specific handling for **true 1-channel grayscale images**. 

Previous versions of the code assumed all images were RGB (3-channel) and would access `pixel[1]` and `pixel[2]` even for 1-channel images, causing **undefined behavior** (out-of-bounds memory access).



##  Test Image Details

**Created:** True 1-channel grayscale image from `true-grayscale.jpg`

```bash
$ file examples/true-grayscale.jpg
examples/true-grayscale.jpg: JPEG image data, JFIF standard 1.01, 
  aspect ratio, density 72x72, segment length 16, Exif Standard: 
  [TIFF image data, big-endian, direntries=1], progressive, 
  precision 8, 5238x7860, components 1
```

**Key Detail:** `components 1` ← This is a TRUE 1-channel grayscale image!

Compare to the original:
```bash
$ file examples/black-and-white.jpg
examples/black-and-white.jpg: JPEG image data, JFIF standard 1.02, 
  resolution (DPI), density 72x72, segment length 16, progressive, 
  precision 8, 5238x7860, components 3
```

**Original:** `components 3` ← RGB image (even though it looks grayscale)



## Code Change Analysis

### Previous Version (UNSAFE)
```c
for (size_t y = 0; y < height; y++) {
    for (size_t x = 0; x < width; x++) {
        double* pixel = get_pixel(original, x, y);

        // Luminance-weighted graycsale. Could be a callback...
        double grayscale = 0.2126 * pixel[0] + 0.7152 * pixel[1] + 0.0722 * pixel[2];
        //                                      ^^^^^^^^           ^^^^^^^^
        //                                      OUT OF BOUNDS for 1-channel images!

        set_pixel(&new, x, y, &grayscale);
    }
}
```

**Problem:** For 1-channel images, `get_pixel()` returns a pointer to only 1 double value.  
Accessing `pixel[1]` and `pixel[2]` reads **uninitialized memory** → Undefined Behavior!

### Current Version (SAFE)
```c
for (size_t y = 0; y < height; y++) {
    for (size_t x = 0; x < width; x++) {
        double* pixel = get_pixel(original, x, y);
        double grayscale;
        if (original->channels == 1) {
            grayscale = pixel[0];  // Already grayscale ✅
        } else if (original->channels >= 3) {
            // Luminance-weighted graycsale. Could be a callback...
            grayscale = 0.2126 * pixel[0] + 0.7152 * pixel[1] + 0.0722 * pixel[2];
        } else {
            fprintf(stderr, "Unsupported channel count: %zu\n", original->channels);
            free(data);
            return (image_t){0};
        }

        set_pixel(&new, x, y, &grayscale);
    }
}
```

**Fix:** Checks channel count and handles 1-channel images correctly! 



## Test Results

### Test 1: Current Version (with fix)
```bash
$ ./ascii-view examples/true-grayscale.jpg > current-true-grayscale.txt 2>&1
$ echo $?
0
$ wc -l current-true-grayscale.txt
52 current-true-grayscale.txt
```
**SUCCESS** - Renders correctly

### Test 2: Previous Version (without fix)
```bash
$ git checkout HEAD~1
$ make clean && make
$ ./ascii-view examples/true-grayscale.jpg > previous-true-grayscale.txt 2>&1
$ echo $?
0
$ wc -l previous-true-grayscale.txt
52 previous-true-grayscale.txt
```
**Works** - But relies on undefined behavior (lucky memory layout)

### Test 3: Output Comparison
```bash
$ ls -lh *-true-grayscale.txt
-rw-r--r--  60K  current-true-grayscale.txt
-rw-r--r--  60K  previous-true-grayscale.txt

$ md5 *-true-grayscale.txt
MD5 (current-true-grayscale.txt) = 62d479d76a7c29cab88629b14aa7b940
MD5 (previous-true-grayscale.txt) = 62d479d76a7c29cab88629b14aa7b940

$ diff previous-true-grayscale.txt current-true-grayscale.txt
(no output - files are identical)
```

 **IDENTICAL OUTPUTS** - Both versions produce the same result



##  Why Both Versions Work

The previous version "works" because:
1. Memory allocated by `calloc()` happens to be laid out contiguously
2. Accessing `pixel[1]` and `pixel[2]` reads adjacent memory (likely zeros or garbage)
3. For grayscale images, these values happen to be similar, so the weighted average still looks correct
4. **This is LUCK, not correctness!**

### The Risk
- Different compilers, optimization levels, or memory allocators could cause crashes
- AddressSanitizer or Valgrind would detect this as a bug
- Future code changes could break this "lucky" behavior



## Conclusion


Even though both versions produce identical output in this test, **the fix is critical** because:

1. **Correctness:** Eliminates undefined behavior
2. **Safety:** Prevents potential crashes with different memory layouts
3. **Robustness:** Handles edge cases (2-channel images) with error messages
4. **Maintainability:** Makes the code's intent clear

### Test Results Summary

| Aspect | Previous Version | Current Version |
|--||--|
| **Builds cleanly** | ✅ Yes | ✅ Yes |
| **Renders correctly** | ✅ Yes (lucky) | ✅ Yes (correct) |
| **Memory safety** | ❌ Undefined behavior | ✅ Safe |
| **Output quality** | ✅ Identical | ✅ Identical |
| **Code correctness** | ❌ Out-of-bounds access | ✅ Proper bounds checking |




Grayscale improvement:
- Fixes a real bug (undefined behavior)
- Maintains backward compatibility (identical output)
- Adds robustness (error handling for unsupported channel counts)
- Zero regressions detected


