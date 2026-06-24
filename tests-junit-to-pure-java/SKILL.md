---
name: tests-junit-to-pure-java
description: Convert JUnit-based tests from openssl-fips-java to pure Java tests
---

# Converting a JUnit test to a pure Java test

## This skill is specific to the openssl-fips-java provider which is available at https://github.com/canonical/openssl-fips-java. The CI on this provider runs JUnit tests with openssl and its FIPS module built from source and installed as test dependencies. However, we also need to run the same tests on an Ubuntu Pro instance, which has openssl FIPS installed as a system package. The current infrastructure that I need to use for Ubuntu Pro has severe restrictions on outbound network access, which makes it impossible to run JUnit tests that require downloading dependencies from Maven Central. Therefore, I need to convert the JUnit tests to pure Java tests that do not require any external dependencies. Given below are steps to convert a JUnit test to a pure Java test.

## 1. Clone the openssl-fips-java and fips-tests repository
```
git clone git@github.com:canonical/openssl-fips-java.git
git clone git@github.com:pushkarnk/fips-tests.git
```
Make `openssl-fips-java` as the working directory.
```
cd openssl-fips-java
```

## 2. Build the openssl-fips-java provider
If openjdk-17-jdk is not installed, install it using:
```
sudo apt-get update
sudo apt-get install openjdk-17-jdk
```

Set JAVA_HOME and compile tests:
```
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
mvn package -DskipTests
```

## 3. Convert the JUnit tests to pure Java tests
Do this for every *Test.java file found in the `src/test/java` directory of the `openssl-fips-java` repository. For example, for the `CipherTest.java` file, do the following:

1. Create or modify the CipherTests.java file in the `fips-tests` directory.

2. Introduce a `public static void main(String[] args)` method in the CipherTests.java file.

3. Remove all the JUnit imports and annotations from the CipherTests.java file.

4. Move any setup or teardown code from `@Before` and `@After` methods into the `main` method.

5. There are some assertion methods in Utils.java. Use them. If you need a new assertion method, add it to Utils.java.

6. Try building the CipherTests.java file using:
```
cd ../fips-tests
javac -cp target/openssl-fips-java-*-SNAPSHOT.jar CipherTests.java
```
7. If the javac command fails, fix the errors and repeat step 6 until the javac command succeeds.

8. After the file compiles successfully, remove all references to `OpenSSLFIPSProvider` from it. The Ubuntu Pro test environment implicitly adds the provider to the classpath and registers it in the `java.security` file, so the tests must not reference it directly. Specifically:
   - Remove the `import com.canonical.openssl.provider.OpenSSLFIPSProvider;` line.
   - Remove any `Security.addProvider(new OpenSSLFIPSProvider());` calls.
   - Drop the `"OpenSSLFIPSProvider"` string argument from every `getInstance(...)` call so that the default provider lookup is used (e.g. `Cipher.getInstance(name, "OpenSSLFIPSProvider")` becomes `Cipher.getInstance(name)`).
   - If the file uses reflection to call `getInstance(String, String)`, update it to call the single-argument `getInstance(String)` instead.
   - Remove the `import java.security.Security;` line if it is no longer used.
   - Recompile the file to confirm it still builds without errors.

9. Change directory back to `openssl-fips-java` and repeat all steps for every *Test.java file found in the `src/test/java` directory of the `openssl-fips-java` repository.

## 4. Commit changes to pushkarnk/fips-tests repository
```
cd ../fips-tests
git add .
git commit -m "Converted JUnit tests to pure Java tests"
git push origin main
```

End of skill.
