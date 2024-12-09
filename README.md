# Explainer for the Cross-Origin Storage (COS) API

<img src="https://raw.githubusercontent.com/tomayac/cross-origin-storage/refs/heads/main/logo-cos.svg" alt="Cross-Origin Storage (COS) logo" width="100">

This proposal outlines the design of the **Cross-Origin Storage (COS)** API, which allows web applications to store and retrieve files across different web origins with explicit user consent. Using concepts introduced in **File System Living Standard** defined by the WHATWG, the COS API facilitates secure cross-origin file storage and retrieval for large files, such as AI models, SQLite databases, offline storage archive files, and shared WebAssembly (Wasm) modules. The API uses file hashes for integrity, while human-readable names allow for easier management.

## Authors

- [Thomas Steiner](mailto:tomac@google.com), Chrome Developer Relations

## Participate

- https://github.com/tomayac/cross-origin-storage/issues

## Introduction

The **Cross-Origin Storage (COS)** API provides a cross-origin file storage and retrieval mechanism for web applications. It allows applications to store and access large files, such as AI models, SQLite databases, offline storage archives, and shared WebAssembly (Wasm) modules across domains securely and with user consent. Files are identified by their hashes, ensuring consistency, and a human-readable name can be assigned to files for easier management. The API uses concepts like `FileSystemHandle` from the **File System Living Standard** with a focus on cross-origin usage.

```js
const hash = 'SHA-256: 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4';
const humanReadableName = 'Large AI Model';

// This triggers a permission prompt:
// example.com wants to access the file "Large AI Model" stored by your browser.
// [Allow this time] [Allow on every visit] [Don't allow]
const handle = await navigator.crossOriginStorage.getFileHandle(hash, humanReadableName);

if (handle) {
  // The file exists in Cross-Origin Storage
  const fileBlob = await handle.getFile();
  // Do something with the blob
  console.log('Retrieved', humanReadableName, fileBlob);
}
```

## Goals

COS aims to:

- Provide a cross-origin storage mechanism for web applications to store and retrieve large files like AI models, SQLite databases, offline storage archives, and Wasm modules.
- Ensure security and user control with explicit consent before accessing or storing files.
- Use SHA-256 (or similar, see [Appendix B](#appendix-b-blob-hash-with-the-web-crypto-api)) hashes for file identification, guaranteeing data integrity and consistency.
- Allow developers to assign human-readable names to files for easier management.

## Non-goals

COS does not aim to:

- Replace existing storage solutions such as the **Origin Private File System**, the **Cache API**, **IndexedDB**, or **localStorage**.
- Replace content delivery networks (CDNs). The assumption is that the required prompting will discourage websites from using the COS API unless it really makes sense to have resources available cross-origin, such as when they can benefit from using a possibly cached version instead of downloading a new one.
- Provide backend or cloud storage solutions.
- Allow cross-origin file access _without_ explicit user consent.

## User research

Feedback from developers working with large AI models, datasets, and WebAssembly modules has highlighted the need for an efficient way to store and retrieve such large files across web applications. These developers are looking for a standardized solution that allows files to be stored once and accessed by multiple applications, without needing to re-upload or re-compile the files repeatedly. COS ensures this is possible while maintaining privacy and security via user consent.

## Use cases

### Use case 1: Large AI models

Developers working with large AI models can store these models once and access them across multiple web applications. By using the COS API, the model can be stored under its hash and retrieved with user consent, minimizing repeated uploads and ensuring file integrity. An example is Google's [Gemma 2](https://huggingface.co/google/gemma-2-2b/tree/main) model [`g-2b-it-gpu-int4.bin'` (1.35 GB)](https://storage.googleapis.com/jmstore/kaggleweb/grader/g-2b-it-gpu-int4.bin'). Another example is the [`Llama-3-70B-Instruct-q3f16_1-MLC` (33 GB)](https://huggingface.co/mlc-ai/Llama-3-70B-Instruct-q3f16_1-MLC/tree/main) model.

### Use case 2: Large database files and offline storage archive files

Web applications may depend on large SQLite databases, for example, for geodata as provided by Geocode Earth [`whosonfirst-data-admin-latest.db.bz2` (8.00 GB)](https://geocode.earth/data/whosonfirst/combined/). Another use case is large archive files, for example, [ZIM files](https://wiki.openzim.org/wiki/ZIM_file_format) like [`wikipedia_en_all_maxi_2024-01.zim` (109.89 GB)](https://library.kiwix.org/#lang=eng&category=wikipedia) as used by PWAs like [Kiwix](https://pwa.kiwix.org/www/index.html). Storing such files once in the COS API has the advantage that multiple web apps can share the same resources.

### Use case 3: Big shared Wasm modules

Web applications that utilize large WebAssembly modules can store these modules using COS and share them across applications. This enables efficient sharing of resources between applications, reducing redundant downloading and improving performance. Examples are Kotlin Multiplatform's bindings to Skia [`skiko.wasm` (3.2 MB)](https://github.com/JetBrains/skiko) or Flutter's [`skwasm.wasm` (1.1 MB)](https://docs.flutter.dev/platform-integration/web/renderers#skwasm).

## Potential solution

### File Storage Process

The **COS** API will be available through `navigator.crossOriginStorage`. Files will be stored and retrieved using their hashes, ensuring that each file is uniquely identified. A human-readable name can be provided for easier management of files.

#### Storing a file

1. The contents of the file will be hashed using SHA-256 (or an equivalent secure algorithm, see [Appendix B](#appendix-b-blob-hash-with-the-web-crypto-api)). The used algorithm will be communicated in the hash as a valid [`HashAlgorithmIdentifier`](https://w3c.github.io/webcrypto/#dom-hashalgorithmidentifier), separated by a colon and the actual hash.
2. A handle for the file will be requested, specifying the file's hash and a human-readable name.
3. A permission prompt will be displayed to the user, asking if it's okay to store the file with the provided name, hash, and size.
4. If a file with the hash already exists, only the human-readable name will be stored by the browser and associated with the current origin.
5. Else, upon user consent, the file and the human-readable name will be stored by the browser and the human-readable name be associated with the current origin.

```js
// Example usage to store a file
const hash = 'SHA-256: 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4'; // Assume the file is already identified with a hash
const humanReadableName = 'Large AI model';

// If the file already exists, nothing to be done
const fileExists = await navigator.crossOriginStorage.getFileHandle(hash);

if (!fileExists) {
  const handle = await navigator.crossOriginStorage.getFileHandle(hash, humanReadableName, { create: true });

  // Request user permission and store the file
  const writableStream = await handle.createWritable();
  await writableStream.write(fileBlob);  // Assuming the blob is available
  await writableStream.close();  // Close the writable stream properly after writing
}
```

This will prompt the user to confirm the storage, displaying the human-readable name, file size, and hash.

#### Retrieving a file

1. The application will request a file handle using the file's hash.
2. A permission prompt will appear, showing the file's human-readable name and hash.
3. After user consent, the file will be retrieved.

```js
// Retrieve the file handle and human-readable name
const hash = 'SHA-256: 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4';

// If the file already exists, get it from COS
const fileExists = await navigator.crossOriginStorage.getFileHandle(hash);

if (fileExists) {
  const { handle, humanReadableName } = await navigator.crossOriginStorage.getFileHandle(hash);

  // Request user permission and retrieve the file
  const fileBlob = await handle.getFile();
  console.log(`Retrieved file: ${humanReadableName}`);

  // Return the file as a Blob
  console.log(fileBlob);  // This will return the Blob object
} else {
  // Obtain the the file from the network
  const fileBlob = await fetch('https://example.com/ai-model.bin').then(response => response.blob());
  // Return the file as a Blob
  console.log(fileBlob);  // This will return the Blob object
}
```

#### Storing and retrieving a file across unrelated pages

To illustrate the capabilities of the COS API, consider the following example where two unrelated pages want to interact with the same large language model. The first page stores the model in COS, while the second page retrieves it, each using different human-readable names in English and Spanish.

##### Page 1: Storing a large language model with an English Name

On the first page, a web application stores a large language model in COS with a human-readable English name, "Large AI Model."

```js
// The known hash of the file
const hash = 'SHA-256: 8f434346648f6b96df89dda901c5176b10a6d83961dd3c1ac88b59b2dc327aa4';

// Check if the file already exists
const fileExists = await navigator.crossOriginStorage.getFileHandle(hash);

if (fileExists) {
  // Use the file and return
  // …
  return;
}

// The file doesn't exist, so fetch it from the network
const fileBlob = await fetch('https://example.com/large-ai-model.bin').then(response => response.blob());
const controlHash = await getBlobHash(fileBlob); // Compute the control hash using the method in Appendix B

// Check if control hash and known hash are the same
if (controlHash !== hash) {
  // Downloaded file and wanted file are different
  // …
  return;
}

const humanReadableName = 'Large AI Model';
const handle = await navigator.crossOriginStorage.getFileHandle(hash, humanReadableName, { create: true });

// Request user permission and store the file
const writableStream = await handle.createWritable();
await writableStream.write(fileBlob);
await writableStream.close();

console.log(`File stored with name: ${humanReadableName}`);
```

##### Page 2: Retrieving the same model with a Spanish name

On the second, entirely unrelated page, a different web application retrieves the same model from COS but refers to it with a human-readable Spanish name, "Modelo de IA Grande."

```js
// The known hash of the file
const humanReadableName = 'Modelo de IA Grande';

// Check if the file exists in COS
const fileExists = await navigator.crossOriginStorage.getFileHandle(hash);

if (fileExists) {
  const { handle } = await navigator.crossOriginStorage.getFileHandle(hash, humanReadableName);

  // Request user permission and retrieve the file
  const fileBlob = await handle.getFile();
  // This now logs the Spanish name, even if the file was stored with an English name by page 1
  console.log(`File retrieved with name: ${humanReadableName}`);

  // Use the fileBlob as needed
} else {
  console.error('File not found in COS.');
}
```

##### Key points

- **Unrelated pages:** The two pages belong to different origins and do not share any context, ensuring the example demonstrates cross-origin capabilities.
- **Human-readable names:** Each page assigns its own human-readable name, localized to the user's context. The COS API associates these names with the file's hash, not with the file contents.
- **Cross-origin sharing:** Despite the different names and origins, the file is securely shared via its hash, demonstrating the APl's ability to facilitate cross-origin file storage and retrieval.

## Detailed design discussion

### User Consent

The permission prompt must clearly display the file's name, size, and hash to ensure users understand what file they are being asked to store or retrieve. The goal is to strike a balance between providing sufficient technical details and maintaining user-friendly simplicity.

### Privacy

Since the file is stored and retrieved upon explicit user permission, there's no way for files stored in COS to become supercookies without raising the user's suspicion. Privacy-sensitive user agents can decide to prompt upon every retrieval operation, others can decide to only prompt once, and auto-allow from thereon. Storing a file in COS always requires the user's permission.

## Open questions

### Minimum resource size

Should there be a required minimum resource size for a resource to be eligible for COS? Most likely not, since it would be trivial to inflate the file size of non-qualifying resources by adding space characters or comments. The assumption is that the required prompting would be scary enough for websites to only use COS for resources where it really makes sense to have them available cross-origin, that is, where they could profit themselves from using a potentially already cached version rather than downloading their own version from the network.

### Eviction

Browsers should likely treat resources in COS under the same conditions as if they were [`persist()`]([https://storage.spec.whatwg.org/#dom-storagemanager-persisted](https://storage.spec.whatwg.org/#dom-storagemanager-persist))ed as per the Storage Living Standard.

User agents are envisioned to offer a manual UI for the user to see and modify what resources are stored in COS and, based on stored information about the origins having used a resource, decide to delete a resource from COS.

### Out-of-bounds access

If a user already has manually downloaded a resource like a large AI model, should the browser offer a way to let the user put the resource in COS? Most likely this doesn't even need specifying, but could just be an affordance provided by the user-agent.

## Considered alternatives

### Alternative: Storing Files Without Hashing

Storing files by their names rather than using hashes would risk collisions and lead to inconsistent access, especially in a cross-origin environment. The use of hashes guarantees unique identification of each file, ensuring that the contents are consistently recognized and retrieved.

## Stakeholder Feedback / Opposition

- **Web Developers**: Positive feedback for enabling sharing large files without repeated downloads, particularly in the context of huge AI models, SQLite databases, offline storage archive files, and large Wasm modules.

## References

- [File System Living Standard](https://fs.spec.whatwg.org/)
- [Web Cryptography API](https://w3c.github.io/webcrypto/)
- [Storage Living Standard ](https://storage.spec.whatwg.org/)

## Acknowledgements

Many thanks for valuable feedback, inspiration, or ideas from:

- Kenji Baheux
- Kevin Moore
- Christian Liebel

## Appendices

### Appendix A: Full IDL

```idl
[SecureContext]
interface mixin NavigatorCrossOriginStorage {
  [SameObject] readonly attribute CrossOriginStorageManager crossOriginStorage;
};
Navigator includes NavigatorCrossOriginStorage;
WorkerNavigator includes NavigatorCrossOriginStorage;

[SecureContext]
interface CrossOriginStorageManager {
  Promise<FileSystemFileHandle> getFileHandle(DOMString hash, optional FileSystemGetFileOptions options = {});
};

// From https://fs.spec.whatwg.org/#dictdef-filesystemgetfileoptions
dictionary FileSystemGetFileOptions {
  boolean create = false;
};
```

### Appendix B: Blob hash with the Web Crypto API

```js
async function getBlobHash(blob) {
  const hashAlgorithmIdentifier = 'SHA-256';

  // Get the contents of the blob as binary data contained in an ArrayBuffer
  const arrayBuffer = await blob.arrayBuffer();

  // Hash the arrayBuffer using SHA-256
  const hashBuffer = await crypto.subtle.digest(hashAlgorithmIdentifier, arrayBuffer);

  // Convert the ArrayBuffer to a hex string
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  const hashHex = hashArray.map(byte => byte.toString(16).padStart(2, '0')).join('');

  return `${hashAlgorithmIdentifier}: ${hashHex}`;
}

// Example usage
const fileBlob = await fetch('https://example.com/ai-model.bin').then(response => response.blob());
getBlobHash(fileBlob).then(hash => {
  console.log('Hash:', hash);
});
```
