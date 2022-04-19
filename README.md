# jpeg-python
JPEG encoder/decoder written in Python

# Image Processing Computer Assignment 4 - Parisa Daj U00743495

## 1. Jpeg Encoding

The main purpose of this assignment is to apply image jpeg compression using the steps from the following image [1]. To do so, it is better to convert RGB data to YIQ data. However, in later digital standards, instead of YIQ, YCbCr is being utilized [2]. In the next step, each channel is divided into 8x8 blocks. on each block, at first a Discrete Cosine Transform is applied. The result then, will be quantized using a quantization matrix. On the resulting matrix, using the Zig-zag algorithm, the matrix is squeezed into a 1-D array. Separating the DC and AC terms of the output, Run-Length Encoding applies to the AC term and Delta encoding to the DC term. Then, using Huffman or Arithmetic Encoding, the 0-1 output will be the encoded result of the input RGB image. As follows, the details in each part will be explained.

![Jpeg encoding process [3]](https://d33wubrfki0l68.cloudfront.net/7237dd0093b6a8070b2c927673fd73bc797561d2/33b0a/images/decoding_jpeg/encoding.png)

### Change Color Space

JPEG algorithm implementations typically encode luminance and chrominance (YUV encoding) instead of RGB. JPEG is great for this because the human eye is not very good at detecting high-frequency brightness changes over a small area, so we can essentially reduce the frequency and the human eye will not recognize the difference. In visual result, Image quality is almost unaffected by compression. Color is determined by the Y component (also known as luminance or luma) and the U and V components (also known as chroma). A color's brightness is determined by the Y component, while the color itself is defined by the U and V components (also known as chroma) [1]. The luma component in YCbCr is Y, while Cb and Cr are the blue-difference and red-difference chroma components. By using gamma-corrected RGB primaries, light intensity is nonlinearly encoded by Y, which is luminance [3]. This conversion is done by convert function from pillow.image package. But, if we want to use YIQ instead of YCbCr, transformRGB2YIQ function can do it for us. In the decoding section, then, we need to use transformYIQ2RGB function instead of converting YCbCr to RGB.

### 8x8 Blocks
Subsampling each channel requires dividing it into 8x8 blocks. Chromatic subsampling yields Minimum Coding Unit (MCU) blocks of size 8x8, 16x8, or most commonly 16x16, depending on the subsampling technique used [4]. Three loops (with 8, 8, 3 iterations) are nested to go through the blocks for each channel.

### DCT
Cosine waves are created using discrete data points in the form of Discrete Cosine Transforms. When working with JPEG, DCT will tell us how to reproduce an 8x8 image block by using a matrix of cosines. An 8x8 coefficient matrix is the result of applying DCT, which shows how much each of the 64 total cosine functions contributed to the 8x8 input matrix. There are typically larger coefficients in the top left corner of a DCT coefficient matrix and smaller coefficients in the bottom right corner. Cosine functions with the lowest frequency appear in the top left corner, and cosines with the highest frequency appear in the bottom right corner. Hence, most images contain only a limited amount of high-frequency information but a great deal of low-frequency information. Because, as previously stated, humans are poor observers of high-frequency changes, if we set the bottom right components of each DCT matrix to 0, the resulting image would remain unchanged [3]. At first, 128 is subtracted from the pixel values to prepare them from DCT calculation. Then, dct_2d function does the discrete cosine transformation from scipy.fftpack package with orthonormalization. 

### Quantization
We haven't done anything lossy up to now. We merely converted 8x8 blocks of YUV components into 8x8 blocks of cosine functions with no information loss. The lossy component appears during the quantization stage. Quantization is the process of converting a pair of values in a defined range into a discrete value. In our example, this is simply a fancy way of saying "change the higher frequency coefficients in the DCT output matrix to 0." When you save an image in JPEG format, most image editing tools prompt you to specify how much compression you require. The percentage you enter here determines how much quantization is used and how much higher frequency information is lost. This is when lossy compression comes into play. When high-frequency information is lost, the resultant JPEG picture cannot be used to reproduce the identical original image. The quantization matrix used in this program is the known Q50 matrix showed below. The quantized matrix is the round of dividing the DCT matrix, element-wise, by the quantization matrix[3]. The following quantization table is defined in load_quantization_table function in "utils.py" file.

![Quantization table [3]](https://d33wubrfki0l68.cloudfront.net/f7711104437cea39bc024520e63d3a030af8538c/86eb2/images/decoding_jpeg/quant-matrix.png)
### Zig-Zag
![Zig-zag procedure [3]](https://people.ece.cornell.edu/land/courses/ece5760/FinalProjects/f2009/jl589_jbw48/jl589_jbw48/zigzag.jpg)
This encoding is chosen because after quantization, the majority of the low frequency (most relevant) information is stored at the beginning of the matrix, and the zig-zag encoding stores all of it at the beginning of the 1-D matrix [3]. This movement is called in block_to_zigzag function which takes points from zigzag_points function at "utils.py" file, and convert the block to the 1-D array. In this function, the movement starts from origin (0, 0) in each block. It checks what is the initial position and depending on the initial state, the possible remaining movements, and the movements made in the past, changes the upcoming position of the matrix element index.

### RLE, DPCM

Repeated data is compressed using run-length encoding. Most of the zig-zag encoded 1-D arrays had so many 0s at the end.
RLE helps us to recover all of that wasted space and use fewer bytes to represent all of those 0s.
Delta encoding is a method of representing a byte in relation to the byte preceding it.
This is useful for the compression that occurs in the following phase.
Every DC value in a DCT coefficient matrix is delta encoded relative to the DC value before it in JPEG.
This implies that if you alter your picture's very first DCT coefficient, the entire image will be messed up,
yet if you change the first value of the final DCT matrix, just a very small portion of your image would be impacted.
This is beneficial since the initial DC value in your picture is generally the most diverse, and by using Delta encoding, we can move the rest of the DC values closer to 0, resulting in improved compression in the following phase of Huffman Encoding [3]. This encoding is applied within the Huffman Tree calling individually for DC and AC, separated chrominance and luminance values. 

### Huffman Encoding
Huffman encoding is a technique for lossless data compression. This type of variable-length mapping is possible because of Huffman encoding.
It takes some input data, maps the most often occurring characters to smaller bit patterns and the least frequently occurring characters to bigger bit patterns,
and then arranges the mapping into a binary tree.
The DCT (Discrete Cosine Transform) information is stored in a JPEG using Huffman encoding. 
A JPEG can have up to four Huffman tables, which are saved in the "Define Huffman Table" section. 
The DCT coefficients are saved in two separate Huffman tables. The first has just the DC values from the zig-zag tables, whereas the second contains only the AC values from the zig-zag tables. This means that we'll have to combine the DC and AC values from two different matrices during decoding. Because the DCT information for the luminance and chrominance channels is saved separately, we have two sets of DC information and two sets of AC information, for a total of four Huffman tables [3]. In "huffman.py" program,  the huffman tree is designed. The final result includes DC and AC huffman tables each consist of chrominance and luminance coded values.

## Decoding Jpeg

### Decode Huffman Table

JPEG files include four Huffman tables. Because this is the final stage in the encoding process, it should be the first step in the decoding process.
he 0 indicates that no Huffman code of length 1 exists. 1 indicates that there is just one Huffman code of length 2.
n the DHT part, there are always 16 bytes of data after the class and ID information [3]. 
To do so, the input is passed to read_image_file function. It uses JPEGFileReader class to load the 4 input tables.
As follows, the DC and AC tables are separated, and after three 8x8x3 nested loops in the blocks, the read_huffman_code decodes the huffman tables. 


### Quantization and Zig-Zag Decode

The result from the decoded huffman table is a one-dimensional output from zig-zag algorithm.
To retrieve the quantized matrix from zig-zag, we need to move on the zig-zag table in the opposing direction using zigzag_to_block function.
To decode the quantized matrix, we can simply multiply it by the quantization table as is done in dequantize function.

### Inverse DCT

Inverse Discrete Cosine Trasform is done using idct_2d function taking advantage of idct function in scipy.fftpack package.
Adding 128 to the pixel values, will result in them ranging between 0 and 255. The decode is completed in this stage and given the first image below, the resulting image will look like the second image.



![Original image](https://user-images.githubusercontent.com/42092569/164095237-bacebd0e-9b6d-44ca-b36b-1799cbea2148.png)


![Decoded Image](https://user-images.githubusercontent.com/42092569/164095543-eb60c2d3-a1d2-41cd-a329-532e1a2fbfdc.png)



## References

[1] Understanding and Decoding a JPEG Image using Python, accessed: April 19, 2022 at [https://yasoob.me/about/]

[2] YIQ Wikipedia, accessed: April 19, 2022 at [https://en.wikipedia.org/wiki/YIQ]

[3] YCbCr Wikipedia, accessed: April 19, 2022 at [https://en.wikipedia.org/wiki/YCbCr]

[4] JPEG Block-splitting Wikipedia, accessed: April 19, 2022 at [https://en.wikipedia.org/wiki/JPEG#Block_splitting]

