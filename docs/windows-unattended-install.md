# Unattended Windows installation

Use this checklist when delivering the Windows agent through a deployment tool.
The installer options are described in the
[Windows installation reference](https://openuem.eu/docs/Installation/Agent/windows/).
This example is for a fresh installation; handle upgrades separately.

## Prepare the installation

1. Use an elevated PowerShell session or a deployment job running as SYSTEM.
2. Select an installer version and architecture from
   [Windows downloads](https://openuem.eu/docs/Downloads/windows/). Record its
   version, download URL and published SHA256 in the deployment definition.
3. Prepare a staging directory accessible only to SYSTEM and administrators.
   Restrict access **before** copying credentials into it. Distribute the installer
   and enrollment files through an authenticated private deployment channel.
4. Obtain `ca.cer`, `agent.cer`, `agent.key` and `sftp.cer` from the administrator
   of the intended OpenUEM environment. `agent.key` is a private credential.
   Never distribute the CA signing key or publish enrollment files in a repository.
5. Confirm that the endpoint can resolve and reach its configured NATS server.

Importing a CA into a browser trust store only addresses that browser's trust.
The agent still needs the certificate and key files specified to its installer.

## Run the installer

Set the paths, server and checksum below for your environment. Use the checksum
for the **selected installer version**, rather than a checksum from an older example.
The sample paths contain spaces intentionally.

```powershell
$ErrorActionPreference = 'Stop'
$installer = 'C:\OpenUEM Bootstrap\openuem-agent-setup.exe'
$credentials = 'C:\OpenUEM Bootstrap\enrollment'
$natsServer = 'uem.example.com:4433'
$expectedSha256 = 'REPLACE_WITH_THE_PUBLISHED_SHA256'

if (Get-Service -Name 'openuem-agent' -ErrorAction SilentlyContinue) {
    throw 'An agent service already exists. Review the existing installation first.'
}
if ($expectedSha256 -notmatch '^[0-9a-fA-F]{64}$') {
    throw 'Set the published SHA256 for the selected installer.'
}
if ((Get-FileHash -LiteralPath $installer -Algorithm SHA256).Hash -ne $expectedSha256) {
    throw 'Installer checksum mismatch.'
}
if ((Get-AuthenticodeSignature -LiteralPath $installer).Status -ne 'Valid') {
    throw 'Installer signature validation failed. Investigate before continuing.'
}
foreach ($file in @('ca.cer', 'agent.cer', 'agent.key', 'sftp.cer')) {
    if (-not (Test-Path -LiteralPath (Join-Path $credentials $file) -PathType Leaf)) {
        throw "Missing enrollment file: $file"
    }
}

$installerArguments = @(
    '/VERYSILENT', '/SUPPRESSMSGBOXES', '/NORESTART'
    "/server=$natsServer"
    "/cacert=`"$credentials\ca.cer`""
    "/cert=`"$credentials\agent.cer`""
    "/key=`"$credentials\agent.key`""
    "/sftpcert=`"$credentials\sftp.cer`""
)
$process = Start-Process -FilePath $installer -ArgumentList ($installerArguments -join ' ') -Wait -PassThru
if ($process.ExitCode -ne 0) {
    throw "Installer exited with code $($process.ExitCode). Inspect the installation before retrying."
}
Get-Service -Name 'openuem-agent'
```

The `/server` value is the NATS endpoint, which may differ from the console's
HTTPS address. For organization and site placement, append `/tenantid=<number>`
and `/siteid=<number>` together to the argument list. Obtain the actual numeric
IDs from the console. Omitting both uses the installer's default placement.

The silent switches suppress installer interaction and automatic reboot. A
nonzero result requires investigation; this example does not automatically
retry or treat a reboot-required result as success.

## Check admission, reporting and credentials

A successful installer exit and a running service are only local checks.

1. Inspect the installed credentials, including their effective file permissions.
   For a default installation, start with:

   ```powershell
   icacls 'C:\Program Files\OpenUEM Agent\certificates\agent.key'
   ```

   Confirm that ordinary users cannot read the private key. For a SYSTEM service,
   SYSTEM and administrators should retain access. If installation fails, inspect
   any credentials left in the destination as well as the staging directory.
2. In the console, find the expected endpoint, verify its organization/site and
   admit it. Confirm a recent inventory report from that endpoint.
3. Remove the source enrollment files and deployment archive when they are no
   longer required. Verify removal, and report cleanup errors without discarding
   the original installation error. Preserve the credentials used by the
   installed service.

For troubleshooting, inspect the installed agent's logs. Restrict access to logs
and review their contents before sharing them. Keep certificate validation enabled;
investigate trust, server name, time and network failures rather than bypassing TLS.

## Validation scope

This document is an installation checklist, not an automated enrollment service.
The example requires your Windows acceptance checks, including process argument
quoting, Authenticode, permissions, service startup, admission and reporting.
It does not establish per-device enrollment credentials, customer isolation or
production readiness. Validate those separately before fleet deployment.
