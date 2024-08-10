Cumulation is the current WIP title for the image/video sorting software I'm working on.
I have a big collection of images, screenshots, video clips, etc. spread across many many folders and hard drives and the plan is to use this software to properly sort and manage that collection.

I also plan to add duplicate detection and since that step is probably going to be one of the more complex ones, I decided to start with that. 

# Duplicate Detection
What I've been doing so far is set up a [Tauri](https://tauri.app/) project and send an array of folder paths to the rust backend of the application.
This works without issues just using a `Vec<String>` on the rust side.

Walking the directory I've implemented so far via [walkdir](https://crates.io/crates/walkdir) as follows:

```rust
for entry in WalkDir::new(curr_path).into_iter().filter_map(|e| e.ok()) {
	if !entry.metadata().unwrap().is_file() {
		continue;
	}
	//...
}
```

`curr_path` here is simply the current path converted from String that the frontend sent back here.

Next I looked at [`VideoDuplicateFinder`](https://github.com/0x90d/videoduplicatefinder)'s code to find some inspiration on how exactly I would implement image duplicate detection. This is done via a method called `GetGrayscaleValues`, which takes in an `Image` and a darkness threshold percentage and returns an array of `byte`s.

[`videoduplicatefinder/VDF.Core/Utils/GrayBytesUtils.cs:41`](https://github.com/0x90d/videoduplicatefinder/blob/1d93ac090f842fb49cd36ed741729aad58e81b0a/VDF.Core/Utils/GrayBytesUtils.cs#L41)
```c# showLineNumbers{41}
public static unsafe byte[]? GetGrayScaleValues(Image original, double darkProcent = 80) {
	// Lock the bitmap's bits.  
	using var tempImage = original.CloneAs<Bgra32>();
	tempImage.TryGetSinglePixelSpan(out var pixelSpan);
	Span<byte> span = MemoryMarshal.AsBytes(pixelSpan);

	byte[] buffer = new byte[GrayByteValueLength];
	int stride = 0;
	stride = tempImage.GetPixelRowSpan(0).Length;
	int bytes = stride * original.Height;

	int count = 0, all = original.Width * original.Height;
	int buffercounter = 0;
	for (int i = 0; i < span.Length; i += 4) {
		byte r = span[i + 2], g = span[i + 1], b = span[i];
		byte brightness = (byte)Math.Round(0.299 * r + 0.5876 * g + 0.114 * b);
		buffer[buffercounter++] = brightness;
		if (brightness <= BlackPixelLimit)
			count++;
	}
	return 100d / all * count >= darkProcent ? null : buffer;
}
```

This method will basically do this in sequence:
- Take in an `Image` with 16x16 pixel size
- Initialize a `byte[]` with a size of 16x16 (256)
- Loop through every pixel in the image (1024) in steps of 4
- Ignore the alpha channel but extract red/green/blue values
- Calculate the [perceived (relative) luminance](https://www.w3.org/TR/AERT/#color-contrast) of the pixel with some slightly modified values
- Add the red value of the pixel to the buffer
- Check if the pixel is bright enough to be perceived (value over `0x20 (32)`)
- Return the buffer if enough pixels (higher than parameter set threshold) are perceivable

In my humble opinion this should be called `GetRedValues` since I don't think the image is put into grayscale anywhere even before running through this method but hey what do I know.
It's copied as [`Bgra32`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.media.pixelformats.bgra32?view=windowsdesktop-8.0) which according to the documentation

> Gets the [Bgra32](https://learn.microsoft.com/en-us/dotnet/api/system.windows.media.pixelformats.bgra32?view=windowsdesktop-8.0#system-windows-media-pixelformats-bgra32) pixel format. [Bgra32](https://learn.microsoft.com/en-us/dotnet/api/system.windows.media.pixelformats.bgra32?view=windowsdesktop-8.0#system-windows-media-pixelformats-bgra32) is a sRGB format with 32 bits per pixel (BPP). 
> Each channel (blue, green, red, and alpha) is allocated 8 bits per pixel (BPP).

Does nothing more than allocate 8 bits (1 byte) per channel per pixel.
Now my version of this looks a bit uglier, however it does the same thing. There's just one issue...
## Accuracy of the algorithm
Now VideoDuplicateFinder works just fine for videos and probably pictures too, although I haven't tested this extensively, however this kind of algorithm is suited a lot better for actual photographs. Video game screenshots or artworks fair a lot worse with this algorithm. 

Now to detect differences between images, all it does is take that `byte[]` and check the absolute difference between the first and second image's red values. It also does some checking for available CPU features to make this process a bit faster but we'll ignore all that optimization stuff for now.

[`videoduplicatefinder/VDF.Core/Utils/GrayBytesUtils.cs:83`](https://github.com/0x90d/videoduplicatefinder/blob/1d93ac090f842fb49cd36ed741729aad58e81b0a/VDF.Core/Utils/GrayBytesUtils.cs#L83)
```c# showLineNumbers{83}
public static unsafe float PercentageDifference(byte[] img1, byte[] img2)
{
    Debug.Assert(img1.Length == img2.Length, "Images must be of the same size");
    long diff = 0;
    if (Avx2.IsSupported)
    {
        Vector256<ushort> vec = Vector256<ushort>.Zero;
        Span<Vector256<byte>> vImg1 = MemoryMarshal.Cast<byte, Vector256<byte>>(img1);
        Span<Vector256<byte>> vImg2 = MemoryMarshal.Cast<byte, Vector256<byte>>(img2);

        for (int i = 0; i < vImg1.Length; i++)
            vec = Avx2.Add(vec, Avx2.SumAbsoluteDifferences(vImg2[i], vImg1[i]));
            
        for (int i = 0; i < Vector256<ushort>.Count; i++)
            diff += Math.Abs(vec.GetElement(i));
    }
    else if (Sse2.IsSupported) {
        Vector128<ushort> vec = Vector128<ushort>.Zero;
        Span<Vector128<byte>> vImg1 = MemoryMarshal.Cast<byte, Vector128<byte>>(img1);
        Span<Vector128<byte>> vImg2 = MemoryMarshal.Cast<byte, Vector128<byte>>(img2);
  
        for (int i = 0; i < vImg1.Length; i++)
            vec = Sse2.Add(vec, Sse2.SumAbsoluteDifferences(vImg2[i], vImg1[i]));

        for (int i = 0; i < Vector128<ushort>.Count; i++)
            diff += Math.Abs(vec.GetElement(i));
    }
    else {
        for (int i = 0; i < img1.Length; i++)
            diff += Math.Abs(img1[i] - img2[i]);
    }
    return (float)diff / img1.Length / 256;
}
```

I put two distinctly different images through it and got an absolute difference of about 20%, so it's safe to say that this isn't the most accurate method to detect differences I assume. However the default threshold for differences is set to `< 4%` so maybe in the very top percentages an overlap is highly unlikely?

Anyway I'll look into more sophisticated duplicate detection algorithms later, for now [this](https://content-blockchain.org/research/testing-different-image-hash-functions/) seems to be a good starting point, as does [this](https://towardsdatascience.com/a-guide-to-building-an-image-duplicate-finder-system-4a46021410f1) on a lower level.

Reading the implementation for a `pHash` algorithm today has almost given me an aneurysm so I'll write a bit about that.

https://github.com/aetilius/pHash/blob/master/src/pHash.cpp#L305
```c++
static CImg<float> ph_dct_matrix(const int N) {
    CImg<float> matrix(N, N, 1, 1, 1 / sqrt((float)N));
    const float c1 = sqrt(2.0 / N);
    for (int x = 0; x < N; x++) {
        for (int y = 1; y < N; y++) {
            matrix(x, y) = c1 * cos((cimg::PI / 2 / N) * y * (2 * x + 1));
        }
    }
    return matrix;
}

static const CImg<float> dct_matrix = ph_dct_matrix(32);
int ph_dct_imagehash(const char *file, ulong64 &hash) {
    if (!file) {
        return -1;
    }
    CImg<uint8_t> src;
    try {
        src.load(file);
    } catch (CImgIOException &ex) {
        return -1;
    }
    CImg<float> meanfilter(7, 7, 1, 1, 1);
    CImg<float> img;
    if (src.spectrum() == 3) {
        img = src.RGBtoYCbCr().channel(0).get_convolve(meanfilter);
    } else if (src.spectrum() == 4) {
        int width = src.width();
        int height = src.height();
        img = src.crop(0, 0, 0, 0, width - 1, height - 1, 0, 2)
                  .RGBtoYCbCr()
                  .channel(0)
                  .get_convolve(meanfilter);
    } else {
        img = src.channel(0).get_convolve(meanfilter);
    }

    img.resize(32, 32);
    const CImg<float> &C = dct_matrix;
    CImg<float> Ctransp = C.get_transpose();

    CImg<float> dctImage = (C)*img * Ctransp;

    CImg<float> subsec = dctImage.crop(1, 1, 8, 8).unroll('x');

    float median = subsec.median();
    hash = 0;
    for (int i = 0; i < 64; i++, hash <<= 1) {
        float current = subsec(i);
        if (current > median) hash |= 0x01;
    }

    return 0;
}
```

In here we try to perform a discrete cosine translation on image values and then retrieve the pHash.

To do this we first construct a `dct` matrix of size 32x32 
```c++
static const CImg<float> dct_matrix = ph_dct_matrix(32);
static CImg<float> ph_dct_matrix(const int N) {
    CImg<float> matrix(N, N, 1, 1, 1 / sqrt((float)N));
    const float c1 = sqrt(2.0 / N);
    for (int x = 0; x < N; x++) {
        for (int y = 1; y < N; y++) {
            matrix(x, y) = c1 * cos((cimg::PI / 2 / N) * y * (2 * x + 1));
        }
    }
    return matrix;
}
```

Then we try to load the image from the file path provided:
```c++
CImg<uint8_t> src;
try {
    src.load(file);
} catch (CImgIOException &ex) {
    return -1;
}
```

And after successfully loading the image we first construct an instance of `CImg<float>` as a tuple called `meanfilter`. Let me explain the values here since this took me a minute.
```c++
CImg<float> meanfilter(7, 7, 1, 1, 1);
```

`CImg` defines multiple possible constructors for the class, one of which takes exactly 5 values:
```c++
CImg (const unsigned int size_x, const unsigned int size_y, const unsigned int size_z, const unsigned int size_c, const T &value)
```

Now most of this is understandable. `size_x` is the width, `size_y` is the height, however the next 2 took a bit of digging but `size_z` is the bit depth of the image (90% confidence it's that anyway).
Now 1 bit images look awful but this is a matrix we construct specifically for convolution so it doesn't matter. `size_c` is the spectrum of the image and lastly `&value` is just what to initially fill every position with.

So all this does is construct a matrix that looks as follows:
$$
\begin{bmatrix}  
1 & 1 & 1& 1& 1& 1& 1\\  
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1
\end{bmatrix}
$$

Next we declare an empty image and then check the spectrum (amount of channels per pixel) of our original image. 3 would be RGB, 4 could be RGBA or CMYK apparently it doesn't differentiate.
We then convert those to `YCbCr`, which is a family of color spaces, that are basically defined as:
- `Y`   = luminance
- `Cb` = Blue difference
- `Cr` = Red difference

We then convolve the `0`th channel of that color space, that being Y (luminance) with our previously constructed `meanfilter`.
```c++
CImg<float> img;
if (src.spectrum() == 3) {
    img = src.RGBtoYCbCr().channel(0).get_convolve(meanfilter);
} else if (src.spectrum() == 4) {
    int width = src.width();
    int height = src.height();
    img = src.crop(0, 0, 0, 0, width - 1, height - 1, 0, 2)
              .RGBtoYCbCr()
              .channel(0)
              .get_convolve(meanfilter);
} else {
    img = src.channel(0).get_convolve(meanfilter);
}
```

Convolving matrices for everyone uninitiated basically just means that you take the our `meanfilter` in this case, overlay it on top of the image's matrix of luminance values step by step and then multiply every overlapping value. Afterwards we sum them together into the center pixel and fill the matrix segment with duplicates of that value. This will give us the *mean* or *average* of all the pixel luminance values per segment.
$$
\begin{align*}
\begin{bmatrix}  
1 & 1 & 1& 1& 1& 1& 1\\  
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1
\end{bmatrix}
\cdot
\begin{bmatrix}  
1 & 1 & 1& 1& 1& 1& 1\\  
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1\\
1 & 1 & 1& 1& 1& 1& 1
\end{bmatrix}
\end{align*}
$$