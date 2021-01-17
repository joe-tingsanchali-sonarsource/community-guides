As explained in our [Test Coverage](https://docs.sonarqube.org/latest/analysis/coverage/) documentation, SonarQube/SonarCloud **does not** run tests or generate reports, but **imports** pre-generated reports from another source. The goal of this guide is show how to troubleshoot and correct typical code coverage import issues into SonarQube/SonarCloud for C# and VB.NET languages. If you are interested in troubleshooting unit test results and execution reporting instead, see "Import unit test results" section of [[Coverage & Test Data] Generate Reports for C#, VB.net](https://community.sonarsource.com/t/coverage-test-data-generate-reports-for-c-vb-net/9871).

# Problems you may have

# Prerequisites

We support the following code coverage tools:

* Visual Studio Code Coverage
* dotCover
* OpenCover/Coverlet

If you do not use any of the above code coverage tools, code coverage import will not work.

Understand the basics of...
* SonarScanner for .NET (formerly known as SonarScanner for MSBuild) by reading its [documentation](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/)
* If you're using Azure DevOps...
  * Understand the integration between Azure DevOps and SonarCloud by reading [this Azure DevOps Lab](https://azuredevopslabs.com//labs/vstsextend/sonarcloud/).
  * How code coverage test files are automatically detected as explained in the [Code Coverage](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/#header-5) section for SonarScanner for .NET:
    * Auto-search for `.trx` files
    * `.coverage` auto-converts to `.coveragexml` via CodeCoverage.exe
  * Know how to construct your pipeline correctly
    * Install appropriate platform dependencies
    * “Prepare Analysis Configuration” task contains proper code coverage parameters
    * "Run Code Analysis" and "Publish Quality Gate Result" tasks are also added to your pipeline

# Troubleshooting steps

* Are you using a valid version of your code coverage tool?

* Are the reports in the right format?
  * Verify which file type matches which code coverage tool

* Do the reports even exist?
  * If you see the following, then ensure the report generation for your code coverage tool actually produced a file and has content in it.
 
     ```
     WARN: The Code Coverage report doesn’t contain any coverage data for the included files.
     ```
 
    * Verify Indexed Paths Compared to Coverage File Paths
 (see https://community.sonarsource.com/t/code-coverage-is-not-shown-in-the-sonarcloud-io/15218/6)
      * The `/p:PathMap` option of MSBuild is not supported for coverage files parsed in SonarQube or SonarCloud.
* Do the reports contain valid information for the expected set of files? (give examples)

* Did you specify the correct report path property to import the report at the right step?
When importing code coverage reports, you need to ensure that you specify the report path with your associated code coverage tool's property for the SonarScanner.MSBuild.exe `begin` command, as detailed in [Test Coverage](https://docs.sonarqube.org/latest/analysis/coverage/):

    * `sonar.cs.vscoveragexml.reportsPaths`
      * Path to Visual Studio code coverage report for C#. Multiple paths may be comma-delimited, or included via wildcards.
      * Ensure that you have converted the code coverage report from Binary into XML (see "Convert the Code Coverage Report from Binary into XML" section on our [[Coverage & Test Data] Generate Reports for C#, VB.net](https://community.sonarsource.com/t/coverage-test-data-generate-reports-for-c-vb-net/9871) Community guide)
    * `sonar.cs.dotcover.reportsPaths`
      * Path to dotCover coverage report for C#.
    * `sonar.cs.opencover.reportsPaths`
      * Path to OpenCover coverage report for C#.
    * `sonar.vbnet.vscoveragexml.reportsPaths`
      * Path to Visual Studio code coverage report for VB.NET.
    * `sonar.vbnet.dotcover.reportsPaths`
      * Path to dotCover coverage report for VB.NET.
    * `sonar.vbnet.opencover.reportsPaths`
      * Path to OpenCover coverage report for VB.NET.

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
    
    ```-d:"sonar.cs.vscoveragexml.reportsPaths=**/*.coveragexml”``` or
    
    ```-d:"sonar.cs.dotcover.reportsPaths=**/*.html"``` or
    
    ```-d:"sonar.vbnet.opencover.reportsPaths=**/someNameCodeCoverage.xml"```
    
    ✅ Values can be comma separated
    
    ```/d:"sonar.cs.vscoveragexml.reportsPaths=coverage/joe*.coveragexml,otherdir/olivier*.coveragexml"```
    
    ✅ `-` and `/` are valid flag markers
    
    ```/d:"sonar.verbose=true``` or
    ```-d:"sonar.verbose=true"``` 
  
    ✅ Report path is relative to the directory where you run SonarScanner for MSBuild 
    
    ```-d:"sonar.cs.vscoveragexml.reportsPaths=coveragexml"```
    
    **Wrong** ❌
    
    ❌  Don't point at directories. **How to fix:** point to files.
    
    ```-d:"sonar.cs.vscoveragexml.reportsPaths=coverage/"```
        
    ❌ Don't use absolute paths. **How to fix:** favor paths relative to the project root directory.
    
    ```-d:"sonar.cs.vscoveragexml.reportsPaths=C:\users\joe\project\coverage\*.xml"```
        
    ❌ Don't point files that are not under the project root directory. **How to fix:** always generate the coverage report under the directory where you run the command from.
  
    ```-d:"sonar.cs.vscoveragexml.reportsPaths=..\coverage\*.xml"```

* Do you know how to get debug logs?

    * Add `/d:"sonar.verbose=true"` to the...
        * SonarScanner.MSBuild.exe `begin` command to get more detailed logs
            * For example: `SonarScanner.MSBuild.exe begin /k:"MyProject" /d:"sonar.verbose=true"`
        * `SonarQubePrepare` or `SonarCloudPrepare` task's `extraProperties` argument if you are using Azure DevOps
            * For example:
              ```markdown
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
    * The parsing and import of code coverage happens during the Scanner for MSBuild `end` command (a.k.a. the Azure DevOps `SonarQubeAnalyze` or `SonarCloudAnalyze` task).

# Specific problems

* Prematurely running report generation

    The report generation process must be executed after the build step and before the end scanner end step. The following steps detail importing .NET reports:

    1. Run the SonarScanner.MSBuild.exe `begin` command, specifying the absolute path where the reports will be available using the /d:propertyKey="path" syntax ("propertyKey" depends on the tool)
    2. Build your project using MSBuild
    3. Run your test tool, instructing it to produce a report at the same location specified earlier to the MSBuild SonarQube Runner (How to generate reports with different tools)
    4. Run the SonarScanner.MSBuild.exe `end` command

* Projects being considered as test projects by the Scanner for MSBuild

    Read the [ProjectCategorisation](https://github.com/SonarSource/sonar-scanner-msbuild/wiki/Analysis-of-product-projects-vs.-test-projects#project-categorisation) section to better understand if the project not having code coverage may be considered a test project.

* Not running all commands from the same folder

    All commands, including the code coverage generation, must be run from the same folder.
    
    To check which is the base path look for this log:
    
    ```
    DEBUG: The current user dir is '<PATH>'.
    ```
    
    To check the path of the indexed files look for this log:
    
    ```
    DEBUG: '<PATH\FILE.cs>' indexed with language 'cs'
    ```
    
    Then if the paths inside the coverage file differ from the path of the indexed files, look for this log:
    
    ```
    INFO: Parsing the OpenCover report <FILE PATH>
    DEBUG: Skipping file with path '<PATH\FILE.cs>' because it is not indexed or does not have the supported language.
    ```
