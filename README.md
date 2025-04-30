# Repo manifest for OP-TEE development
This git contains repo manifests to be able to clone all source code needed to
be able to setup a full OP-TEE developer build.

All official OP-TEE documentation has moved to http://optee.readthedocs.io. The
information that used to be here in this git can be found under [manifests].

// OP-TEE core maintainers
# Wrapper qemu
## Combination of OP-TEE and PSACrypto

This Project combines the PSA Crypto implementation of mbedtls with OP-TEE.
In this Branch PSA Crypto and Op-TEE are combined through a wrapper, which maps the PSA Crypto function calls to OP-TEE function calls.
These are then directed normally through the Op-TEE architecture resolved through OP-TEE library methods.


## Setup

This Project requires the same setup as normal OP-TEE (Keychain, setup, QEMU...).
here is a setup used for OP-TEE on ubuntu 22.04 with a QEMU setup
### Install Dependecies

```sh
apt update && apt upgrade -y
 
apt install -y \
    adb \
    acpica-tools \
    autoconf \
    automake \
    bc \
    bison \
    build-essential \
    ccache \
    cpio \
    cscope \
    curl \
    device-tree-compiler \
    e2tools \
    expect \
    fastboot \
    flex \
    ftp-upload \
    gdisk \
    git \
    libattr1-dev \
    libcap-ng-dev \
    libfdt-dev \
    libftdi-dev \
    libglib2.0-dev \
    libgmp3-dev \
    libhidapi-dev \
    libmpc-dev \
    libncurses5-dev \
    libpixman-1-dev \
    libslirp-dev \
    libssl-dev \
    libtool \
    libusb-1.0-0-dev \
    make \
    mtools \
    netcat \
    ninja-build \
    python3-cryptography \
    python3-pip \
    python3-pyelftools \
    python3-serial \
    python3-tomli \
    python-is-python3 \
    rsync \
    swig \
    unzip \
    uuid-dev \
    wget \
    xdg-utils \
    xsltproc \
    xterm \
    xz-utils \
    zlib1g-dev
```
### Install the repo tool
```sh
curl https://storage.googleapis.com/git-repo-downloads/repo > /bin/repo && chmod a+x /bin/repo
```
### Initialize the OP-TEE Repo

```sh
mkdir optee
cd  optee
repo init -u https://github.com/lennard2000/manifest.git -m wrapper_qemu.xml && repo sync -j10
```
### Download the OP-TEE Toolchains
```sh
cd build
make -j3 toolchains
make -j$(nproc) check
```
After this the Setup should be completed.

## Commands

This Project uses the default OP-TEE commands.

Here is a list of commands used to compile and run the Project
- To compile the project:
```sh
make
```
- To compile and run the project (using QEMU):
```sh
make run
```
- To run the project without compiling (only use after compiling at least once):
```sh
make run-only
```
> these Commands only work in the `build` directory, so before executing them `cd optee/buid` may need to be executed
## Testing
To test the project, simply compile and start it. This will automatically run the default OP-TEE tests. If compilation and basic tests succeed, you can assume the OP-TEE implementation is working.
```sh
make
```
To verify OP-TEE with PSA Crypto, run the project and execute the example Trusted Applications:
```sh
make run
optee_example_acipher
optee_example_aes
optee_example_hello_world
```
*Note*: These commands correspond to the default OP-TEE examples, but now include PSA tests from the mbedtls repository, slightly modified for OP-TEE. These calls should not produce output; they perform self-validation using constant values. Errors or mismatches will trigger error messages.

> output, the command line of the normal world after executing the commands
 ```
# optee_example_aes 
# optee_example_acipher
# optee_example_hello_world
 ```
```
make run
xtest
```
This runs the OP-TEE xtest suite. You can also run specific tests by specifying the test number:
```sh
xtest <test_number>
```


## Required changes to PSA files

To ensure compatibility with this project, your Trusted Application (TA) source file must implement the following functions:
```c
TEE Result TA_CreateEntryPoint(void)
TEE Result TA_OpenSessionEntryPoint(uint32 tparam types, TEE Param params[4], void **sess ctx) 
void TA_CloseSessionEntryPoint(void *sess ctx)
TEE_Result TA_InvokeCommandEntryPoint(void *session_id,
                                      uint32_t command_id,
                                      uint32_t parameters_type,
                                      TEE_Param parameters[4])
```
- TA_CreateEntryPoint
> Called when the TA entry point is created. For this project, no initialization is needed here.
- TA_OpenSessionEntryPoint
> Called when a new session is opened. This function must initialize both the wrapper and PSA Crypto. At minimum, it should include:
```c
TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types, TEE_Param params[4], void **sess_ctx) {
    create_session(sess_ctx);

    psa_status_t status = psa_crypto_init();
    if (status != PSA_SUCCESS) {
        return TEE_ERROR_GENERIC;
    }
    return TEE_SUCCESS;
}
```
- TA_CloseSessionEntryPoint
> Called when a session is closed. This should clean up any wrapper or session-specific resources.
- TA_InvokeCommandEntryPoint
> Called whenever a command is invoked by the host application. This function should call the relevant PSA Crypto function or your own wrapper functions containing those calls.

## Limitations
- *Single Operation Support*
> The wrapper only supports one operation at a time, when a new initialization function for a PSA function is called the old / current operation will be **cleared**, regardless of completion status.
- Application Invocation
> To invoke a Trusted Application, a host application with the same name and correct linking must be present.

## Dependencies
This project uses OP-TEE version 4.2.0
The mbedtls version used for the PSA implementation is 3.6.1
No other libraries are required
For compiling  GNU or another C compiler is required

# PSA Syscalls qemu
## Combination of OP-TEE and PSACrypto through syscalls

This Project combines the PSA Crypto implementation of mbedtls with OP-TEE.
In this Branch, PSA Crypto and Op-TEE are combined through a trapdoor design, which passes PSA function calls in the trusted application to their respective PSA functions in OP-TEE OS.

## Setup

This Project requires the same setup as normal OP-TEE (Keychain, setup, qemu...).
here is a setup used for OP-TEE on ubuntu 22.04 with a qemu setup

This setup is mostly the same as for wrapper qemu, with some slight changes
here is the complete setup

### Install Dependecies

```sh
apt update && apt upgrade -y
 
apt install -y \
    adb \
    acpica-tools \
    autoconf \
    automake \
    bc \
    bison \
    build-essential \
    ccache \
    cpio \
    cscope \
    curl \
    device-tree-compiler \
    e2tools \
    expect \
    fastboot \
    flex \
    ftp-upload \
    gdisk \
    git \
    libattr1-dev \
    libcap-ng-dev \
    libfdt-dev \
    libftdi-dev \
    libglib2.0-dev \
    libgmp3-dev \
    libhidapi-dev \
    libmpc-dev \
    libncurses5-dev \
    libpixman-1-dev \
    libslirp-dev \
    libssl-dev \
    libtool \
    libusb-1.0-0-dev \
    make \
    mtools \
    netcat \
    ninja-build \
    python3-cryptography \
    python3-pip \
    python3-pyelftools \
    python3-serial \
    python3-tomli \
    python-is-python3 \
    rsync \
    swig \
    unzip \
    uuid-dev \
    wget \
    xdg-utils \
    xsltproc \
    xterm \
    xz-utils \
    zlib1g-dev
```
### Install the repo tool
```sh
curl https://storage.googleapis.com/git-repo-downloads/repo > /bin/repo && chmod a+x /bin/repo
```
### Initialize the OP-TEE Repo

```sh
mkdir optee
cd  optee
repo init -u https://github.com/lennard2000/manifest.git -m psa_syscalls_qemu.xml && repo sync -j10
```
### Download the OP-TEE Toolchains
```sh
cd build
make -j3 toolchains
make -j$(nproc) check
```
After this the Setup should be completed.

## Commands

This Project uses the default OP-TEE commands.

Here is a list of commands used to compile and run the Project
- To compile the project:
```sh
make
```
- To compile and run the project (using QEMU):
```sh
make run
```
- To run the project without compiling (only use after compiling at least once):
```sh
make run-only
```
> these Commands only work in the `build` directory, so before executing them `cd optee/buid` may need to be executed
```sh
make run
xtest
```
This runs xTest to verify the functionality of OP-TEE (for this config this **could fail**, since we use an altered version of OP-TEE OS)
> We use a fork of OP-TEE Test to mitigate the risk of invalid implementations of OP-TEE OS and OP-TEE Test
You can also run specific tests by running
```sh
xtest <test_number>
```


## Required changes to PSA files

here is a list of methods that need to be added to a PSA file for it to correctly work with this project
```c
TEE Result TA_CreateEntryPoint(void)
TEE Result TA_OpenSessionEntryPoint(uint32 tparam types, TEE Param params[4], void **sess ctx) 
void TA_CloseSessionEntryPoint(void *sess ctx)
TEE_Result TA_InvokeCommandEntryPoint(void *session_id,
                                      uint32_t command_id,
                                      uint32_t parameters_type,
                                      TEE_Param parameters[4])
```
- TA_CreateEntryPoint:
> Called when the TA entry point is created; no special initialization is needed for this project.
- TA_OpenSessionEntryPoint:
> Called when a session is opened. Initialize anything required for your TA here.
- TA_CloseSessionEntryPoint:
> Called when a session is closed. Clean up resources initialized in TA_OpenSessionEntryPoint.
- TA_InvokeCommandEntryPoint:
>  Called when a command is invoked; this should dispatch to the correct PSA Crypto functions or wrappers.

## Limitations
This approach includes all functions inside psa_crypto.c in the mbedtls library.
Other function calls can be added quite easily.
Here is a step-by-step instruction on how to add them.

### Adding new function calls
#### Add the syscall and expose the function
- Add the function header to `lib/libutee/include/utee_syscalls.h` with the prefix `_utee_cryp`
- Add the function header to `lib/libutee/include/tee_internal_api.h` with the prefix `__GP11_` this function will be exposed to the TA
- Expose the function in `lib/libutee/include/tee_api_compat.h`
```
#define exposedFunctionName __GP11_functionNameFromStepBefore
```
> This maps the exposed function name to the correct internal function.
- Implement the mapping in `lib/libutee/tee_api_objects.c`
```
returntype __GP11_functionName(params) {
    return _utee_cryp_functionName(params);
}
```
- Add a syscall number in `lib/libutee/include/tee_syscall_numbers.h`
```
#define internalName 99
```
> (Pick the next available number, and update `TEE_SCN_MAX` accordingly.)
- Register the syscall in `lib/libutee/include/utee_syscalls_asm.S`
```
UTEE_SYSCALL _utee_cryp_functionName   , nameFrom_lib/libutee/include/tee_syscall_numbers.h, numberOfArguments
```
#### Forward the syscall to PSA in the OP-TEE OS core:

- Add the header to `core/include/tee/tee_svc_cryp.h` with the prefix `syscall_cryp_`
- Register the syscall in `core/kernel/scall.c`:
> `SYSCALL_ENTRY(syscall_cryp_functionName)`
- Add the header for the proxy function in `core/include/crypto/crypto.h` with the suffix `_proxy`
- Implement the syscall in `core/tee/tee_svc_cryp.c`:
```
returnType syscall_cryp_functionName(params) {
    return functionName_proxy(params);
}
```
- Implement the proxy in `lib/libmbedtls/core/psa_crypto.c`
```
returnType functionName_proxy(params) {
    return PSA_function(params);
}
```

## Dependencies
This project uses OP-TEE version 4.2.0
The mbedtls version used for the PSA implementation is 3.6.1
No other libraries are required
For compiling  GNU or another C compiler is required,

[manifests]: https://optee.readthedocs.io/en/latest/building/gits/build.html#manifests
