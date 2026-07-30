---
name: script-ubuntu-pro-fips-test-setup
description: Set up Ubuntu Pro FIPS environment for openjdk-fips-openssl testing
---

# Generating a bash script to set up Ubuntu Pro FIPS environment for openjdk-fips-openssl testing

This assumes that you have an Ubuntu Pro instance on which you have run `sudo pro enable fips-updates` and subsequently rebooted the instance. This skill will generate a script to do the following:

1. Note your private PPA $URL and its $KEYID and set it up in `apt`.
2. Install the $PACKAGES you need to run the tests against.
3. Clone the `fips-tests` repository.
4. Run the tests and report results.
5. Clean up the environment by removing the PPA and uninstalling the packages.

Here are steps to generate the script:


1. This skill needs the following inputs:
    $URL - PPA URL from the  https://launchpad.net/~pushkarnk/+archivesubscriptions page
    $KEYID - Key ID for the PPA
    $PACKAGES - Packages from the PPA to be installed for testing

   Set up the http://squid.internal:3128 as the HTTP_PROXY and HTTPS_PROXY environment variables.

   DO NOT configure the proxy directly in apt.

2. Create a new file `setup-ubuntu-pro-fips-test.sh` and generate commands for the rest  of the steps in this file.

3. Add a command to run`sudo pro status` and ensure `fips-updates` are enabled. If not, exit and report failure.
   Ensure the command does not print a `NOTICE` with message `FIPS support requires system reboot to complete configuration`. If it does, exit and report failure.

4. Detect the Ubuntu version on this machine: jammy, noble etc. Assign it to a variable $UBUNTU_VERSION.

5. Add `sudo install -d -m 0755 /etc/apt/keyrings` to create the keyrings directory if it does not exist.

6. Add these commands to add the PPA key to the keyrings directory:
```
gpg --no-default-keyring --keyring /tmp/openjdk-fips-tmp.gpg \ --keyserver hkp://keyserver.ubuntu.com --recv-keys $KEYID

gpg --no-default-keyring --keyring /tmp/openjdk-fips-tmp.gpg \ --export $KEYID \ | sudo tee /etc/apt/keyrings/openjdk-fips.gpg > /dev/null 

rm -f /tmp/openjdk-fips-tmp.gpg
```

7. Add these commands to add the PPA to the apt sources list:
```
echo "deb [signed-by=/etc/apt/keyrings/openjdk-fips.gpg] ${URL} ${UBUNTU_VERSION} main" | sudo tee /etc/apt/sources.list.d/openjdk-fips.list

sudo apt update
```

8. Add a command to install the $PACKAGES using `sudo apt install -y ${PACKAGE}-jre` for each package in $PACKAGES. Use DEBIAN_FRONTEND=noninteractive.

9. Add a command to clone the `fips-tests` repository using:
```
git clone https://github.com/pushkarnk/fips-tests.git
cd fips-tests
```

10. Add a command to install `openjdk-17-jdk` using `sudo apt install -y openjdk-17-jdk`. Use DEBIAN_FRONTEND=noninteractive.

11. Add a command to detect the architecture:
```
ARCH=$(dpkg --print-architecture)
```
12. Add commands to run tests for $PACKAGES. For each package, detect the base openjdk version (17 or 21) from the name, say $VERSION. Then run the tests:
```
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-${ARCH}
export JAVA_FIPS_JRE=/usr/lib/jvm/java-${VERSION}-openjdk-fips-openssl-${ARCH}/
./run.sh
```
Save the output of the test in a file `$PWD/test-results.txt` and report the results.

13. Clean up. Remove the installed packages: openjdk-17-jdk, $PACKAGES and their dependencies. Remove the PPA from the apt sources list and remove the keyring file. Use `sudo apt autoremove -y` to remove any unused dependencies. Please make sure these pacakges are removed:
1. openjdk-{17,21}-openjdk-fips-openssl-${ARCH}
2. libopenssl-fips-java
3. libopenssl-fips-java-jni
