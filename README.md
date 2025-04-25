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

This Project requires the same setup as normal OP-TEE (Keychain, setup, quemo...).
here is a setup used for optee on ubuntu 22.04 with a quemo setup
```
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
    
    curl https://storage.googleapis.com/git-repo-downloads/repo > /bin/repo && chmod a+x /bin/repo
```
After this you will need to initiate the repo
```
mkdir /optee
cd  optee
repo init -u https://github.com/lennard2000/manifest.git -m wrapper_qemu.xml && repo sync -j10
```
The next step is to download the toolchains required for OP-TEE
```
cd build
make -j3 toolchains
make -j$(nproc) check
```
After this the Setup should be completed.

## Commands

This Projects uses the default OP-TEE commands.
Here is a list of commands used to compile and run the Project
```
make
``` 
This command compiles the project
```
make run
```
compiles the project and runs it (using Quemo if on ubuntu)
```
make run-only
```
starts the project without compiling. You need to compile the project before starting it though

## Testing
To test the project you simply need to compile the project and start it.
This already runs the default OP-TEE tests, so after successfully compiling we can assume that the OP-TEE implementation works correctly.
```
make
```
To make sure OP-TEE works correctly with PSACrypto you need to run the project and execute the example Trusted applications
```
make run
acipher
aes
hello_world
```
These are the same names as the default OP-TEE examples, but contain the PSA test contained in the mbedtls repository,  slightly altered to conform to OP-TEE .
These calls should not have an output, since they validate themself using constant values. In case of mismatches or errors in the setup / functionality an error will be thrown

```
make run
xtest
```
This runs xTest to verify the functionality of OP-TEE (for this config this should not fail, since OP-TEE OS isn't altered)
You can also run specific tests by running
```
xtest <test_number>
```


## Required changes to PSA files

here is a list of methods that need to be added to a PSA file for it to correctly work with this project
```
TEE Result TA_CreateEntryPoint(void)
TEE Result TA_OpenSessionEntryPoint(uint32 tparam types, TEE Param params[4], void **sess ctx) 
void TA_CloseSessionEntryPoint(void *sess ctx)
TEE_Result TA_InvokeCommandEntryPoint(void *session_id,
                                      uint32_t command_id,
                                      uint32_t parameters_type,
                                      TEE_Param parameters[4])
```
The function  TA_CreateEntryPoint is called when the entry point is created, for this project it doesn't need to do anything.
The  TA_OpenSessionEntryPoint function is called when a session opens, it should at least contain the following function calls
```
TEE_Result TA_OpenSessionEntryPoint(uint32_t param_types, TEE_Param params[4], void **sess_ctx) {
    create_session(sess_ctx);

    psa_status_t status = psa_crypto_init();
    if (status != PSA_SUCCESS) {
        return TEE_ERROR_GENERIC;
    }
    return TEE_SUCCESS;
}
``` 
These calls initialize the wrapper and PSACrypto.
The function void TA_CloseSessionEntryPoint should at least clear the initialization of the wrapper.
The function TEE_Result TA_InvokeCommandEntryPoint is called when a command is invoked.
This function should contain a call to the correct PSACrypto function or to functions containing them.

## Limitations
The wrapper only supports one operation at a time, when a new initialization function for a PSA function is called the old / current operation will be **cleared**, whether it is finished or not.
To invoke an Trusted application a host with the same name and the correct linking needs to be present.

## Dependencies
This project uses OP-TEE version 4.2.0
The mbedtls version used for the PSA implementation is 3.6.1
No other libraries are required
For compipling  GNU or another C compiler is required,

# PSA Syscalls qemu
## Combination of OP-TEE and PSACrypto through syscalls

This Project combines the PSA Crypto implementation of mbedtls with OP-TEE.
In this Branch, PSA Crypto and Op-TEE are combined through a trapdoor design, which passes PSA function calls in the trusted application to their respective PSA functions in OP-TEE OS.

## Setup

This Project requires the same setup as normal OP-TEE (Keychain, setup, qemu...).
here is a setup used for optee on ubuntu 22.04 with a qemu setup

This setup is mostly the same as for wrapper qemu, with some slight changes
here is the complete setup

```
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
    
    curl https://storage.googleapis.com/git-repo-downloads/repo > /bin/repo && chmod a+x /bin/repo
```
After this you will need to initiate the repo
```
mkdir /optee
cd  optee
repo init -u https://github.com/lennard2000/manifest.git -m psa_syscalls_qemu.xml && repo sync -j10
```
The next step is to download the toolchains required for OP-TEE
```
cd build
make -j3 toolchains
make -j$(nproc) check
```
After this, the Setup should be completed.

## Commands

This Projects uses the default OP-TEE commands.
Here is a list of commands used to compile and run the Project
```
make
``` 
This command compiles the project
```
make run
```
compiles the project and runs it (using Quemo if on ubuntu)
```
make run-only
```
starts the project without compiling. You need to compile the project before starting it though

# Testing
To test the project you simply need to compile the project and start it.
This already runs the default OP-TEE tests, so after successfully compiling we can assume that the OP-TEE implementation works correctly.
```
make
```
To make sure OP-TEE works correctly with PSACrypto you need to run the project and execute the example Trusted applications
```
make run
acipher
aes
hello_world
```
These are the same names as the default OP-TEE examples, but contain the PSA test contained in the mbedtls repository,  slightly altered to conform to OP-TEE .
These calls should not have an output, since they validate themself using constant values. In case of mismatches or errors in the setup / functionality an error will be thrown

```
make run
xtest
```
This runs xTest to verify the functionality of OP-TEE (for this config this should not fail, since OP-TEE OS isn't altered)
You can also run specific tests by running
```
xtest <test_number>
```


## Required changes to PSA files

here is a list of methods that need to be added to a PSA file for it to correctly work with this project
```
TEE Result TA_CreateEntryPoint(void)
TEE Result TA_OpenSessionEntryPoint(uint32 tparam types, TEE Param params[4], void **sess ctx) 
void TA_CloseSessionEntryPoint(void *sess ctx)
TEE_Result TA_InvokeCommandEntryPoint(void *session_id,
                                      uint32_t command_id,
                                      uint32_t parameters_type,
                                      TEE_Param parameters[4])
```
The function  TA_CreateEntryPoint is called when the entry point is created, for this project it doesn't need to do anything.
The  TA_OpenSessionEntryPoint function is called when a session opens, it should contain specific calls or initializations required during runtime of this TA

The function void TA_CloseSessionEntryPoint should clear the initializations or setup executed in TA_OpenSessionEntryPoint.
The function TEE_Result TA_InvokeCommandEntryPoint is called when a command is invoked.
This function should contain a call to the correct PSACrypto function or to functions containing them.

## Limitations
This approach includes all functions inside psa_crypto.c in the mbedtls library.
Other function calls can be added quite easily.
Here is a step-by-step instruction on how to add them.

### Adding new function calls
Go to the file lib/libutee/include/utee_syscalls.h and add the header of the function, it should habe the prefix "_utee_cryp"

Go to the file lib/libutee/include/tee_internal_api.h and add the header of the function with the prefix "__GP11_" this will be exposed to the TA

Go to the file lib/libutee/include/tee_api_compat.h and add the following line "#define exposedFunctionName __GP11_functionNameFromStepBefore" with this line exposedFunctionName will be exposed to the TA and point to __GP11_functionNameFromStepBefore

Go to the file lib/libutee/tee_api_objects.c and add the following
```
returntype __GP11_functionName(params) {
    return _utee_cryp_functionName(params);
}
```
This will map the exposed function to the syscall we will define now
Go to the file lib/libutee/include/tee_syscall_numbers.h and add a syscall number, just pick the next number
the added line should look like this 
```
#define internalName 99
```
remember to increase the TEE_SCN_MAX after adding syscalls here

Go to the file lib/libutee/include/utee_syscalls_asm.S and add the syscall like this
```
UTEE_SYSCALL _utee_cryp_functionName   , nameFrom_lib/libutee/include/tee_syscall_numbers.h, numberOfArguments
```
At this point we already added the syscalls and exposed the function to the TA's.
Now we add the code so it is forwarded to PSA

Go to the file core/include/tee/tee_svc_cryp.h and add the header of the function with a "syscall_cryp_" prefix

Go to the file core/kernel/scall.c and add the following line 
```
SYSCALL_ENTRY(syscall_cryp_functionName),
```

Now we add the last headers for the function to core/include/crypto/crypto.h with the suffix "_proxy"

Now go to the file core/tee/tee_svc_cryp.c and implement the "syscall_cryp_functionName" function here to call the "functionName_proxy" function
```
returnType syscall_cryp_functionName(params) {
    return functionName_proxy(params);
}
```
The final step is to go to file lib/libmbedtls/core/psa_crypto.c and add an implemetation of the functionName_proxy function which calls the PSA function

```
returnType functionName_proxy(params) {
    return PSA_function(params);
}
```

## Dependencies
This project uses OP-TEE version 4.2.0
The mbedtls version used for the PSA implementation is 3.6.1
No other libraries are required
For compipling  GNU or another C compiler is required,

[manifests]: https://optee.readthedocs.io/en/latest/building/gits/build.html#manifests
