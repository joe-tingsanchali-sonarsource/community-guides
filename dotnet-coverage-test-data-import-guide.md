As explained in our [Test Coverage](https://docs.sonarqube.org/latest/analysis/coverage/) documentation, SonarQube/SonarCloud **does not** run tests or generate reports, but **imports** pre-generated reports from another source. The goal of this guide is show how to troubleshoot and correct typical code coverage import issues into SonarQube/SonarCloud for C# and VB.<span></span>NET languages. If you are interested in troubleshooting unit test results and execution reporting instead, see "Import unit test results" section of [[Coverage & Test Data] Generate Reports for C#, VB.net](https://community.sonarsource.com/t/coverage-test-data-generate-reports-for-c-vb-net/9871).

# Prerequisites

We support the following code coverage tools:

* Visual Studio Code Coverage
* dotCover
* OpenCover/Coverlet

If you do not use any of the above code coverage tools, code coverage import will not work.

Understand the basics of...
* SonarScanner for .NET (formerly known as SonarScanner for MSBuild) by reading its [documentation](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/)
* Integration with Azure DevOps/TFS environment (if you are using Azure DevOps/TFS)
  * Understand the link between Azure DevOps and SonarCloud by reading this [SonarCloud Azure DevOps Lab](https://azuredevopslabs.com//labs/vstsextend/sonarcloud/).
  * How code coverage test files are automatically detected as explained in the [Code Coverage](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/#header-5) section for SonarScanner for .NET:
    * Auto-search for `.trx` files in any TestResults folder located under the $Build.SourcesDirectory path (otherwise, check $Agent.TempDirectory path)
    * Auto-search for CodeCoverage.exe path by searching in `VsTestToolsInstallerInstalledToolLocation` environment variable (set by the "VsTestToolsPlatformInstaller" task or by the user)
    * `.coverage` auto-converts to `.coveragexml` via CodeCoverage.exe

  * Know how to construct your pipeline correctly
    * Install appropriate platform dependencies
    * “Prepare Analysis Configuration” task contains proper code coverage parameters
    * "Run Code Analysis" and "Publish Quality Gate Result" tasks are also added to your pipeline

# Troubleshooting Steps

* Are you using a valid version of your code coverage tool?

* Are the reports in the right format?
  * Verify which file type matches which code coverage tool

* Do the reports even exist?
  * If you see the following, then ensure the report generation for your code coverage tool actually produced a file and has content in it.

    > ```
    > WARN: The Code Coverage report doesn’t contain any coverage data for the included files.
    > ```
 
  * Verify indexed paths compared to coverage file paths
 (see https://community.sonarsource.com/t/code-coverage-is-not-shown-in-the-sonarcloud-io/15218/6)
    * The `/p:PathMap` option of MSBuild is not supported for coverage files parsed in SonarQube or SonarCloud.
* Do the reports contain valid information for the expected set of files? (give examples)

* Did you specify the correct report path property to import the report at the right step?
  * When importing code coverage reports, you need to ensure that you specify the report path with your associated code coverage tool's property for the SonarScanner.MSBuild.exe or dotnet `begin` command, as detailed in [Test Coverage](https://docs.sonarqube.org/latest/analysis/coverage/):

    * `sonar.cs.vscoveragexml.reportsPaths`
      * Path to Visual Studio Code coverage report for C#. Multiple paths may be comma-delimited, or included via wildcards.
      * Ensure that you have converted the code coverage report from Binary into XML (see "Convert the Code Coverage Report from Binary into XML" section on our [[Coverage & Test Data] Generate Reports for C#, VB.net](https://community.sonarsource.com/t/coverage-test-data-generate-reports-for-c-vb-net/9871) Community guide)
    * `sonar.cs.dotcover.reportsPaths`
      * Path to dotCover coverage report for C#.
    * `sonar.cs.opencover.reportsPaths`
      * Path to OpenCover coverage report for C#.
    * `sonar.vbnet.vscoveragexml.reportsPaths`
      * Path to Visual Studio Code Coverage report for VB.<span></span>NET. Multiple paths may be comma-delimited, or included via wildcards. See [Notes on importing .NET reports](https://docs.sonarqube.org/latest/analysis/coverage/#header-3) for more information.
    * `sonar.vbnet.dotcover.reportsPaths`
      * Path to dotCover coverage report for VB.<span></span>NET.
    * `sonar.vbnet.opencover.reportsPaths`
      * Path to OpenCover coverage report for VB.<span></span>NET.
  * SonarScanner for .NET has automatic detection of `sonar.cs.vscoveragexml.reportsPaths` and `sonar.cs.vstest.reportsPaths` if the proper `.trx` and `.coverage` files have been found in specific folders. See [Code Coverage](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/#header-5) section for more information on these analysis parameters can be auto-set.


* Did you specify the correct path to your code coverage file?

  * Make sure you understand the regular expression for matching files and directories:

    |Symbol|Meaning|
    |---|---|
    |?|a single character|
    |*|any number of characters|
    |**|any number of directories|
    
  * Here is a resource to understand some other examples: [Pattern matching guide](https://confluence.atlassian.com/fisheye/pattern-matching-guide-960155410.html)

  * Examples of Correct and Incorrect Usage
    
    **Right** ✅
    
    ✅ Do use glob patterns to match the ultimately created
    `-d:"sonar.cs.vscoveragexml.reportsPaths=**/*.coveragexml"` or
    `-d:"sonar.cs.dotcover.reportsPaths=**/*.html"` or
    `-d:"sonar.vbnet.opencover.reportsPaths=**/someNameCodeCoverage.xml"`
    
    ✅ Values can be comma separated
    `/d:"sonar.cs.vscoveragexml.reportsPaths=coverage/joe*.coveragexml,otherdir/olivier*.coveragexml"`
    
    ✅ `-` and `/` are valid flag markers
    `/d:"sonar.verbose=true` or `-d:"sonar.verbose=true"` 
  
    ✅ Report path is relative to the directory where you run SonarScanner for 'MSBuild'
    `-d:"sonar.cs.vscoveragexml.reportsPaths=coveragexml"`
    
    **Wrong** ❌
    
    ❌  Don't point at directories. **How to fix:** point to files.
    `-d:"sonar.cs.vscoveragexml.reportsPaths=coverage/"`
        
    ❌ Don't use absolute paths. **How to fix:** favor paths relative to the project root directory.
    `-d:"sonar.cs.vscoveragexml.reportsPaths=C:\users\joe\project\coverage\*.xml"`
        
    ❌ Don't point files that are not under the project root directory. **How to fix:** always generate the coverage report under the directory where you run the command from.
    `-d:"sonar.cs.vscoveragexml.reportsPaths=..\coverage\*.xml"`

* Do you know how to get debug logs?
  * Add `/d:"sonar.verbose=true"` to the...
    * SonarScanner.MSBuild.exe or dotnet `begin` command to get more detailed logs
      * For example: `SonarScanner.MSBuild.exe begin /k:"MyProject" /d:"sonar.verbose=true"`
    * "SonarQubePrepare" or "SonarCloudPrepare" task's `extraProperties` argument if you are using Azure DevOps
      * For example:
        ```yml
        # Applies to SonarQubePrepare as well
        - task: SonarCloudPrepare@1
            inputs:
              SonarCloud: 'sonarcloud'
              organization: 'zucchinibreadco'
              scannerMode: 'MSBuild'
              projectKey: 'zuchhinibreadco_sonar-scanning-someconsoleapp'
              projectName: 'sonar-scanning-someconsoleapp'
              extraProperties: |
                sonar.cs.vstest.reportsPaths=$(Agent.TempDirectory)\**\*.trx
                sonar.cs.vscoveragexml.reportsPaths=$(Agent.TempDirectory)\**\*.coveragexml
                sonar.verbose=true
        ``` 
  * The parsing and import of code coverage happens during the Scanner for MSBuild `end` command (a.k.a. the Azure DevOps "SonarQubeAnalyze" or "SonarCloudAnalyze" task).

# Specific Problems

* **Prematurely running report generation**

  The report generation process must be executed after the build step and before the scanner `end` step. The following steps detail importing .NET reports:

  1. Run the SonarScanner.MSBuild.exe (or dotnet) `begin` command, specifying the absolute path where the reports will be available using the `/d:propertyKey="path"` syntax ("propertyKey" depends on the tool)
  2. Build your project using MSBuild (or dotnet)
  3. Run your test tool, instructing it to produce a report at the same location specified earlier to the MSBuild/dotnet SonarQube runner (see [How to generate reports with different tools](https://community.sonarsource.com/t/coverage-test-data-generate-reports-for-c-vb-net/9871))
  4. Run the SonarScanner.MSBuild.exe (or dotnet) `end` command

* **Projects being considered as test projects by the Sonar Scanner for .NET**

  Read the [ProjectCategorisation](https://github.com/SonarSource/sonar-scanner-msbuild/wiki/Analysis-of-product-projects-vs.-test-projects#project-categorisation) section to better understand if the project not having code coverage may be considered a test project.

* **Not running all commands from the same folder**

  All commands, including the code coverage generation, must be run from the same folder.
  
  To check which is the base path look for this log:
  
  ```text
  DEBUG: The current user dir is '<PATH>'.
  ```
  
  To check the path of the indexed files look for this log:
  
  ```text
  DEBUG: '<PATH\FILE.cs>' indexed with language 'cs'
  ```
  or
  ```text
  DEBUG: '<PATH\FILE.vb>' indexed with language 'vb'
  ```
  
  Then if the paths inside the coverage file differ from the path of the indexed files, look for this log:
  
  ```text
  INFO: Parsing the OpenCover report <FILE PATH>
  DEBUG: Skipping file with path '<PATH\FILE.cs>' because it is not indexed or does not have the supported language.
  ```
  or
  ```text
  INFO: Parsing the OpenCover report <FILE PATH>
  DEBUG: Skipping file with path '<PATH\FILE.vb>' because it is not indexed or does not have the supported language.
  ```
