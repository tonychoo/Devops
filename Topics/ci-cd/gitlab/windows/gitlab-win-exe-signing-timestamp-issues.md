# Troubleshooting GitLab CI/CD Pipeline Signing Issues on Windows Builds

## Issue Overview

Recently, our GitLab CI/CD pipeline for Windows builds began failing during the code signing phase. The pipeline would complete the build process successfully but encounter errors during the executable signing step, preventing deployment of our application.

The error appeared in the GitLab pipeline logs as follows:

```
** Visual Studio 2019 Developer Command Prompt v16.8.1
** Copyright (c) 2020 Microsoft Corporation
**********************************************************************
Done Adding Additional Store
Error information: "Error: SignerSign() failed." (-2146869243/0x80096005)
SignTool Error: An unexpected internal error has occurred.
ScSignTool Error: The timestamp signature and/or certificate could not be verified or is malformed. (0x80096005)
```

The Python stack trace revealed the error occurred in the `sign_file` function within our build scripts:

```python
subprocess.CalledProcessError: Command 'cmd /c ""%VS160COMNTOOLS%\VsDevCmd.bat" && 
"[path]\\ScSignTool.exe" -pin [pin] sign 
/sha1 [sha1 value] /fd SHA256 
/tr http://timestamp.digicert.com /a  [exe path]"' 
returned non-zero exit status 1.
```

## Investigation Process

### Step 1: Local Reproduction

The first step in our troubleshooting approach was to isolate the issue by reproducing it outside the CI/CD environment. We ran the ScSignTool.exe locally with the same parameters to determine if the error was environment-specific or related to the tool configuration itself.

After multiple attempts with the local signing tool, we were able to successfully run the command in some instances, suggesting the issue might be related to specific parameters or configuration rather than a fundamental problem with the certificate or signing infrastructure.

### Step 2: Documentation Research

With some evidence that the tool could work under certain conditions, we consulted the official Microsoft documentation for signtool at https://learn.microsoft.com/en-us/windows/win32/seccrypto/signtool.

A critical piece of information was found in the documentation notes:

> "The Windows SDK, Windows Hardware Lab Kit (HLK), Windows Driver Kit (WDK), and Windows Assessment and Deployment Kit (ADK) builds 20236 and later require that you specify the digest algorithm. The SignTool sign command requires the file digest algorithm option (/fd) and the time stamp digest algorithm option (/td) during signing and time stamping, respectively.
> 
> If /fd isn't specified during signing and if /td isn't specified during time stamping, the command throws a warning, error code 0, initially. In later versions of SignTool, the warning becomes an error. We recommend SHA256. It's considered to be more secure than SHA1 by the industry."

### Step 3: Root Cause Identification

Upon reviewing our command parameters, we noticed our configuration included the file digest algorithm option (/fd SHA256) but was missing the time stamp digest algorithm option (/td). 

The error message "The timestamp signature and/or certificate could not be verified or is malformed" aligned with the documentation's warning about missing the /td parameter, confirming this as the likely cause of our pipeline failures.

## Solution Implementation

Based on our findings, we modified the signing command in our build script to include the time stamp digest algorithm option:

```python
# Original command
cmd = f'cmd /c "{vsdev_cmd} && "{sign_tool}" -pin {sign_pin} sign /sha1 {sign_thumbprint} /fd SHA256 /tr {timestamp_server} /a  "{file_path}""'

# Updated command
cmd = f'cmd /c "{vsdev_cmd} && "{sign_tool}" -pin {sign_pin} sign /sha1 {sign_thumbprint} /fd SHA256 /td SHA256 /tr {timestamp_server} /a  "{file_path}""'
```

After implementing this change, we ran the ScSignTool.exe locally again and observed successful completion of the signing process.

## Verification and Deployment

We committed the updated build script to our repository and triggered a new pipeline run. The build completed successfully with the executable properly signed, confirming our solution resolved the issue.

## Key Learnings

1. **Parameter Requirements Evolution**: Microsoft's signing tools have evolved to require explicit specification of both file and timestamp digest algorithms for security reasons.

2. **Error Interpretation**: The generic "SignerSign() failed" error can be caused by missing parameters, not just by certificate issues.

3. **Documentation Importance**: While error messages can be cryptic, official documentation often contains critical information about parameter requirements and best practices.

4. **Local Testing Value**: Reproducing the issue locally allowed us to isolate the problem and test solutions efficiently before implementing them in the CI/CD pipeline.

This troubleshooting case demonstrates the importance of careful parameter configuration when working with security tools in automated build environments, especially as security requirements continue to evolve.