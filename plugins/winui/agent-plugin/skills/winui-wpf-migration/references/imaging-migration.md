# Imaging Migration

Use this reference when the WPF application uses `System.Windows.Media.Imaging`, WPF bitmap transforms/encoders, `Image.Source`, or persisted imaging settings.

## Separate Display from Processing

WPF often uses `BitmapSource` for both UI display and pixel processing. WinUI migrations should make that boundary explicit:

| Need | Typical WinUI/Windows type |
|---|---|
| Display an image in XAML | `Microsoft.UI.Xaml.Media.Imaging.BitmapImage` or `SoftwareBitmapSource` |
| Decode, encode, transform, or inspect metadata | `Windows.Graphics.Imaging.BitmapDecoder`, `BitmapEncoder`, `BitmapTransform`, `SoftwareBitmap` |

`BitmapImage` is a UI source, not a replacement for WPF's processing pipeline.

Replace `PresentationCore`/`System.Windows.Media.Imaging` dependencies in the converted project. Do not enable WPF in the WinUI application to make old imaging code compile.

## Core Type Map

| WPF | Windows.Graphics.Imaging / WinUI | Notes |
|---|---|---|
| `BitmapDecoder.Create(...)` and format-specific decoders | `BitmapDecoder.CreateAsync(...)` | Unified async decoder; format is detected from the stream. |
| `decoder.Frames[0]` | `await decoder.GetFrameAsync(0)` | Frame access is async. |
| `decoder.Frames.Count` | `decoder.FrameCount` | `int` → `uint`. |
| `decoder.CodecInfo.ContainerFormat` | `decoder.DecoderInformation.CodecId` | A decoder ID is not a container GUID or an encoder ID; map explicitly. |
| `BitmapSource` pixel data | `SoftwareBitmap` | Use the pixel-data type in processing interfaces. |
| `BitmapImage` for display | `Microsoft.UI.Xaml.Media.Imaging.BitmapImage` | Similar name, different framework namespace/lifetime. |
| `FormatConvertedBitmap` | `SoftwareBitmap.Convert(...)` or decoder pixel-format request | Validate alpha and color behavior. |
| `TransformedBitmap` / `ScaleTransform` | `BitmapTransform` configured on decoder/encoder operations | WinRT often expresses transforms declaratively. |
| `CroppedBitmap` | `BitmapTransform.Bounds` | Bounds use pixel coordinates and unsigned sizes. |
| Format-specific encoders | `BitmapEncoder.CreateAsync(codecId, stream)` | Unified async encoder factory. |
| `encoder.Save(stream)` | `await encoder.FlushAsync()` | Observe failures/cancellation and stream position. |
| `BitmapMetadata` | `BitmapProperties` / image property queries | Query names and supported write behavior differ. |

## Streams

Windows.Graphics.Imaging consumes WinRT stream interfaces such as `IRandomAccessStream`:

```csharp
await using FileStream input = File.OpenRead(path);
using IRandomAccessStream randomAccess = input.AsRandomAccessStream();

BitmapDecoder decoder = await BitmapDecoder.CreateAsync(randomAccess);
```

Keep the underlying stream and its WinRT wrapper open through decoding/transcoding, and scope their disposal together.

`BitmapEncoder` expects an empty output stream. If a destination may already contain data, set its `Size` to `0` and seek to the beginning before creating the encoder. Do not truncate the decoder's source stream before the source-dependent encoding operation has completed.

For same-file replacement, follow the official SimpleImaging pattern: encode to a separate empty `InMemoryRandomAccessStream`; after `FlushAsync`, seek the temporary stream and destination to zero, set the destination's `Size` to `0`, and copy the encoded bytes.

## Decode, Transform, and Encode

Same-format resize/transcode shape, where `outputStream` is separate from `inputStream`:

```csharp
BitmapDecoder decoder = await BitmapDecoder.CreateAsync(inputStream);

outputStream.Size = 0;
outputStream.Seek(0);
BitmapEncoder encoder = await BitmapEncoder.CreateForTranscodingAsync(
    outputStream,
    decoder);

encoder.BitmapTransform.ScaledWidth = targetWidth;
encoder.BitmapTransform.ScaledHeight = targetHeight;
encoder.BitmapTransform.InterpolationMode = BitmapInterpolationMode.Fant;

await encoder.FlushAsync();
```

`CreateForTranscodingAsync` can only create an encoder in the decoder's image format and does not support codec-specific encoding options. Use it for same-format edits where unspecified source data should be preserved. For a format conversion, such as PNG to JPEG, or for options such as JPEG quality, decode the pixels and create a fresh encoder with `CreateAsync(targetEncoderId, outputStream, encodingOptions)`.

Before calling `SetSoftwareBitmap`, request or convert to a pixel format accepted by the encoder path (commonly `Bgra8`, `Rgba8`, or `Rgba16` as supported). Keep that bitmap alive until its frame is committed, then dispose it. Do not pass an arbitrary decoded gray/indexed format with a mismatched alpha mode.

Map WPF `HighQuality`/`Fant` scaling to `BitmapInterpolationMode.Fant`; `Linear` and `NearestNeighbor` have matching modes. Preserve the chosen quality for default/low-quality settings and compare resized pixels within tolerance.

## Encoder Options

WPF exposes format-specific encoder classes and strongly typed properties. Windows.Graphics.Imaging uses codec IDs and bitmap property sets for many options.

JPEG `QualityLevel` changes from an integer percentage (1–100) to a floating-point value (0–1). Divide by `100f`, not by the integer `100`:

```csharp
int qualityPercent = 85; // The application's validated setting.
var properties = new BitmapPropertySet
{
    ["ImageQuality"] = new BitmapTypedValue(qualityPercent / 100f, PropertyType.Single),
};

outputStream.Size = 0;
outputStream.Seek(0);
BitmapEncoder encoder = await BitmapEncoder.CreateAsync(
    BitmapEncoder.JpegEncoderId,
    outputStream,
    properties);
```

Pass TIFF compression as a creation option too, on an empty output stream:

```csharp
var tiffOptions = new BitmapPropertySet
{
    ["TiffCompressionMethod"] = new BitmapTypedValue(
        (byte)TiffCompressionMode.Lzw, PropertyType.UInt8),
};
BitmapEncoder tiffEncoder = await BitmapEncoder.CreateAsync(
    BitmapEncoder.TiffEncoderId, outputStream, tiffOptions);
```

`TiffCompressionMode.Lzw` is `4`; `2` means `Ccitt3`. Encoding options belong in `CreateAsync`/`GoToNextFrameAsync`, not the frame's metadata property set. Verify codec support with actual files.

Other codec differences:

- PNG interlace: `InterlaceOption` uses `PropertyType.Boolean`; verify the encoded output with the target SDK/codec.
- WMP/HDP/JPEG XR. Windows.Graphics.Imaging exposes JPEG XR decoder/encoder IDs; verify deployed codec behavior, metadata, and output compatibility rather than declaring the format unsupported.

## Display a SoftwareBitmap

`SoftwareBitmapSource` requires a display-compatible pixel/alpha format:

```csharp
SoftwareBitmap displayBitmap = bitmap;

if (bitmap.BitmapPixelFormat != BitmapPixelFormat.Bgra8 ||
    bitmap.BitmapAlphaMode != BitmapAlphaMode.Premultiplied)
{
    displayBitmap = SoftwareBitmap.Convert(
        bitmap,
        BitmapPixelFormat.Bgra8,
        BitmapAlphaMode.Premultiplied);
}

var source = new SoftwareBitmapSource();
await source.SetBitmapAsync(displayBitmap);
image.Source = source;
```

Decide who owns and disposes the original and converted bitmaps. Avoid leaking one `SoftwareBitmap` per preview refresh.

## Async and Concurrency

Change synchronous per-file methods to return `Task`, propagate async to callers, and keep a bounded concurrency limit:

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = maxConcurrentImages,
    CancellationToken = cancellationToken,
};
await Parallel.ForEachAsync(files, options,
    async (file, ct) => await ResizeFileAsync(file, ct));
```

- Limit parallelism by both CPU and expected decoded image size; a few large images can exhaust memory.
- Observe all task exceptions.
- Do not use `.Result`/`.Wait()` on the UI thread.
- Dispatch only UI progress/preview updates, not the decode itself.

## Multi-Frame Images

For a fresh multi-page TIFF encoder, commit each frame before disposing its bitmap. Keep both streams open and start with an empty output stream. Select `orientationMode` and `colorManagementMode` from the application's existing behavior; calculate each frame's target dimensions using its existing aspect-ratio/crop policy.

```csharp
BitmapEncoder encoder = await BitmapEncoder.CreateAsync(
    BitmapEncoder.TiffEncoderId, outputStream, tiffOptions);
for (uint i = 0; i < decoder.FrameCount; i++)
{
    BitmapFrame frame = await decoder.GetFrameAsync(i);
    // Calculate targetWidth/targetHeight for this frame.
    var frameTransform = new BitmapTransform
    {
        ScaledWidth = targetWidth, ScaledHeight = targetHeight,
        InterpolationMode = BitmapInterpolationMode.Fant,
    };
    using SoftwareBitmap bitmap = await frame.GetSoftwareBitmapAsync(
        BitmapPixelFormat.Bgra8, BitmapAlphaMode.Premultiplied,
        frameTransform, orientationMode, colorManagementMode);
    encoder.SetSoftwareBitmap(bitmap);
    // Write the selected current-frame metadata before committing it.
    if (i + 1 < decoder.FrameCount)
        await encoder.GoToNextFrameAsync(tiffOptions);
    else
        await encoder.FlushAsync();
} // Each bitmap is disposed only after its commit has completed.
```

`GoToNextFrameAsync` commits the current frame and appends an empty one. Never call it after the last frame; finish with `FlushAsync`. Set container metadata before the first commit. GIF output also needs frame delay/disposal metadata. Reject or explicitly handle a target codec that cannot represent the source's frames.

## Metadata and Orientation

`CreateForTranscodingAsync` starts from the source decoder and is useful when preserving source information; `CreateAsync` starts a new encode path.

Validate at least:

- EXIF orientation and whether pixels are physically rotated.
- Date taken, camera, GPS, comment, rating, and custom metadata required by the app.
- ICC profile/color management.
- Whether “remove metadata” also removes orientation needed for correct display.
- Metadata behavior across resize, format conversion, and multi-frame output.

Use property queries supported by the selected codec. Treat unsupported or rejected metadata writes as expected error paths rather than assuming every WPF metadata field maps.

## Persisted Imaging Settings

WPF codec container GUIDs or enum values may be stored in settings or exposed through IPC/CLI contracts. Removing the WPF type must not corrupt those contracts.

Keep persisted IDs unchanged and map only values found in the application's released settings. For example, WPF JPEG/PNG container GUIDs and WinRT decoder IDs need separate lookup tables:

```csharp
var legacyToEncoder = new Dictionary<Guid, Guid>
{
    [new("19e4a5aa-5662-4fc5-a0c0-1758028e1057")] = BitmapEncoder.JpegEncoderId,
    [new("1b7cfaf4-713f-473c-bbcd-6137425faeaf")] = BitmapEncoder.PngEncoderId,
};
var decoderToEncoder = new Dictionary<Guid, Guid>
{
    [BitmapDecoder.JpegDecoderId] = BitmapEncoder.JpegEncoderId,
    [BitmapDecoder.PngDecoderId] = BitmapEncoder.PngEncoderId,
};
if (!decoderToEncoder.TryGetValue(decoder.DecoderInformation.CodecId, out Guid encoderId))
    throw new NotSupportedException("No output encoder is configured for this format.");
```

Use the same checked lookup for a persisted container GUID. Do not pass a decoder ID or `Guid.Empty` to `CreateAsync`.

If settings serialized WPF enum values, retain their names/numbers in an application-owned enum, for example `enum LegacyPngInterlace { Default = 0, On = 1, Off = 2 }`. Test deserialization with settings from supported released versions before removing the WPF enum reference.

## Testing

Compare the correct invariant. Dimensions, visible pixels within tolerance, metadata, and user-visible behavior are often more meaningful than byte equality across different codec implementations.

Match numeric types in assertions: decoder `FrameCount`/`PixelWidth`/`PixelHeight` and transform sizes are `uint`; `SoftwareBitmap` dimensions and constructor arguments are `int`. For a 96-pixel fixture, use `Assert.AreEqual(96u, outputDecoder.PixelWidth)` and `Assert.AreEqual(96, outputBitmap.PixelWidth)`.

For pixel comparisons, decode expected and actual output to the same format, alpha mode, orientation, and color policy. With an expected BGRA byte array prepared under that policy:

```csharp
byte[] actualPixels = (await outputDecoder.GetPixelDataAsync(
    BitmapPixelFormat.Bgra8, BitmapAlphaMode.Premultiplied,
    new BitmapTransform(), orientationMode, colorManagementMode)).DetachPixelData();
Assert.AreEqual(expectedPixels.Length, actualPixels.Length);
for (int i = 0; i < actualPixels.Length; i++)
    Assert.IsTrue(Math.Abs(actualPixels[i] - expectedPixels[i]) <= tolerance);
```

Reopen the encoded output to test metadata. For a fixture whose policy retains orientation `6` without rotating pixels:

```csharp
var values = await outputDecoder.BitmapProperties.GetPropertiesAsync(
    new[] { "System.Photo.Orientation" });
Assert.IsTrue(values.TryGetValue("System.Photo.Orientation", out BitmapTypedValue value));
Assert.AreEqual((ushort)6, (ushort)value.Value);
```

When pixels are physically rotated, assert the resulting dimensions/pixels and the policy's normalized or absent orientation tag. Test removal using the supported query for the specific field; an empty result for one query does not prove that all metadata was stripped.

If existing tests use unsafe pointers through `IMemoryBufferByteAccess`, enable `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>` only in that test project and keep the buffer/reference lifetime scoped to the pointer access. The managed `GetPixelDataAsync` example above does not require unsafe code.

## Official References

- [`BitmapEncoder.CreateAsync`](https://learn.microsoft.com/uwp/api/windows.graphics.imaging.bitmapencoder.createasync)
- [`BitmapEncoder.CreateForTranscodingAsync`](https://learn.microsoft.com/uwp/api/windows.graphics.imaging.bitmapencoder.createfortranscodingasync)
- [Create, edit, and save bitmap images](https://learn.microsoft.com/windows/apps/develop/media-authoring-processing/imaging)
- [`GoToNextFrameAsync`](https://learn.microsoft.com/uwp/api/windows.graphics.imaging.bitmapencoder.gotonextframeasync), [encoding options](https://learn.microsoft.com/windows/win32/wic/-wic-creating-encoder), and [`TiffCompressionMode`](https://learn.microsoft.com/uwp/api/windows.graphics.imaging.tiffcompressionmode)
- [`SoftwareBitmap` constructors](https://learn.microsoft.com/uwp/api/windows.graphics.imaging.softwarebitmap.-ctor) and [bitmap property queries](https://learn.microsoft.com/uwp/api/windows.graphics.imaging.bitmapproperties.getpropertiesasync)
