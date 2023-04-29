# Proper Packaging Principles

These proper packaging principles are meant to be a guideline for software developers on how to build and host software packages for macOS applications. These are based on experience from members of the [MacAdmins community](https://macadmins.org) who have collectively spent _hundreds of thousands_ of hours deploying and maintaining software. The goal is to help you, help us. When software if packaged properly, it is easy to deploy, update, and manage and allows MacAdmins and end users alike ensure they are running the latest version of your software.

## Summary

 1. **Build native macOS packages**

     Using third-party package formats (such as [install4j](https://www.ej-technologies.com/products/install4j/overview.html)) or custom internal software installers (Looking at you, Adobe...) is never a good idea and needlessly increases the complexity of installing software. Most (if not all) Mac management tools have native support for deploying native macOS packages.

 2.  **Version numbers go up**

     Most tools available to manage software deployments use version numbers to determine if a software package needs to be installed or updated. No matter how minor a change to software package, the version number should be incremented to ensure these tools know they need to install or update the package.
The [recommended format](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring) is three period-separated integers, such as 1.21.42. See also: [semver](https://semver.org)

 3. **Naming conventions are necessary and helpful**

     Give your packages meaningful names and version numbers.  `Installer.pkg` or `product.pkg` is not helpful and can lead to filenames such as `Installer (1).pkg` and `product (2).pkg` cluttering up the Downloads folder. Instead, consider filenames that include your vendor and/or product name, compiled architecture, and version number, e.g. `product-{arch}-{version}.pkg`. 
     
     See [Appendix A](#appendix-a---package-filename-naming-conventions) for more details.

 4.  **Downloading packages should be easy**

     A static URL should be provided to download the latest package of the software.
     
     See [Appendix B](#appendix-b---static-download-urls) for more details.

5. **Do not assume that your package will be installed interactively via the GUI (Installer.app)** 

     In the age of Mobile Device Management (MDM), installation via the [`InstallApplication`](https://developer.apple.com/documentation/devicemanagement/installapplicationcommand/command) command is possible, in addition to command line installations via the `installer` command.

6. **Software configuration and licensing should be done separately**

    Any necessary configuration or licensing of software should be done outside of the native macOS package. In a managed environment, this ideally should be handled using MDM Profiles. For non-managed, end-user environments, this should be handled by the software using something like a first-run dialog.

# Appendix A - Package Filename Naming Conventions
## Best Practices
1. Include your vendor/developer name and/or the product/project name.
2. Include the architecture type, using `arch` or `uname -m` types, or `universal`.
3. Include the version number.

Using these best practices allows for key metadata about the package to be known without needing to inspect the package. It also helps to avoid filenames such as `Installer (1).pkg` and `product (2).pkg` when downloading multiple architectures or versions of the same software package.

##### Examples

    product-{arch}-{version}.pkg
    product-x86_64-1.21.42.pkg
    product-arm64-1.21.42.pkg
    product-universal-1.21.42.pkg

### Which names to use?
Recommending that a package filename should include the vendor/developer name and/or the product/project name is sometimes easier said than done. This can be due to multiple factors, including single-product developers when the vendor name and the product name are identical; e.g. [BlueJ](https://www.bluej.org/), Dropbox, and TIDAL to name a few.

Ultimately, the package filename should be recognizable and easy to identify what is going to be installed.

# Appendix B - Static Download URLs
## Best Practices
1. The URL should be static and NOT contain any metadata that will change with each package.
2. If including the architecture type, use `arch` or `uname -m` types, or `universal`.
3. Use the `Content-Disposition` HTTP header with type `attachment`  and the `filename` parameter.
4. Use TLS (HTTPS)
5. Use a domain name that matches the vendor and/or product name(s)

### Static URL
Using a static URL allows for finding the latest version of a software package easier, as well as provides the opportunity for automation to obtain the latest version. Users and IT professionals can document or bookmark the URL for repeated use, or link to it in a blog post or community forum to help ensure it will be valid the next time it is referenced. The URL should NOT contain any metadata, such as version number.

#### Examples
##### Good
A static URL (redirects acceptable), but may need to change in the future due to lack of metadata in the path. May be considered the "most common" download, while less common download URLs may contain more metadata.

    https://example.com/product/download
    https://example.com/download/product.pkg
    https://dl.example.com/product.pkg

###### Real-world Examples
* https://zoom.us/client/latest/ZoomInstallerIT.pkg

##### Better
A static URL (redirects acceptable), but contains metadata such as `arch` and `release` that can be altered as needed, such as replacing `arch` with `x86_64` or `arm64`, or replacing `release` with `stable` or `beta`

    https://example.com/download/product/arch/release/latest
    https://example.com/download/product/arch/release/product.pkg
    https://dl.example.com/product/arch/release/product.pkg

###### Real-world Examples
* https://dl.google.com/dl/chrome/mac/universal/stable/gcem/GoogleChrome.pkg
* https://dl.google.com/dl/chrome/mac/universal/beta/GoogleChromeBeta-Enterprise.pkg

### Including Architecture Type
Using the output of `arch` or `uname -m` allows for automation when acquiring software. Note that these values should match those used with macOS and not any other operating system as they are often reported differently. `x86_64` (Intel) or `arm64` (Apple Silicon) are currently the 2 most commonly reported machine architectures using this method.

    curl -LOJ https://example.com/download/product/$(arch)/release/latest
    curl -LOJ https://example.com/download/product/$(uname -m)/release/latest

### `Content-Disposition` HTTP Header
This can allow for saving downloaded files with relevant metadata, such as architecture type and/or version number. 

For example, a download URL of `https://dl.example.com/product/arch/release/product.pkg` should have the `Content-Disoposition` HTTP header set to `Content-Disposition: attachment; filename="product-arm64-1.21.42.pkg"`. This can help prevent saving the download with filename such as `product.pkg`, which can result in not knowing what version of the software package you have downloaded, while also preventing unhelpful filenames such as `product (1).pkg` and `product (2).pkg` from cluttering up your Downloads folder.

This header is supported by modern browsers, as well as many of the tools and languages that are used by Mac Admins, such as 
* [curl](https://curl.se/docs/manpage.html#-J)
* [wget](https://www.gnu.org/software/wget/manual/wget.html#index-Content_002dDisposition)
* [python3](https://docs.python.org/3/library/http.client.html#http.client.HTTPResponse.headers) (via `HTTPResponse.headers.get_filename()`)
* [Swift](https://developer.apple.com/documentation/foundation/urlresponse/1415924-suggestedfilename)

#### Test URL
This URL will generate the appropriate header to help build and test automations against.
```bash
https://httpbin.org/response-headers?content-disposition=%20attachment%3Bfilename%3D%22product-arm64-1.21.42.pkg%22
```
You can test this yourself with curl
```
curl -LOJ "https://httpbin.org/response-headers?content-disposition=%20attachment%3Bfilename%3D%22product-arm64-1.21.42.pkg%22"

cat "product-arm64-1.21.42.pkg"
```

* https://dl.pstmn.io/download/latest/osx ([Postman](https://www.postman.com/))

  `Content-Disposition: attachment; filename=Postman%20v9.11.0%20for%20macOS%20(x64).zip`

* https://launcher-public-service-prod06.ol.epicgames.com/launcher/api/installer/download/EpicGamesLauncher.dmg

  `Content-Disposition: filename=EpicInstaller-14.6.2.dmg`

* https://go.microsoft.com/fwlink/?linkid=2093504 (Microsoft Edge - Stable) 

  `Content-Disposition: attachment; filename=MicrosoftEdge-112.0.1722.64.pkg`


### Use TLS (HTTPS)
Serving software downloads from an insecure HTTP URL allows "person-in-the-middle" attacks like [this one from 2016](https://www.macrumors.com/2016/02/09/sparkle-hijacking-vulnerability/). Prevent this by ensuring all your web hosts and content distribution servers are using HTTPS with valid TLS certificates.

### Use a known domain
Software package downloads should come from a domain belonging to the developer to help promote trust in the download.  For example, it is fairly easy for a bad actor to create any S3 bucket name that may match your download, e.g. `https://product-name-latest.s3.amazonaws.com/product-arm64-1.21.42.pkg`

#### Bad Real-world Examples
* https://cellprofiler-releases.s3.amazonaws.com/CellProfiler-macOS-4.2.1.zip

# References and Resources

## Apple Software Versions
* https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleversion
* https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring
* https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/DistributionDefinitionRef/Chapters/Distribution_XML_Ref.html

## Apple Developer References
* [Creating Distribution-Signed Code for Mac](https://developer.apple.com/forums/thread/701514)
* [Packaging Mac Software for Distribution](https://developer.apple.com/forums/thread/701581)
* [Signing a Mac Product For Distribution](https://developer.apple.com/forums/thread/128166)
* [Placing Content in a Bundle](https://developer.apple.com/documentation/bundleresources/placing_content_in_a_bundle)
* [Updating Mac Software](https://developer.apple.com/documentation/security/updating_mac_software)
* [Signing a daemon with a restricted entitlement](https://developer.apple.com/documentation/xcode/signing-a-daemon-with-a-restricted-entitlement)
* [Embedding a command-line tool in a sandboxed app](https://developer.apple.com/documentation/xcode/embedding-a-helper-tool-in-a-sandboxed-app)
* [Embedding nonstandard code structures in a bundle](https://developer.apple.com/documentation/xcode/embedding-nonstandard-code-structures-in-a-bundle)
* [Distribution XML Reference](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/DistributionDefinitionRef/Chapters/Distribution_XML_Ref.html)

## macOS Software Packaging Tools
### macOS Built-in
* [pkgbuild](x-man-page://pkgbuild)
* [productbuild](x-man-page://productbuild)
* [installer](x-man-page://installer)
* [pkgutil](x-man-page://pkgutil)

### Third-party
* [munki-pkg](https://github.com/munki/munki-pkg)
* [The Luggage](https://github.com/unixorn/luggage)

## Other Resources
* [Packaging for Apple Administrators](https://books.apple.com/us/book/packaging-for-apple-administrators/id1173928620)
* [The Encylopaedia of Packages – MacDevOps YVR ’21](https://scriptingosx.com/mdo21/)
* [http://s.sudre.free.fr/Stuff/Ivanhoe/FLAT.html](http://s.sudre.free.fr/Stuff/Ivanhoe/FLAT.html)
* [Guidelines for Mac software packaging](https://wiki.afp548.com/index.php/Guidelines_for_Mac_software_packaging)
  * [Original "The Commandments of Packaging" on Posterous by Gary Larizza](https://web.archive.org/web/20130304070133/http://glarizza.posterous.com/the-commandments-of-packaging)
* [Mac Package Hall of Shame](https://macpkghallofshame.tumblr.com/)


## Other Thoughts
* It's ***macOS***. It is no longer *Mac OS X*, or *OS X*, or *OSX*.

  In the [Apple Style Guide](https://help.apple.com/applestyleguide)  there is  [a specific section](https://help.apple.com/applestyleguide/#/apsg72b28652?sub=apd246e83209)  about  _**Mac operating systems**_  that contains style guidelines for how to reference macOS versions.

* If you have a product targeting *Windows*, the Mac counterpart should be *macOS*, not *Mac*. *Mac* is the hardware, akin to *Dell*, *Lenovo*, or *HP*.
    * *Exception*: *mac* is an acceptable 3-letter short notation of *macOS* to match *win* for *Windows*
