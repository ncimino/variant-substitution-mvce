# Variant Substitution MVCE

The goal is to swap out one module (library) for testing to be the release 
(optimized) variant while leaving the rest as debug (which is default when 
testing).

# Building

There are dependent projects, so go ahead and build those first:

    ./gradlew -p ./list/ publishAllPublicationsToLocalRepo
    ./gradlew -p ./utilities/ publishAllPublicationsToLocalRepo

Then run the project that attempts to swap out the module variant:

    ./gradlew -p ./app/ assembleRelease

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

# Further Debug

The utilities library was outfitted with an `#ifdef NDEBUG` which is only set when the 
`cppCompile` task property `optimized` is set. Then the `NDEBUG` macro is defined the
`utilities` library will also append the message `DEBUG` to the output string on join.

This means that if we have successfully pulled in a debug version of the utilities
library we will see `DEBUG` and if it is optimized we will not see this string.

For example building `app` both optimized and non-optimized:

    $ ./gradlew -p ./app/ clean assembleRelease assembleDebug
    
    BUILD SUCCESSFUL in 1s
    9 actionable tasks: 9 executed

Then running optimized produces:

    $ ./app/build/install/main/release/app 
    Hello, World!

Then running non-optimized produces:

    $ ./app/build/install/main/debug/app 
    Hello,DEBUG World!DEBUG

# Tested Solutions

## Final Solution - Did work

This is the code that was suggested in the Gradle Community Slack:

See: https://gradle-community.slack.com/archives/CAH4ZP3GX/p1628976229066300?thread_ts=1628881212.065000&cid=CAH4ZP3GX

    dependencies {
        implementation("com.company:utilities:6.5.4") {
            attributes {
                attribute(CppBinary.OPTIMIZED_ATTRIBUTE, false)
            }
        }
        implementation "com.company:list:3.2.1"
        attributesSchema {
            attribute(CppBinary.OPTIMIZED_ATTRIBUTE) {
                compatibilityRules.ordered(Comparator.naturalOrder())
            }
        }
    }


## Attempted Solution - Did not work

This is the code that was expected to make this work:

See: https://gradle-community.slack.com/archives/CAH4ZP3GX/p1628881212065000

    dependencies {
        implementation "com.company:utilities:6.5.4"
        implementation "com.company:list:3.2.1"
    }
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




