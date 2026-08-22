# 🖼️ Image and Metadata

A mint address identifies the token on chain.

Metadata gives humans something more useful to look at.

During `createToken()`, Moonz publishes the token image first and then builds metadata that references the resulting image URI.

The SDK coordinates both stages for you.

{% hint style="info" %}
The image and metadata publication flow uses the public Moonz API as part of token creation.
{% endhint %}

## 🌌 The Publication Order

The creation flow handles media in this order:

```text
Mint reserved
      ↓
Upload image
      ↓
Receive image URI
      ↓
Build metadata payload
      ↓
Include image URI
      ↓
Publish metadata
      ↓
Receive metadata URI
```

The metadata therefore points at the image that Moonz has already published.

## 🖼️ Image Is Required

`CreateTokenInput` requires:

```ts
image:
  MoonzImageInput
```

The exported image type is:

```ts
type MoonzImageInput =
  | Blob
  | Uint8Array
  | ArrayBuffer;
```

Those are the supported public input forms.

## 🌐 Browser Blob

In a browser environment you can pass a `Blob`.

For example:

```ts
const blob =
  new Blob(
    [bytes],
    {
      type:
        "image/png"
    }
  );

await moonz.createToken({
  creator:
    wallet.publicKey,

  name:
    "Example Token",

  symbol:
    "EXAMPLE",

  image:
    blob,

  devBuySol:
    "0.5",

  signTransaction:
    wallet.signTransaction
});
```

If the `Blob` already has a content type, the SDK can use it automatically.

## 📄 Browser File

A browser `File` is also a `Blob`.

That means a file selected by an upload control can be passed directly.

For example:

```ts
const file =
  input.files?.[0];

if (!file) {
  throw new Error(
    "Choose an image"
  );
}

await moonz.createToken({
  creator:
    wallet.publicKey,

  name:
    "Example Token",

  symbol:
    "EXAMPLE",

  image:
    file,

  devBuySol:
    "0.5",

  signTransaction:
    wallet.signTransaction
});
```

If no explicit filename is provided, the SDK can use the browser `File` name.

## 🔢 Uint8Array

Raw image bytes can also be supplied as:

```ts
Uint8Array
```

For example:

```ts
await moonz.createToken({
  creator,
  name:
    "Example Token",

  symbol:
    "EXAMPLE",

  image:
    imageBytes,

  imageFilename:
    "example.png",

  imageContentType:
    "image/png",

  devBuySol:
    "0.5",

  signTransaction
});
```

The SDK copies the bytes and converts them into a `Blob` for upload.

## 🧱 ArrayBuffer

An `ArrayBuffer` is also accepted.

```ts
await moonz.createToken({
  creator,
  name,
  symbol,

  image:
    arrayBuffer,

  imageFilename:
    "token.png",

  imageContentType:
    "image/png",

  devBuySol:
    "0.5",

  signTransaction
});
```

The SDK wraps the buffer in a `Blob` using the selected content type.

## 🚫 Unsupported Image Inputs

The public image conversion helper only accepts:

```text
Blob

Uint8Array

ArrayBuffer
```

If another value reaches the conversion layer, creation fails with:

```text
Unsupported image input
```

Applications should normalize their image input before calling `createToken()`.

## 🏷️ Image Filename

The optional field:

```ts
imageFilename
```

lets you explicitly define the uploaded filename.

For example:

```ts
imageFilename:
  "moon-cat.png"
```

This is especially useful when using raw byte inputs that do not carry a filename themselves.

## 📄 Filename Fallback

If `imageFilename` is not provided, the SDK checks whether the image is a browser `File`.

If so, it uses:

```ts
input.image.name
```

Otherwise the fallback filename is:

```text
token-image
```

So the effective logic is:

```text
Explicit imageFilename?
      |
      ├── YES
      |     ↓
      |   Use it
      |
      └── NO
            ↓
       Browser File?
         /       \
       YES       NO
        ↓         ↓
   File name   token-image
```

## 🎨 Image Content Type

The optional field:

```ts
imageContentType
```

lets you specify the media type.

For example:

```ts
imageContentType:
  "image/png"
```

or:

```ts
imageContentType:
  "image/jpeg"
```

## 🧬 Content Type Fallback

If `imageContentType` is omitted and the image is a `Blob`, the SDK checks:

```ts
input.image.type
```

If that is also empty, the fallback is:

```text
application/octet-stream
```

For raw `Uint8Array` or `ArrayBuffer` input without an explicit content type, the same generic fallback is used.

## 🧠 Be Explicit With Raw Bytes

For raw byte input, it is better to provide both:

```ts
imageFilename

imageContentType
```

For example:

```ts
image:
  pngBytes,

imageFilename:
  "launch.png",

imageContentType:
  "image/png"
```

That gives the upload stage clear media information.

## 📦 FormData Upload

The SDK constructs a `FormData` payload for the image.

It includes:

```text
creator

file
```

The creator is the normalized Solana creator address.

The file is the image converted into a `Blob` with the selected filename and content type.

## 📡 Image Endpoint

The image is uploaded to:

```text
POST /launch/{mint}/pinata-image
```

using the configured Moonz API.

With the default SDK configuration that means the request is under:

```text
https://api.moonz.fun/launch
```

## 🌙 Mint Before Image

Notice that the endpoint contains:

```text
{mint}
```

The mint is already reserved before image publication begins.

That gives Moonz a stable launch identity throughout the rest of the creation sequence.

## 🚦 Image Progress

Before upload, the SDK emits:

```text
UPLOADING_IMAGE
```

After the API returns a valid image URI, it emits:

```text
IMAGE_PINNED
```

Both progress messages include the reserved mint.

## 🔗 Image URI

The image response must contain:

```ts
uri
```

If the Moonz API does not return an image URI, creation fails with:

```text
Moonz API did not return image URI
```

A successful URI becomes:

```ts
result.imageUri
```

in the final creation result.

## 🧾 Metadata Comes Next

Once the image URI exists, the SDK emits:

```text
PINNING_METADATA
```

Then it submits the metadata request.

## 📦 Metadata Payload

The metadata publication request contains:

```text
creator

name

symbol

description

image

extensions
```

The `image` value is the URI returned by the previous image publication stage.

Conceptually:

```text
Local image
    ↓
Moonz image publication
    ↓
image URI
    ↓
Metadata.image
```

## 🏷️ Name in Metadata

The metadata uses the normalized token name prepared earlier by `createToken()`.

The SDK trims the input name before the creation sequence begins.

## 🔠 Symbol in Metadata

The symbol has already been:

```text
trimmed

converted to uppercase
```

before metadata publication.

So an input such as:

```text
moon
```

is published through the creation flow as:

```text
MOON
```

## 📝 Description Fallback

Description is optional.

If:

```ts
input.description
```

is undefined, the metadata request sends:

```text
empty string
```

rather than leaving the creation flow without a description value.

## 🧩 Extensions

The optional:

```ts
extensions
```

field can carry additional metadata information.

Its public type is:

```ts
Record<string, unknown>
```

For example:

```ts
extensions: {
  website:
    "https://example.com",

  twitter:
    "https://x.com/example",

  telegram:
    "https://t.me/example"
}
```

The SDK passes the object through to the Moonz metadata publication request.

## 📭 Extensions Fallback

If no extensions are supplied, the SDK sends:

```ts
{}
```

This keeps the metadata request shape predictable.

## 📡 Metadata Endpoint

The SDK publishes metadata through:

```text
POST /launch/{mint}/pinata
```

The response is expected to contain:

```ts
uri
```

## 🚦 Metadata Progress

After a successful metadata response, the SDK emits:

```text
METADATA_PINNED
```

The flow then continues into:

```text
QUOTING_DEV_BUY
```

## 🔗 Metadata URI

If the API does not return a metadata URI, the SDK throws:

```text
Moonz API did not return metadata URI
```

A successful metadata URI becomes:

```ts
result.metadataUri
```

## 🌐 Image URI Versus Metadata URI

The final result returns both.

```ts
result.imageUri

result.metadataUri
```

They represent different things.

```text
imageUri
   ↓
Published token image

metadataUri
   ↓
Published token metadata
that references the image
```

## 🔎 Example Result

```ts
const result =
  await moonz.createToken({
    creator:
      wallet.publicKey,

    name:
      "Example Token",

    symbol:
      "EXAMPLE",

    description:
      "Created with Moonz",

    image:
      selectedFile,

    extensions: {
      website:
        "https://example.com"
    },

    devBuySol:
      "0.5",

    signTransaction:
      wallet.signTransaction
  });

console.log(
  result.imageUri
);

console.log(
  result.metadataUri
);
```

## 🧭 Building a Creation Form

A practical creation interface can collect:

```text
Name

Symbol

Description

Image

Optional links or extensions
```

Then pass those values directly into `createToken()`.

## 🖥️ Browser Example

A simplified browser flow might be:

```ts
const file =
  imageInput.files?.[0];

if (!file) {
  throw new Error(
    "Image is required"
  );
}

const result =
  await moonz.createToken({
    creator:
      wallet.publicKey,

    name:
      tokenName,

    symbol:
      tokenSymbol,

    description:
      tokenDescription,

    image:
      file,

    imageFilename:
      file.name,

    imageContentType:
      file.type,

    extensions: {
      website:
        websiteUrl
    },

    devBuySol:
      devBuyAmount,

    slippageBps:
      100,

    signTransaction:
      wallet.signTransaction
  });
```

## 🧠 Do Not Publish Metadata First

The SDK deliberately publishes the image before metadata because the metadata payload needs the resulting image URI.

The sequence is:

```text
Correct

Image
  ↓
Image URI
  ↓
Metadata
```

not:

```text
Metadata
  ↓
Hope image URI appears later
```

## 🛡️ Let createToken Coordinate It

You do not need to manually call the Moonz image and metadata endpoints when using the high level creation method.

The normal integration is:

```ts
await moonz.createToken({
  ...
});
```

The SDK coordinates the order and carries the returned image URI into the metadata request.

## 🔁 Failure Keeps Mint Context

By the time image and metadata publication happen, the mint has already been reserved.

If a later creation stage fails, `createToken()` includes that mint in its wrapped error message.

This is useful for diagnosing where a partially completed launch stopped.

## 🌌 Media Journey

The image and metadata portion of creation can be summarized as:

```text
Creator
  ↓
Image input
  ↓
Blob normalization
  ↓
Filename
  ↓
Content type
  ↓
FormData
  ↓
/pinata-image
  ↓
imageUri
  ↓
Name
Symbol
Description
Extensions
  ↓
/pinata
  ↓
metadataUri
  ↓
Continue launch
```

{% hint style="success" %}
Your application supplies the content.

The SDK handles publication order and carries the resulting URIs through the creation flow.
{% endhint %}

## 💰 Next Stop

The token now has an identity and metadata.

Next we prepare the creator's opening purchase.

Next stop:

**Dev Buy**
