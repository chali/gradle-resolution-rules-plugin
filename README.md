# Gradle Resolution Rules Plugin

![Support Status](https://img.shields.io/badge/Nebula-supported-brightgreen.svg)
![Version](https://img.shields.io/maven-central/v/com.netflix.nebula/gradle-resolution-rules-plugin.svg)
[![Build Status](https://travis-ci.org/nebula-plugins/gradle-resolution-rules-plugin.svg?branch=master)](https://travis-ci.org/nebula-plugins/gradle-resolution-rules-plugin)
[![Coverage Status](https://coveralls.io/repos/nebula-plugins/gradle-resolution-rules-plugin/badge.svg?branch=master&service=github)](https://coveralls.io/github/nebula-plugins/gradle-resolution-rules-plugin?branch=master)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/nebula-plugins/gradle-resolution-rules-plugin?utm_source=badgeutm_medium=badgeutm_campaign=pr-badge)
[![Apache 2.0](https://img.shields.io/github/license/nebula-plugins/gradle-resolution-rules-plugin.svg)](http://www.apache.org/licenses/LICENSE-2.0)

Gradle plugin for providing dependency resolution rules.

Gradle resolution strategies and module metadata provide an effective way to solve the most common dependency issues, however sharing these rules between projects is cumbersome, and requires custom plugins or `apply from` calls. This plugin provides general purpose rule types, allowing rules to be published, versioned, shared between projects, and optionally [dependency locked](https://github.com/nebula-plugins/gradle-dependency-lock-plugin).

These rule types solve the most common cause of dependency issues in projects, including:

- Duplicate classes caused by changes to group or artifact ids, without renaming packages
- Duplicate classes caused by bundle dependencies, which do not conflict resolve against the 'regular' dependencies for that library
- Lack of version alignment between libraries, where version alignment is needed for compatibility

We produce a rules for dependencies found in Maven Central and other public repositories. See [OSS rules](#oss-rules) below for more information.

# Usage

Refer to the [Gradle Plugin Portal](https://plugins.gradle.org/plugin/nebula.resolution-rules) for usage instructions.

## Dependency Lock Compatibility

To use [nebula.dependency-lock](https://github.com/nebula-plugins/gradle-dependency-lock-plugin) with this plugin, apply `nebula.dependency-lock` later than the resolution rules plugin.

# Consuming rules

Dependency rules are read from the `resolutionRules` configuration. Zip and jar archives are supported, as are flat JSON files. JSON files within archives can at any directory level, and more than one json file can be provided.

```groovy
dependencies {
    resolutionRules files('local-rules.json')
    resolutionRules 'com.myorg:resolution-rules:latest.release'
}
```
## OSS rules

We produce a rules for dependencies found in Maven Central and other public repositories, to use those rules in your project add:

```groovy
dependencies {
    resolutionRules 'com.netflix.nebula:gradle-resolution-rules:latest.release'
}
```

See the [resolution-rules project](https://github.com/nebula-plugins/gradle-resolution-rules) for details of the rules, and instructions on how to enable optional rule sets.

## Excluding rules

Rules may be excluded by filename, omitting the `.json` extension:

```groovy
nebulaResolutionRules {
    exclude = ['local-rules']
}
```

## Including optional rules

Optional rules files are prefixed with `optional-`, and must be included explicitly, omitting the `.json` extension from the filename:

```groovy
dependencies {
    resolutionRules files('optional-local-rules.json')
}

nebulaResolutionRules {
    include = ['optional-local-rules']
}
```

# Producing rules

The `nebula.resolution-rules-producer` plugin is provided to facilitate creation of rule files. This plugin can be used to validate and package rule files. To use, put your rules file in `src/resolutionRules/resolution-rules.json` and execute the packageRules task. If successful, a jar with the validated rules file will be placed in your build directory. For specifying more than one source file, see documentation below.

## Usage

```groovy
buildscript {
    repositories {
        jcenter()
    }

    dependencies {
        classpath 'com.netflix.nebula:gradle-resolution-rules-plugin:1.4.0'
    }
}

apply plugin: 'nebula.resolution-rules-producer'
```

Or using the Gradle plugin portal:

```groovy
plugins {
    id 'nebula.resolution-rules-producer' version '1.4.0'
}
```

Once configured, run the following:

    $ ./gradlew packageRules

## Customizing rule file locations

```groovy
checkResolutionRulesSyntax {
    rules files('./alternative-rules.json', './src/rules/moreRules.json')
}
```

Prefix the rule filename with `optional-` to make a rules filename optional, and prevent it from being applied by default.

# Dependency rules types

## Replace
 
Replace rules cause dependencies to be replaced with one at different coordinates, if both dependencies are present, avoiding class conflicts due to coordinate relocations.

## Substitute

Dependency is force replaced with new coordinates, regardless if the new dependency is visible in the graph.

## Deny

Deny rules fail the build if the specified dependency is present, forcing consumers to make a decision about the replacement of a dependency.

Versions are optional, and access to an entire dependency can be denied. This is particularly useful for 'bundle' style dependencies, which fat jar dependencies without shading classes.
 
## Reject 

Reject rules prevent a dependency version from being considered by dynamic version or `latest.*` version calculation. It does not prevent that dependency from being used if it's explicitly requested by the project.

## Align

Align rules allows the user to have a group of dependencies if present to have the same version. Example: if `jersey-core` and `jersey-server` are present make sure they use the same version. By default this will choose the largest version provided among the list.

This rule has different options depending on your use case. `includes` and `excludes` are mutually exclusive, the plugin will error out if both are present in one rule.

##### If you need to skip a rule, you can use this extension

```groovy
nebulaResolutionRules { // If you need to skip a rule 
    skipAlignRules = ['<names>']
}
```

#### Lock only a few dependencies in a group

```json
{
    "align": [
        {
            "group": "example.foo",
            "includes": [ "bar", "baz" ]
        }
    ]
}
```

#### Lock most dependencies in a group, but exclude some

```json
{
    "align": [
        {
            "name": "example",
            "group": "example.bar",
            "excludes": [ "qux", "thud" ]
        }
    ]
}
```

#### Lock all dependencies in a group matching a regular expression

```json
{
    "align": [
        {
            "group": "example.foo.*",
        }
    ]
}
```

#### Lock all dependencies, including those matching a regular expression

```json
{
    "align": [
        {
            "group": "example.foo",
            "includes": [ "(bar|baz)" ]
        }
    ]
}
```

## Exclude

Exclude rules excludes a dependency completely, similar to deny, but does not support a version and silently removes the dependency, rather than causing an error.

# Example rules JSON

```json
{
    "replace" : [
        {
            "module" : "asm:asm",
            "with" : "org.ow2.asm:asm",
            "reason" : "The asm group id changed for 4.0 and later",
            "author" : "Example Person <person@example.org>",
            "date" : "2015-10-07T20:21:20.368Z"
        }
    ],
    "substitute": [
        {
            "module" : "bouncycastle:bcprov-jdk15",
            "with" : "org.bouncycastle:bcprov-jdk15:latest.release",
            "reason" : "The latest version of BC is required, using the new coordinate",
            "author" : "Example Person <person@example.org>",
            "date" : "2015-10-07T20:21:20.368Z"
        }
    ],
    "deny": [
        {
            "module": "com.google.guava:guava:19.0-rc2",
            "reason" : "Guava 19.0-rc2 is not permitted",
            "author" : "Example Person <person@example.org>",
            "date" : "2015-10-07T20:21:20.368Z"
        },
        {
            "module": "com.sun.jersey:jersey-bundle",
            "reason" : "jersey-bundle is a fat jar that includes non-relocated (shaded) third party classes, which can cause duplicated classes on the classpath. Please specify the jersey- libraries you need directly",
            "author" : "Example Person <person@example.org>",
            "date" : "2015-10-07T20:21:20.368Z"
        }
    ],
    "reject": [
        {
            "module": "com.google.guava:guava:12.0",
            "reason" : "Guava 12.0 significantly regressed LocalCache performance",
            "author" : "Example Person <person@example.org>",
            "date" : "2015-10-07T20:21:20.368Z"
        }
    ],
    "align": [
        {
            "name": "alignJersey",
            "group": "com.sun.jersey",
            "reason": "Make sure jersey-core, jersey-server, etc. are aligned e.g. 1.19.1",
            "author": "Example Person <person@example.org>",
            "date": "2015-10-08T20:15:14.321Z"
        }
    ],
    "exclude": [
        {
            "module": "io.netty:netty-all",
            "reason": "Bundle dependencies are harmful, they do not conflict resolve with the non-bundle dependencies",
            "author" : "Danny Thomas <dmthomas@gmail.com>",
            "date" : "2015-10-07T20:21:20.368Z"
        }
    ]
}
```
