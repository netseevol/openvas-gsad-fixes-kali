# GSAD login compatibility patch for GSA 28.3 on Kali Linux

This guide explains how to build the patched GSAD Debian package and install it on a separate Kali machine. The target machine only needs the binary `.deb` package and its runtime dependencies; it does not need a compiler or development packages.

The patch addresses the GSA 28 login failure where `POST /login` returns HTTP 404 and subsequent authenticated requests return HTTP 401 with `Token missing or bad`. It adds the newer login endpoint and session-duration handling to the Kali GSAD source package.

These instructions target the package combination diagnosed on 21 October 2026:

| Component | Version or requirement |
| --- | --- |
| Distribution | Kali Linux, `kali-rolling` |
| Original GSAD source | `24.16.0-1` |
| Patched GSAD package | `24.16.0-1+gsa28compat5` |
| GSA web interface | `greenbone-security-assistant=28.3.0-0kali1` |
| GVM runtime library | `libgvm22t64`, version `22.41.0-1` in the diagnosed package set |
| Binary package architecture used below | `amd64` |

**Build against GVM ABI 22 for this package combination.** The corresponding Kali `gvmd` package requires `libgvm22t64`. A GSAD package linked against `libgvm23` cannot resolve the reported dependency conflict with that `gvmd` package.

The source bundle can select either ABI 22 or ABI 23 and prefers ABI 23 when it is installed. Therefore, check the actual build result. Both builds have the same package version and filename.

The installation instructions assume GVM/OpenVAS is already configured on the target machine. Initial database, scanner, certificate, and feed setup are outside this guide.

**Build the package on the build machine**

1. Extract the source bundle and enter its directory:

   ```bash
   tar -xzf gsad-24.16.0-gsa28-backport.tar.gz
   cd gsad-24.16.0-gsa28-backport
   ```

2. Update the package index and install the build tools and ABI 22 runtime:

   ```bash
   sudo apt-get update
   sudo apt-get --no-remove install \
     build-essential dpkg-dev libgvm22t64
   ```

   If this fails because installed `libgvm23` conflicts with `libgvm22t64`, follow **Resolve the reported runtime library conflict** below, then repeat this step.

3. Install the build dependencies from the local source description:

   ```bash
   sudo apt-get build-dep --no-remove \
     ./sources/gsad_24.16.0-1+gsa28compat5.dsc
   ```

   The local `.dsc` supplies the build dependency list, so this command does not require a `deb-src` repository entry. Do not install `libgvm-dev` for this build: the source package includes the matching public GVM headers needed for compilation.

4. Build as an ordinary user:

   ```bash
   ./build.sh
   ```

   Confirm that the output includes:

   ```text
   Building against installed GVM ABI 22
   ```

   The script builds the Debian package, checks the binary and its dependencies, and runs an APT installation simulation on the build machine. It does not install the package. Its simulation must succeed before you use its suggested installation command.

   If the script selects ABI 23, correct the installed runtime library using the conflict instructions below and run it again. The normal clean phase clears the previous build configuration.

5. Explicitly verify that the resulting package uses ABI 22:

   ```bash
   ./verify-deb.sh \
     ./build/gsad_24.16.0-1+gsa28compat5_amd64.deb 22
   ```

   Expected final message:

   ```text
   PASS: ABI 22, matching dependency libgvm22t64, no bundled GVM libraries.
   ```

6. Create a checksum for the binary you will transfer:

   ```bash
   cd build
   sha256sum gsad_24.16.0-1+gsa28compat5_amd64.deb \
     > gsad_24.16.0-1+gsa28compat5_amd64.deb.sha256
   ```

   Copy the `.deb` and its `.sha256` file to the target machine. The source archive, `.dsc`, build directory, and development headers are not needed there.

**Install the binary package on the target machine**

Run the following commands in the directory containing the transferred `.deb` and checksum file.

1. Verify the transferred file and inspect its metadata:

   ```bash
   sha256sum --check gsad_24.16.0-1+gsa28compat5_amd64.deb.sha256

   dpkg --print-architecture

   dpkg-deb -f ./gsad_24.16.0-1+gsa28compat5_amd64.deb \
     Package Version Architecture Depends
   ```

   The checksum must pass. Both the machine and package architecture must be `amd64` for this binary. The package's `Depends` field must include `libgvm22t64` and must not require `libgvm23`.

   If the package requires `libgvm23`, obtain the ABI 22 build before continuing. Renaming the file or changing its dependency metadata does not change the library used by the binary.

2. Update the package index and preview installation:

   ```bash
   sudo apt-get update

   sudo apt-get -s --no-remove install \
     ./gsad_24.16.0-1+gsa28compat5_amd64.deb \
     greenbone-security-assistant=28.3.0-0kali1
   ```

   `-s` simulates the transaction without installing anything. `--no-remove` stops the operation if it would remove an installed package. This target-machine simulation is still required even if the build-machine simulation succeeded.

   If the simulation fails with the reported `libgvm23` conflict, use the conflict instructions below and repeat the simulation. If the requested GSA version is no longer available, inspect `apt-cache policy greenbone-security-assistant`; these instructions specifically target `28.3.0-0kali1`.

3. Once the simulation succeeds, install the binary and the matching GSA frontend:

   ```bash
   sudo apt-get --no-remove install \
     ./gsad_24.16.0-1+gsa28compat5_amd64.deb \
     greenbone-security-assistant=28.3.0-0kali1
   ```

   APT installs the required runtime dependencies automatically. Do not run `build.sh` or `apt-get build-dep` on a machine that is only receiving the binary.

4. Stop any manually started debug instance of GSAD, then restart the service:

   ```bash
   sudo systemctl restart gsad
   sudo systemctl --no-pager --full status gsad
   sudo apt-get check
   ```

   Restarting GSAD ends existing web sessions. Open the existing GSA URL in a new private browser window and log in. In browser developer tools, verify that `POST /login` succeeds and the subsequent authenticated `gmp?cmd=get_settings` request returns HTTP 200.

**Resolve the reported runtime library conflict**

Use this procedure only on a machine where installed `libgvm23` blocks `libgvm22t64`. Skip it when the ABI 22 runtime is already installed without conflicts.

In the diagnosed package set, `libgvm-dev=23.9.3-1` requires `libgvm23`, while the available `gvmd` requires `libgvm22t64`. The official `libgvm23` package declares a conflict that prevents these runtimes from being installed together.

Preview the explicit replacement:

```bash
sudo apt-get -s install libgvm-dev- libgvm23- libgvm22t64
```

For the reported machine, the simulation removed only `libgvm-dev` and `libgvm23` and installed `libgvm22t64`. Check the result on each machine. If it proposes removing GSAD, GVMD, the scanner, or other packages, stop and resolve that additional dependency before proceeding.

If the removal list contains only `libgvm-dev` and `libgvm23`, apply the same transaction:

```bash
sudo apt-get install libgvm-dev- libgvm23- libgvm22t64
```

The trailing `-` explicitly requests removal of those two packages. This one transaction intentionally omits `--no-remove`; keep that option on the build-dependency and GSAD installation commands.

Return to the build or installation step that previously failed. An ABI 23 binary must be rebuilt against ABI 22; replacing the runtime on the target alone does not fix that binary.

The separate APT list of packages that are "automatically installed and are no longer required" is not the removal list for this transaction. Do not run `apt autoremove` as part of these instructions, particularly on the build machine, where the list may contain headers needed for compilation.

**Scope and verification**

This is a local backport of the login endpoint and session-duration behavior. It is not a complete backport of every newer GSAD API. The patch was compiled and its session behavior tested with both GVM library ABIs; actual installation and browser login must be verified on the target Kali system using the steps above.

The package retains the `gvmd` dependency and uses normal Debian shared-library dependency generation. It does not bundle GVM runtime libraries or alter the official library packages' dependency declarations.

**References**

- [Kali GSAD package tracker](https://pkg.kali.org/pkg/gsad)
- [Kali GSA package tracker](https://pkg.kali.org/pkg/greenbone-security-assistant)
- [Upstream GSAD login endpoint change](https://github.com/greenbone/gsad/pull/403)
- [APT command reference](https://manpages.debian.org/trixie/apt/apt-get.8.en.html)
