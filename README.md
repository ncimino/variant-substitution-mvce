# Variant Substitution MVCE

The goal is to swap out one module (library) for testing to be the release 
(optimized) variant while leaving the rest as debug (which is default when 
testing).

# Building

There are dependent projects, so go ahead and build those first:

    ./gradlew -p ./list/ publishAllPublicationsToLocalRepo
    ./gradlew -p ./utilities/ publishAllPublicationsToLocalRepo

Then run the project that attempts to swap out the module variant:

    ./gradlew -p ./app/ test

This produces:

    * What went wrong:
    Execution failed for task ':installTest'.
    > Could not resolve all files for configuration ':nativeRuntimeTest'.
    > Could not resolve com.company:utilities_release:6.5.4.
        Required by:
            project : > com.company:utilities:6.5.4
        > No matching variant of com.company:utilities_release:6.5.4 was found. The consumer was configured to find attribute 'org.gradle.usage' with value 'native-runtime', attribute 'org.gradle.native.debuggable' with value 'true', attribute 'org.gradle.native.optimized' with value 'false', attribute 'org.gradle.native.operatingSystem' with value 'linux', attribute 'org.gradle.native.architecture' with value 'x86-64' but:
            - Variant 'releaseLink' capability com.company:utilities_release:6.5.4 declares attribute 'org.gradle.native.architecture' with value 'x86-64', attribute 'org.gradle.native.debuggable' with value 'true', attribute 'org.gradle.native.operatingSystem' with value 'linux':
                - Incompatible because this component declares attribute 'org.gradle.native.optimized' with value 'true', attribute 'org.gradle.usage' with value 'native-link' and the consumer needed attribute 'org.gradle.native.optimized' with value 'false', attribute 'org.gradle.usage' with value 'native-runtime'
            - Variant 'releaseRuntime' capability com.company:utilities_release:6.5.4 declares attribute 'org.gradle.native.architecture' with value 'x86-64', attribute 'org.gradle.native.debuggable' with value 'true', attribute 'org.gradle.native.operatingSystem' with value 'linux', attribute 'org.gradle.usage' with value 'native-runtime':
                - Incompatible because this component declares attribute 'org.gradle.native.optimized' with value 'true' and the consumer needed attribute 'org.gradle.native.optimized' with value 'false'


# Expected Solution

This is the code that is expected to make this work:

    afterEvaluate {
        configurations.nativeRuntimeTest {
            resolutionStrategy.dependencySubstitution {
                substitute(module('com.company:utilities:6.5.4'))
                        .using variant(module('com.company:utilities:6.5.4')) {
                    attributes {
                        attribute(Attribute.of("org.gradle.native.optimized", Boolean.class), true)
                    }
                }
            }
        }
    }




