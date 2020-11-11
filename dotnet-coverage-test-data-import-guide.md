As explained in our [Test Coverage](https://docs.sonarqube.org/latest/analysis/coverage/) documentation, SonarQube/SonarCloud does not run tests or generate reports, but it imports pre-generated reports from another source. The goal of this guide is show how to troubleshoot and correct typical code coverage import issues into SonarQube/SonarCloud for C# and VB.NET languages. If you are interested in troubleshooting unit test results and execution reporting instead, see "Import unit test results" section of [[Coverage & Test Data] Generate Reports for C#, VB.net](https://community.sonarsource.com/t/coverage-test-data-generate-reports-for-c-vb-net/9871).

# Requirements

First, we support the following code coverage tools:

* Visual Studio Code Coverage
* dotCover
* OpenCover/Coverlet
* NCover3

If you do not use any of the above code coverage tools, code coverage import will not work.

Second, when importing code coverage reports, you need to ensure that you specify the report path with your associated code coverage tool's property for the SonarScanner.MSBuild.exe `begin` command, as detailed in [Test Coverage](https://docs.sonarqube.org/latest/analysis/coverage/):

* sonar.cs.vscoveragexml.reportsPaths
    * Path to Visual Studio Code Coverage report. Multiple paths may be comma-delimited, or included via wildcards.
* sonar.cs.dotcover.reportsPaths
    * Path to dotCover coverage report.
* sonar.cs.opencover.reportsPaths
    * Path to OpenCover coverage report.
* sonar.vbnet.dotcover.reportsPaths
    * Path to dotCover coverage report.
* sonar.vbnet.opencover.reportsPaths
    * Path to OpenCover coverage report.

Third, understand the basics of 
SonarScanner for MSBuild by reading its docs [here](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner-for-msbuild/), in case you are using it in your pipeline.
Azure Devops - understand the integration between Azure DevOps and SonarCloud by reading [this guideline](https://azuredevopslabs.com//labs/vstsextend/sonarcloud/)..

#### How to Get Debug Level Logs

Add `/d:sonar.verbose=true` or `/d:"sonar.verbose=true"` to the
 SonarScanner.MSBuild.exe `begin` command to get more detailed logs. Usually, you will find your error here!
`SonarCloudPrepare` task in case you are using Azure DevOps

For example:
`SonarScanner.MSBuild.exe begin /k:“MyProject” /d:sonar.verbose=true`

The parsing and import of code coverage happens during the Scanner for MSBuild `end` command (the Azure Devops `SonarCloudAnalyze` task).

# Typical Code Coverage Import Fixes

#### Ensure Code Coverage Property Report Path

Make sure you understand the regular expression for matching files and directories:
|Symbol|Meaning|
|---|---|
|?|a single character|
|*|any number of characters|
|**|any number of directories|

Here is a resource to understand some other examples: [Pattern matching guide](https://confluence.atlassian.com/fisheye/pattern-matching-guide-960155410.html)

Examples of Correct Usage:

Right :o:

:o: `-d:"sonar.cs.vscoveragexml.reportsPaths=**/*.coveragexml”`

:o: `/d:"sonar.cs.vscoveragexml.reportsPaths=**/jojo_jt 2020-10-14 19_02_26.coveragexml”`(`-` and `/` are valid flag markers)

:o: `-d:"sonar.cs.vscoveragexml.reportsPaths=coveragexml”` (assuming you are running SonarScanner.MSBuild.exe in same directory as your *.coveragexml file)

:o: `-d:"sonar.cs.dotcover.reportsPaths=**/*.html"`

:o: `-d:"sonar.vbnet.opencover.reportsPaths=**/someNameCodeCoverage.xml"`

Wrong :x:

?

#### Verify Coverage File Exists and Has Content
If you see the following, then ensure the report generation for your code coverage tool actually produced a file and has content in it.

```
WARN: The Code Coverage report doesn’t contain any coverage data for the included files.
```

#### Verify Indexed Paths Compared to Coverage File Paths
(see https://community.sonarsource.com/t/code-coverage-is-not-shown-in-the-sonarcloud-io/15218/6)

#### Prematurely running report generation

The report generation process must be executed after the begin step and before the end MSBuild command. The following steps detail importing .NET reports:

1. Run the SonarScanner.MSBuild.exe `begin` command, specifying the absolute path where the reports will be available using the /d:propertyKey="path" syntax ("propertyKey" depends on the tool)
2. Build your project using MSBuild
3. Run your test tool, instructing it to produce a report at the same location specified earlier to the MSBuild SonarQube Runner (How to generate reports with different tools)
4. Run the SonarScanner.MSBuild.exe `end` command

#### Projects being considered as test projects by the Scanner for MSBuild

Read the [ProjectCategorisation](https://github.com/SonarSource/sonar-scanner-msbuild/wiki/Analysis-of-product-projects-vs.-test-projects#project-categorisation) section to better understand if the project not having code coverage may be considered a test project.

#### Not running all commands from the same folder

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
