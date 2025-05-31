Program Overview

This C program renders three rotating 3D cubes of different sizes in an ASCII-art style within a terminal window. By computing the projection of 3D cube surfaces onto a 2D plane and applying simple shading via different ASCII characters, the program creates the visual effect of continuously spinning cubes. The application relies on classical 3D-to-2D projection math combined with a rudimentary z-buffer algorithm to ensure correct rendering of overlapping surfaces.

Platform Compatibility

    The program includes a conditional compilation block to support both POSIX-compliant systems (e.g., Linux, macOS) and Windows. On POSIX systems, the standard usleep() function is available, so no additional code is required.

    On Windows (_WIN32 defined), a custom usleep() function is provided. This implementation uses the Win32 API’s CreateWaitableTimer, SetWaitableTimer, and WaitForSingleObject calls to achieve microsecond-resolution sleeps. The function converts the requested microsecond delay into the 100-nanosecond units required by the Windows timer and then waits for that duration.

    #ifndef _WIN32
      // On non-Windows platforms, <unistd.h> would normally provide usleep()
    #else
      #include <windows.h>
      void usleep(__int64 usec) {
        HANDLE timer;
        LARGE_INTEGER ft;
        ft.QuadPart = -(10 * usec); // Negative value indicates relative time in 100ns units
        timer = CreateWaitableTimer(NULL, TRUE, NULL);
        SetWaitableTimer(timer, &ft, 0, NULL, NULL, 0);
        WaitForSingleObject(timer, INFINITE);
        CloseHandle(timer);
      }
    #endif

Global Variables and Configuration

    Rotation Angles

    float A, B, C;

These three floats represent the rotation angles around the X, Y, and Z axes, respectively. They are initially zero (implicitly, since global variables are zero-initialized) and incremented each frame to create continuous rotation.

Cube and Screen Dimensions

float cubeWidth = 20;
int width = 160, height = 44;

    cubeWidth sets the half-size of a cube along each axis (initially 20 units).

    width and height define the dimensions of the ASCII “screen” (160 columns by 44 rows).

Z-Buffer and Character Buffer

    float zBuffer[160 * 44];
    char buffer[160 * 44];

    zBuffer stores, for each screen pixel, the reciprocal of the cube-to-camera distance (1/z). By comparing these values, the program can determine which surface is closest and should overwrite previous characters.

    buffer holds the ASCII character to be displayed at each pixel, based on the face of the cube that is visible.

Rendering Parameters

int backgroundASCIICode = '.';
int distanceFromCam = 100;
float horizontalOffset;
float K1 = 40;
float incrementSpeed = 0.6;

    backgroundASCIICode is the character used to fill the screen before any cube points are drawn (a dot).

    distanceFromCam translates all cube vertices along the Z axis so that they lie in front of the “camera” instead of behind the origin.

    horizontalOffset shifts the entire cube horizontally (used to position multiple cubes side by side).

    K1 is a scaling constant for projection, controlling how large the cube appears once projected onto 2D.

    incrementSpeed determines the step size when sampling points on each cube’s surface; smaller values produce denser sampling and smoother surfaces at the cost of performance.

Temporary Coordinate Variables

    float x, y, z, ooz;
    int xp, yp, idx;

    These variables are used within each rendering call to hold computed 3D-to-2D projected coordinates (x, y, z, ooz) and their integer equivalents (xp, yp) for indexing into the buffers. idx is the one-dimensional index into the flattened 160×44 array.

3D Rotation and Projection Functions

The program uses three helper functions—calculateX(), calculateY(), and calculateZ()—to perform a composite rotation of a point (i, j, k) around the three principal axes and then extract its final X, Y, or Z coordinate. The rotation is applied in the order: rotate by angle A around the X-axis, then by B around the Y-axis, then by C around the Z-axis. In each function:

    i corresponds to the original X coordinate of a vertex on the cube face.

    j corresponds to the original Y coordinate.

    k corresponds to the original Z coordinate.

For example, the code for computing the rotated X component is:

    float calculateX(int i, int j, int k) {
      return j * sin(A) * sin(B) * cos(C)
           - k * cos(A) * sin(B) * cos(C)
           + j * cos(A) * sin(C)
           + k * sin(A) * sin(C)
           + i * cos(B) * cos(C);
    }

Similarly, calculateY() and calculateZ() apply the same rotation transformations but extract the Y and Z components, respectively. Once these rotated coordinates are obtained, the point’s Z is offset by distanceFromCam to ensure positive depth.

Surface Sampling and Rendering (calculateForSurface)
    
    void calculateForSurface(float cubeX, float cubeY, float cubeZ, int ch) {
      // 1. Rotate and translate:
      x = calculateX(cubeX, cubeY, cubeZ);
      y = calculateY(cubeX, cubeY, cubeZ);
      z = calculateZ(cubeX, cubeY, cubeZ) + distanceFromCam;
    
      // 2. Compute “one over Z” (used for simple depth buffering):
      ooz = 1.0f / z;
    
      // 3. Project the 3D point onto the 2D screen:
      xp = (int)(width / 2 + horizontalOffset + K1 * ooz * x * 2);
      yp = (int)(height / 2 + K1 * ooz * y);
    
      // 4. Determine the index in the flattened buffers:
      idx = xp + yp * width;
      if (idx >= 0 && idx < width * height) {
        // 5. Depth test: if this point is closer, update the z-buffer and character buffer
        if (ooz > zBuffer[idx]) {
          zBuffer[idx] = ooz;
          buffer[idx] = ch;
        }
      }
    }

    Rotation & Translation (Steps 1–2): Call the three calculateX/Y/Z functions to get the rotated coordinates, then shift the resulting Z by distanceFromCam. The variable ooz stores 1/z, which serves as a simple depth metric (larger ooz indicates a closer surface).

    Projection (Step 3): Convert the 3D point (x, y, z) into 2D screen coordinates (xp, yp). The center of the screen is (width/2, height/2). Multiplying by K1 * ooz executes a rudimentary perspective projection. The factor of 2 by which x is multiplied simply stretches horizontally to maintain aspect ratio.

    Buffering & Depth Testing (Steps 4–5): Compute the linear index idx for the one-dimensional buffer and zBuffer arrays. If the newly computed depth (ooz) exceeds the stored zBuffer[idx] (meaning this point is in front of whatever has been drawn there), update both zBuffer[idx] and buffer[idx] with the new depth and the ASCII character ch, respectively.

Each call to calculateForSurface thus draws (or overwrites) a single pixel in the ASCII output, using the character ch to indicate which face of the cube that point belongs to.

Main Loop and Cube Construction

The main() function enters an infinite loop in which the following steps occur each frame:

    Clear Buffers

memset(buffer, backgroundASCIICode, width * height);
memset(zBuffer, 0, width * height * sizeof(float));

    The entire display buffer is filled with the background character '.'.

    The z-buffer is reset to zero, so that any newly drawn point (with ooz > 0) will pass the depth test.

Draw Three Cubes of Different Sizes
The program draws three cubes, each with a different cubeWidth (20, 10, and 5) and a different horizontalOffset (−2×size, +1×size, +8×size, respectively). For each cube:

    Nested for loops iterate over cubeX and cubeY within the range [-cubeWidth, +cubeWidth) in steps of incrementSpeed (0.6).

    For each (cubeX, cubeY) pair, the six faces of the cube are sampled by holding one coordinate constant at ±cubeWidth and varying the third coordinate. Each face uses a unique ASCII character:

        Face at Z = −cubeWidth: '@'

        Face at X = +cubeWidth: '$'

        Face at X = −cubeWidth: '~'

        Face at Z = +cubeWidth: '#'

        Face at Y = −cubeWidth: ';'

        Face at Y = +cubeWidth: '+'

    Each call to calculateForSurface(...) projects a point on one face and writes into the buffers if it passes the depth test.

    By iterating over both cubeX and cubeY, the program effectively samples each square face of the cube.

Render to Terminal
    
    printf("\x1b[H"); // Move cursor to top-left of terminal
    for (int k = 0; k < width * height; k++) {
        putchar(k % width ? buffer[k] : '\n');
    }

    \x1b[H is an ANSI escape code that repositions the cursor at row 1, column 1 (top-left) without clearing the terminal.

    Then the program iterates over every position in the buffer, printing its stored ASCII character.

    Whenever k % width == 0 (the start of a new row), it prints a newline instead of buffer[k] to maintain the 2D layout.

Update Rotation Angles

    A += 0.05f;
    B += 0.05f;
    C += 0.01f;

Slightly increment each rotation angle so that, in the next frame, the cubes appear to have turned around all three axes. By updating these small increments every iteration, the cubes spin continuously.

Frame Delay

    usleep(8000 * 2);

    Introduce a delay of approximately 16 milliseconds (8,000 µs × 2 = 16,000 µs). This throttles the loop so that the ASCII animation does not scroll too quickly. On a typical terminal, this achieves roughly 60 frames per second (or whatever rate the system can sustain).

How the Visual Effect Comes Together

    Z-Buffering ensures that closer faces of a cube overwrite farther faces at the same screen pixel, preventing “holes” or incorrect face ordering.

    ASCII Characters (@, $, ~, #, ;, +) differentiate the six faces of each cube, providing simple visual cues for orientation and depth.

    Multiple Cube Sizes and Offsets place three cubes side by side (large cube, medium cube, small cube), each turning in sync. By adjusting cubeWidth and horizontalOffset before sampling each cube, the code stacks them horizontally across the screen.

    Continuous Rotation (via incremental updates to angles A, B, and C) simulates a smooth 3D spinning effect. The slight difference in rotation increments (C changes more slowly) produces a subtle variation in how the cubes rotate around the Z axis compared to the X and Y axes.

When compiled and run in a terminal that supports ANSI escape sequences, the program continuously redraws the three cubes as they rotate, giving an engaging ASCII-art visualization of 3D geometry and depth.

Compilation and Execution

To compile and run this code on a Unix-like system (Linux or macOS), use:

    gcc -o rotating_cubes rotating_cubes.c -lm
    ./rotating_cubes

    The -lm flag links the math library (required for sin(), cos(), etc.).

    On Windows, use a compiler that supports Win32 APIs (for example, MinGW). No additional libraries are required beyond the standard C runtime and Windows API.

Key Takeaways

    3D to 2D Projection: The program calculates rotated 3D coordinates and applies a simple perspective projection (xp = screen_center_x + K1 * (x / z), etc.) to determine where to plot each point in 2D.

    Depth Buffering: By keeping track of the reciprocal depth (1/z) for each pixel, the code ensures that nearer surfaces overwrite farther ones, avoiding visual artifacts where a back face would otherwise appear on top.

    ASCII Art Rendering: Each point on a cube’s surface is represented by a single ASCII character. Different characters are chosen for different faces, making each face visually distinct.

    Platform-Specific Timing: The custom usleep() for Windows guarantees consistent frame timing across platforms.

This combination of 3D math, real-time depth testing, and ASCII rendering creates a compelling demonstration of how one can simulate 3D animations purely within a text terminal.
