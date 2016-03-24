# ec-specs-tool
Easy to use testing tool for writing specifications and acceptance tests

# Latest release

https://github.com/electric-cloud/ec-specs-tool/releases/latest

# Steps to use Specs 

1. Unzip the file ec-specs-tool.zip into a directory, say 'c:\tools\ec-specs-tool'
2. The directory structure under 'c:\tools\ec-specs-tool' should look like this:
 - /samples
 - /utils
 - ec-specs
 - ec-specs.bat 
3. Make sure that Java 8 is available on the system as either a JDK or a JRE. Open a command terminal and type the following command to confirm the java version:

  > java -version

  If the command does not return java 8 in the version information, then make sure JDK 8 or JRE 8 is installed and the PATH environment variable is set to point to the installed Java 8 version.

4. Open a command terminal and type the following to run the samples/BasicSpec.groovy specification test.
> c:\tools\ec-specs-tool\ec-specs.bat c:\tools\ec-specs-tool\samples\BasicSpec.groovy

  ```
  On *nix system, we need to grant execute permission on the ex-spec script first, 
  so use the following commands to run the samples/BasicSpec.groovy specification test, 
  assuming the zip file is extracted in /opt/tools/ec-spec-tool:
  a) chmod +x /opt/tools/ec-spec-tool/ec-spec
  b) /opt/tools/ec-spec-tool/ec-spec /opt/tools/ec-spec-tool/ec-spec/samples/BasicSpec.groovy
  ```
5. The tool will download the gradle files (one time deal), then compile the *BasicSpec.groovy* file before running the tests against the ElectricFlow server on localhost.
 
  You can use the following command to point to a server on another system:
  > c:\tools\ec-specs-tool\ec-specs.bat c:\tools\ec-specs-tool\samples\BasicSpec.groovy --server hostname
 
That's it!
You can use the specification tests in c:\tools\ec-specs-tool\samples as reference.

# Offline mode

If your system does not have internet access, you need to take the additional steps

* Download https://services.gradle.org/distributions/gradle-2.9-bin.zip
* Copy it in c:\tools\ec-specs-tool\distributions
* Then change in the C:\ec-specs-tool\utils\graddle\wrapper\gradle-wrapper.properties
  * distributionUrl=https\://services.gradle.org/ditributions/gradle-2.9-bin.zip into
  * distributionUrl=file\:///C:/ec-specs-tool/ditributions/gradle-2.9-bin.zip

# Spock Reference Documentation
http://spockframework.github.io/spock/docs/1.0/index.html

Recommended:
* Chapter 3. Spock Primer
* Chapter 4. Data Driven Testing
