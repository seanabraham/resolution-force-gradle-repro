This is a simple repro of a problem with dependency locking in the presence of a resolution strategy.


## Setup
In this repo, the project was initialized and then dependency locking was enabled and the lock states were written.

In particular, in compileClasspath, runtimeClasspath, etc. Notice the following lines:

```
gradle/dependency-locks/testCompileClasspath.lockfile
7:com.google.guava:guava:27.0.1-jre

gradle/dependency-locks/default.lockfile
7:com.google.guava:guava:27.0.1-jre

gradle/dependency-locks/compileClasspath.lockfile
7:com.google.guava:guava:27.0.1-jre

gradle/dependency-locks/testRuntimeClasspath.lockfile
7:com.google.guava:guava:27.0.1-jre

gradle/dependency-locks/runtimeClasspath.lockfile
7:com.google.guava:guava:27.0.1-jre

```

Subsequently, a resolution strategy was added, notably forcing a version that is _not_ what is specified
in the lock state.

```groovy
configurations.all {
  resolutionStrategy {
    force 'com.google.guava:guava:27.0-jre'
  }
}
```

## Behavior
After adding the resolution strategy, running `./gradlew dependencyInsight --dependency guava` includes the following oddity:
```
com.google.guava:guava:27.0-jre
   variant "compile" [
      org.gradle.status             = release (not requested)
      org.gradle.usage              = java-api
      org.gradle.component.category = library (not requested)
   ]
   Selection reasons:
      - Forced
      - By constraint : dependency was locked to version '27.0-jre'

com.google.guava:guava:{strictly 27.0-jre} -> 27.0-jre
\--- compileClasspath

com.google.guava:guava:{strictly 27.0.1-jre} -> 27.0-jre
\--- compileClasspath
```

In addition, running `./gradlew jar` **succeeds** where I would expect it to **fail** as the dependency resolution differs from the lock state.


Intersetingly, running `./gradlew jar --write-locks` produces the following diff:
```diff
diff --git a/gradle/dependency-locks/compileClasspath.lockfile b/gradle/dependency-locks/compileClasspath.lockfile
@@ -3,8 +3,8 @@
 # This file is expected to be part of source control.
 com.google.code.findbugs:jsr305:3.0.2
 com.google.errorprone:error_prone_annotations:2.2.0
-com.google.guava:failureaccess:1.0.1
-com.google.guava:guava:27.0.1-jre
+com.google.guava:failureaccess:1.0
+com.google.guava:guava:27.0-jre
 com.google.guava:listenablefuture:9999.0-empty-to-avoid-conflict-with-guava
 com.google.j2objc:j2objc-annotations:1.1
 org.apache.commons:commons-math3:3.6.1
```

I'm not sure what the expected behavior is here. While it's good that it is possible to update the lock state to reflect the resolution strategy, the main issue is that the build doesn't fail in the presence of incompatible lock state. Moreover, it's confusing that it's acceptable from Gradle's perspective to have a successful build when a "strictly" constraint on a dependency is not met (either explicitly OR via the lock state). It leaves an open question -- even if the first issue was fixed and the build _did_ fail before the lock state was updated, should it be possible to have a valid state where a "strictly" constraint is not met on a dependency?
