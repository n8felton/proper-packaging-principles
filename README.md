# Proper Packaging Principles

These proper packaging principles are meant to be a guideline for software developers on how to build and host software packages for macOS applications. These are based on experience from members of the [MacAdmins community](https://macadmins.org) who have collectively spent _hundreds of thousands_ of hours deploying and maintaining software. The goal is to help you, help us. When software is properly packaged, it becomes easy to deploy, update, and manage. This enables both MacAdmins and end users to ensure they are running the latest version of your software.

## Summary

1. **Build native macOS packages**

   Using third-party package formats (such as [install4j](https://www.ej-technologies.com/products/install4j/overview.html)) or custom internal software installers (Looking at you, Adobe...) is never a good idea and needlessly increases the complexity of installing software. Most (if not all) Mac management tools have native support for deploying native macOS packages.

1. **Version numbers go up**

   Most tools available to manage software deployments use version numbers to determine if a software package needs to be installed or updated. No matter how minor a change to software package, the version number should be incremented to ensure these tools know they need to install or update the package.
   The [recommended format](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring) is three period-separated integers, such as 1.21.42. See also: [semver](https://semver.org)

1. **Choose a good, stable component package identifier**

   Software deployment tools query the macOS receipts database — keyed on the component package identifier — to determine what is installed and whether it needs updating. Choose a recognizable reverse-domain identifier (e.g. `com.example.Product`), keep it consistent across versions, and only encode a version in the identifier when multiple versions install side-by-side.

   See [Appendix D](#appendix-d---component-package-identifiers) for more details.

1. **Naming conventions are necessary and helpful**

   Give your packages meaningful names and version numbers. `Installer.pkg` or `product.pkg` is not helpful and can lead to filenames such as `Installer (1).pkg` and `product (2).pkg` cluttering up the Downloads folder. Instead, consider filenames that include your vendor and/or product name, compiled architecture, and version number, e.g. `product-{arch}-{version}.pkg`.

   See [Appendix A](#appendix-a---package-filename-naming-conventions) for more details.

1. **Downloading packages should be easy**

   A public, static URL should be provided to download the latest package of the software. While there are exceptions, _MOST_ software should
   be available without the need for _ANY_ authentication (See [Software configuration and licensing should be done separately](#licensing-separately)).

   See [Appendix B](#appendix-b---download-urls) for more details on the download URL itself.

1. **Serve downloads with proper HTTP semantics**

   Beyond a clean URL, the server should return headers that let tools save files with correct names (`Content-Disposition`), detect changes without re-downloading (`ETag` / `Last-Modified`), verify integrity (a published checksum), and resume interrupted transfers (`Accept-Ranges`).

   See [Appendix C](#appendix-c---serving-downloads-http-headers--integrity) for more details.

1. **Do not assume that your package will be installed interactively via the GUI (Installer.app)**

   In the age of Mobile Device Management (MDM), installation via the [`InstallApplication`](https://developer.apple.com/documentation/devicemanagement/installapplicationcommand/command) command is possible, in addition to command line installations via the `installer` command.

   <!----><a name="licensing-separately"></a>

1. **Software configuration and licensing should be done separately**

   Any necessary configuration or licensing of software should be done outside of the native macOS package. In a managed environment, this ideally should be handled using MDM Profiles. For non-managed, end-user environments, this should be handled by the software using something like a first-run dialog.

# Appendix A - Package Filename Naming Conventions

## Best Practices

1. Include your vendor/developer name and/or the product/project name.
1. Include the architecture type, using `uname -m` values, or `universal`.
1. Include the version number.

Using these best practices allows for key metadata about the package to be known without needing to inspect the package. It also helps to avoid filenames such as `Installer (1).pkg` and `product (2).pkg` when downloading multiple architectures or versions of the same software package.

##### Examples

```
product-{arch}-{version}.pkg
product-x86_64-1.21.42.pkg
product-arm64-1.21.42.pkg
product-universal-1.21.42.pkg
```

### Which names to use?

Recommending that a package filename should include the vendor/developer name and/or the product/project name is sometimes easier said than done. This can be due to multiple factors, including single-product developers when the vendor name and the product name are identical; e.g. [BlueJ](https://www.bluej.org), Dropbox, and TIDAL to name a few.

Ultimately, the package filename should be recognizable and easy to identify what is going to be installed.

# Appendix B - Download URLs

## Best Practices

1. The URL should be static and NOT contain any metadata that will change with each package.
1. If including the architecture type, use `uname -m` values, or `universal`.
1. Use TLS (HTTPS)
1. Use a domain name that matches the vendor and/or product name(s)

How the server _serves_ that download — response headers, integrity, and resumability — is covered separately in [Appendix C](#appendix-c---serving-downloads-http-headers--integrity).

### Static URL

Using a static URL allows for finding the latest version of a software package easier, as well as provides the opportunity for automation to obtain the latest version. Users and IT professionals can document or bookmark the URL for repeated use, or link to it in a blog post or community forum to help ensure it will be valid the next time it is referenced. The URL should NOT contain any metadata, such as version number.

#### Examples

##### Good

A static URL (redirects acceptable), but may need to change in the future due to lack of metadata in the path. May be considered the "most common" download, while less common download URLs may contain more metadata.

```
https://example.com/product/download
https://example.com/download/product.pkg
https://dl.example.com/product.pkg
```

###### Real-world Examples

- https://zoom.us/client/latest/ZoomInstallerIT.pkg

##### Better

A static URL (redirects acceptable), but contains metadata such as `arch` and `release` that can be altered as needed, such as replacing `arch` with `x86_64` or `arm64`, or replacing `release` with `stable` or `beta`

```
https://example.com/download/product/arch/release/latest
https://example.com/download/product/arch/release/product.pkg
https://dl.example.com/product/arch/release/product.pkg
```

###### Real-world Examples

- https://dl.google.com/dl/chrome/mac/universal/stable/gcem/GoogleChrome.pkg
- https://dl.google.com/dl/chrome/mac/universal/beta/GoogleChromeBeta-Enterprise.pkg

### Including Architecture Type

Using the output of `uname -m` allows for automation when acquiring software. On macOS this reports `x86_64` (Intel) or `arm64` (Apple Silicon), the 2 most commonly reported machine architectures.

Note that these values should match those used with macOS and not any other operating system, as they are often reported differently. The Apple Silicon token is the most common pitfall: macOS reports `arm64`, while Linux reports `aarch64` for the same architecture. Use the macOS value (`arm64`).

Prefer `uname -m` over the `arch` command. On macOS, `arch` with no arguments reports `i386` on Intel (not `x86_64`), so it is not suitable for this purpose.

```
curl -LOJ https://example.com/download/product/$(uname -m)/release/latest
```

### Use TLS (HTTPS)

Serving software downloads from an insecure HTTP URL allows "person-in-the-middle" attacks like [this one from 2016](https://www.macrumors.com/2016/02/09/sparkle-hijacking-vulnerability). Prevent this by ensuring all your web hosts and content distribution servers are using HTTPS with valid TLS certificates.

### Use a known domain

Software package downloads should come from a domain belonging to the developer to help promote trust in the download. For example, it is fairly easy for a bad actor to create any S3 bucket name that may match your download, e.g. `https://product-name-latest.s3.amazonaws.com/product-arm64-1.21.42.pkg`

#### Bad Real-world Examples

- https://cellprofiler-releases.s3.amazonaws.com/CellProfiler-macOS-4.2.1.zip
- https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-darwin-arm64

# Appendix C - Serving Downloads (HTTP Headers & Integrity)

Once a [download URL](#appendix-b---download-urls) exists, how the server responds to a request for it matters just as much as the URL itself. The following practices help tools and automation save files correctly, avoid re-downloading unchanged packages, verify integrity, and recover from interrupted transfers.

## Best Practices

1. Use the `Content-Disposition` HTTP header with type `attachment` and the `filename` parameter.
1. Provide an `ETag` HTTP header that changes whenever the package contents change.
1. Provide a `Last-Modified` HTTP header that reflects when the package was last updated.
1. Publish a cryptographic checksum (such as SHA-256) of the package so downloads can be verified.
1. Support HTTP range requests, advertised via the `Accept-Ranges` header, so large package downloads can be resumed.

### `Content-Disposition` HTTP Header

This can allow for saving downloaded files with relevant metadata, such as architecture type and/or version number.

For example, a download URL of `https://dl.example.com/product/arch/release/product.pkg` should have the `Content-Disposition` HTTP header set to `Content-Disposition: attachment; filename="product-arm64-1.21.42.pkg"`. This can help prevent saving the download with filename such as `product.pkg`, which can result in not knowing what version of the software package you have downloaded, while also preventing unhelpful filenames such as `product (1).pkg` and `product (2).pkg` from cluttering up your Downloads folder.

This header is supported by modern browsers, as well as many of the tools and languages that are used by Mac Admins, such as

- [curl](https://curl.se/docs/manpage.html#-J)
- [wget](https://www.gnu.org/software/wget/manual/wget.html#index-Content_002dDisposition)
- [python3](https://docs.python.org/3/library/http.client.html#http.client.HTTPResponse.headers) (via `HTTPResponse.headers.get_filename()`)
- [Swift](https://developer.apple.com/documentation/foundation/urlresponse/1415924-suggestedfilename)

#### Test URL

This URL will generate the appropriate header to help build and test automations against.

```
https://httpbin.org/response-headers?content-disposition=%20attachment%3Bfilename%3D%22product-arm64-1.21.42.pkg%22
```

You can test this yourself with curl

```
curl -LOJ "https://httpbin.org/response-headers?content-disposition=%20attachment%3Bfilename%3D%22product-arm64-1.21.42.pkg%22"

cat "product-arm64-1.21.42.pkg"
```

##### Real-world Examples

- https://dl.pstmn.io/download/latest/osx ([Postman](https://www.postman.com))

  `Content-Disposition: attachment; filename=Postman%20for%20macOS%20(x64).zip`

- https://launcher-public-service-prod06.ol.epicgames.com/launcher/api/installer/download/EpicGamesLauncher.dmg

  `Content-Disposition: attachment; filename=EpicInstaller-20.1.0.dmg`

### `ETag` HTTP Header

The [`ETag`](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/ETag) HTTP header provides a unique identifier for a specific version of a resource. When the package contents change, the `ETag` value should change as well. This allows automation to determine whether the package at a static URL has changed without downloading the entire file.

Tools and automation can issue a conditional request using the [`If-None-Match`](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/If-None-Match) request header with a previously stored `ETag` value. If the resource has not changed, the server responds with `304 Not Modified` and no body, saving bandwidth and time. If it has changed, the server responds with `200 OK`, the new content, and an updated `ETag`.

Prefer a strong `ETag` (one without the `W/` weak-validator prefix) derived from the package contents, such as a hash of the file. Avoid values that change on every request or that are tied to a specific server in a load-balanced fleet, as these defeat the purpose of caching and change detection.

Ideally, provide both an `ETag` and a [`Last-Modified`](#last-modified-http-header) header: the `ETag` acts as the strong, content-derived validator while `Last-Modified` serves as a fallback. Per [RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110#section-13.1.3), when a client sends both `If-None-Match` and `If-Modified-Since`, the server evaluates `If-None-Match` first.

```
ETag: "a1b2c3d4e5f6"
```

You can test conditional requests with curl using the `--etag-save` and `--etag-compare` options:

```
curl -LOJ --etag-save product.etag https://example.com/download/product.pkg
curl -LOJ --etag-compare product.etag --etag-save product.etag https://example.com/download/product.pkg
```

### `Last-Modified` HTTP Header

The [`Last-Modified`](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/Last-Modified) HTTP header indicates the date and time the package was last changed. Like the `ETag` header, this allows automation to determine whether a package at a static URL has been updated without downloading the entire file.

Tools can issue a conditional request using the [`If-Modified-Since`](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/If-Modified-Since) request header with a previously stored `Last-Modified` value. If the resource has not changed since that time, the server responds with `304 Not Modified` and no body. If it has changed, the server responds with `200 OK` and the new content.

The value must be an [HTTP-date](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/Date#syntax) in GMT, as defined by [RFC 9110, Section 5.6.7](https://datatracker.ietf.org/doc/html/rfc9110#section-5.6.7), and should accurately reflect when the package contents actually changed rather than when the file was deployed or copied.

```
Last-Modified: Mon, 02 Jan 2006 15:04:05 GMT
```

You can inspect this header with curl by requesting only the headers:

```
curl -sI https://example.com/download/product.pkg | grep -i last-modified
```

#### Validators and Redirects

If your static download URL [redirects](#static-url) to another location, such as a versioned object on a CDN, the `ETag` and `Last-Modified` headers must be present on the final `200 OK` response, not on the intermediate `3xx` redirect. Tools that follow the redirect chain (e.g. `curl -L`) read the validators from the final response, so headers set only on the redirect will be ignored.

### Package Checksum

Publishing a cryptographic checksum, such as SHA-256, allows downloaders to verify that the bytes they received are authentic and uncorrupted. This is distinct from change detection: an `ETag` is **not** an integrity guarantee, as a server can set it to any value, while a checksum lets a downloader confirm the package matches what the developer published.

Checksums are commonly published as a sidecar file alongside the package (e.g. `product.pkg.sha256`) or listed on the download or release page.

```
shasum -a 256 product-arm64-1.21.42.pkg
```

Downloaders can verify against a published checksum:

```
echo "a1b2c3...  product-arm64-1.21.42.pkg" | shasum -a 256 --check
```

### Range Requests (`Accept-Ranges`)

Supporting HTTP range requests allows clients to resume an interrupted download instead of starting over, which is valuable for large installers and on unreliable networks.

A server advertises support with the [`Accept-Ranges`](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/Accept-Ranges) header set to `Accept-Ranges: bytes`. Clients then request a portion of the file using the [`Range`](https://developer.mozilla.org/docs/Web/HTTP/Reference/Headers/Range) request header, and the server responds with `206 Partial Content`.

```
Accept-Ranges: bytes
```

You can resume an interrupted download with curl using the `-C -` option:

```
curl -LOJ -C - https://example.com/download/product.pkg
```

# Appendix D - Component Package Identifiers

Many popular software deployment tools reference the macOS receipts store to query metadata about installed software packages, such as file paths, package versions, and installation dates. This information is used to determine if a software package needs to be updated. Despite often being called a "receipts database", it is not a single database but a set of receipt files — a `.plist` (metadata) and a `.bom` (bill of materials) per package — kept under `/var/db/receipts/`.

This information is linked to the component package identifier. Therefore, it is highly recommended to choose a recognizable identifier that follows macOS conventions and to maintain consistency with it.

> The macOS Installer recognizes a package as an upgrade to an already-installed package only if the package identifiers match. Therefore, it is advisable to set a meaningful, consistent identifier when you build the package.
>
> — <cite>`pkgbuild` man page</cite>

The identifier alone does not determine whether a package is installed — it is how the Installer and deployment tools _locate_ the matching receipt. Once an identifier matches, the package **version** is compared to decide whether the package is an upgrade or a downgrade. Both the identifier and a meaningful version are required for reliable update detection (see [Package Version](#package-version)).

Component package identifiers typically use reverse domain name notation, for example, `com.example.Product`.

### Component vs. Distribution Identifiers

`pkgbuild` builds a _component_ package and sets its `--identifier`. This is the identifier recorded under `/var/db/receipts/` and the one deployment tools read. `productbuild` wraps one or more component packages into a _distribution_ (product) archive, which can carry its own separate identifier. That outer identifier is metadata for the distribution file and is not what tools match against the receipts store, so the guidance here applies to the **component** identifier.

## Best Practices

1. Use reverse domain name notation, for example `com.example.Product`.
1. Choose a recognizable, meaningful identifier and keep it consistent across versions.
1. Never change the identifier between releases. A changed identifier looks like a different, unrelated package: the old receipt is orphaned, the upgrade is missed, and admins must intervene manually.
1. Always set a meaningful package version (`pkgbuild --version`). Do not ship the default of `0`, which prevents proper upgrade/downgrade detection.
1. Restrict identifiers to reverse-DNS characters — ASCII letters, digits, `.`, and `-` — and keep casing consistent. Avoid spaces, underscores, and other punctuation.
1. Optionally include a `pkg` segment, for example `com.example.pkg.Product`.
1. Use a distinct identifier for each component when one distribution package ships multiple payloads (app, helper, framework), sharing a common prefix, for example `com.example.pkg.Product.Helper`. Do not reuse one identifier for unrelated payloads.
1. Only embed a version in the identifier string when multiple versions can be installed, used, and updated side-by-side.

### Examples

#### Good

```
com.example.product
com.example.Product
com.example.Product.15
```

##### Real-world Examples

```
com.amazon.corretto.17
com.apple.MacEvalUtility
com.tapbots.Ivory
```

#### Better

```
com.example.pkg.Product
com.example.pkg.Product15
```

##### Real-world Examples

```
com.apple.pkg.Xcode
com.ninxsoft.pkg.mist-cli
fr.whitebox.pkg.Packages
```

### Package Version

The package version is distinct from the version that may appear _inside_ an identifier string. Every package carries a version, set with `pkgbuild --version`, that is recorded in the receipt.

> Packages with the same identifier are compared using this version, to determine if the package is an upgrade or downgrade. If you don't specify a version, a default of zero is assumed, but this may prevent proper upgrade/downgrade checking.
>
> — <cite>`pkgbuild` man page</cite>

In other words, the identifier selects the receipt and the package version decides the rest, so always set a meaningful version that goes up with each release rather than shipping the default `0`. This package version is separate from the application's `CFBundleShortVersionString` and need not match it, though keeping them aligned avoids confusion (see [Version numbers go up](#summary)).

### Versioning Identifiers

A version should only be embedded in the identifier string itself when multiple versions of a product can be installed, used, and updated side-by-side. Otherwise, version information does not belong in the identifier — the package version (above) already tracks it.

This is why identifiers such as `com.example.Product.15` and `com.amazon.corretto.17` carry a version: they identify a specific major version that is intended to coexist with other major versions on the same system. When only one version of a product is installed at a time, omit the version from the identifier so the macOS Installer can recognize new packages as upgrades. Embedding a changing version in the identifier of a single-instance product breaks upgrade detection, leaving an orphaned receipt for every release.

### Related Commands

`pkgutil --pkgs` — List all installed package IDs

`pkgutil --pkg-info <package-id>` — Print extended information about the specified `<package-id>`

`pkgutil --files <package-id>` — Print a list of files on disk that are associated with the specified `<package-id>`

# References and Resources

## Apple Software Versions

- https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleversion
- https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring
- https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/DistributionDefinitionRef/Chapters/Distribution_XML_Ref.html

## Apple Developer References

- [Creating Distribution-Signed Code for Mac](https://developer.apple.com/forums/thread/701514)
- [Packaging Mac Software for Distribution](https://developer.apple.com/forums/thread/701581)
- [Signing a Mac Product For Distribution](https://developer.apple.com/forums/thread/128166)
- [Placing Content in a Bundle](https://developer.apple.com/documentation/bundleresources/placing_content_in_a_bundle)
- [Updating Mac Software](https://developer.apple.com/documentation/security/updating_mac_software)
- [Signing a daemon with a restricted entitlement](https://developer.apple.com/documentation/xcode/signing-a-daemon-with-a-restricted-entitlement)
- [Embedding a command-line tool in a sandboxed app](https://developer.apple.com/documentation/xcode/embedding-a-helper-tool-in-a-sandboxed-app)
- [Embedding nonstandard code structures in a bundle](https://developer.apple.com/documentation/xcode/embedding-nonstandard-code-structures-in-a-bundle)
- [Distribution XML Reference](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/DistributionDefinitionRef/Chapters/Distribution_XML_Ref.html)
- [Installer-dev mailing list](https://lists.apple.com/mailman/listinfo/installer-dev)

## macOS Software Packaging Tools

### macOS Built-in

- [pkgbuild](x-man-page://pkgbuild)
- [productbuild](x-man-page://productbuild)
- [installer](x-man-page://installer)
- [pkgutil](x-man-page://pkgutil)

### Third-party Package Creation

- [munki-pkg](https://github.com/munki/munki-pkg)
- [The Luggage](https://github.com/unixorn/luggage)
- [Packages](http://s.sudre.free.fr/Software/Packages/about.html)

### Third-party Utilities

- [Suspicious Package](https://www.mothersruin.com/software/SuspiciousPackage)
- [Pacifist](https://www.charlessoft.com)
- [Apparency](https://mothersruin.com/software/Apparency)

## Other Resources

- [Packaging for Apple Administrators](https://books.apple.com/us/book/packaging-for-apple-administrators/id1173928620)
- [The Encyclopedia of Packages – MacDevOps YVR ’21](https://scriptingosx.com/mdo21)
- [Guidelines for Mac software packaging](https://wiki.afp548.com/index.php/Guidelines_for_Mac_software_packaging)
  - [Original "The Commandments of Packaging" on Posterous by Gary Larizza](https://web.archive.org/web/20130304070133/http://glarizza.posterous.com/the-commandments-of-packaging)
- [Mac Package Hall of Shame](https://macpkghallofshame.tumblr.com)
- [PackageMaker How-to](http://s.sudre.free.fr/Stuff/PackageMaker_Howto.html)
- [Installation - The Lost Scrolls](http://s.sudre.free.fr/Stuff/Installer/Menu.html)
- [Flat Package Format - The missing documentation](http://s.sudre.free.fr/Stuff/Ivanhoe/FLAT.html)
- [Packages - Resources](http://s.sudre.free.fr/Software/Packages/resources.html)

## Other Thoughts

- It's ***macOS***. It is no longer *Mac OS X*, or *OS X*, or *OSX*.

  In the [Apple Style Guide](https://support.apple.com/guide/applestyleguide) there is a [_**Mac operating systems**_](https://support.apple.com/guide/applestyleguide/m-apsg72b28652/web#apd246e83209) section that contains style guidelines for how to reference macOS versions.

- If you have a product targeting *Windows*, the Mac counterpart should be *macOS*, not *Mac*. *Mac* is the hardware, akin to *Dell*, *Lenovo*, or *HP*.

  - *Exception*: *mac* is an acceptable 3-letter short notation of *macOS* to match *win* for *Windows*
