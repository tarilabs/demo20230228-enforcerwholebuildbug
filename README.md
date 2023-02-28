A project where module3 depends on module2, and module2 depends on module1.

However, we want to shade module2 (globbing module1 inside of itself).

✅ You can run the full project (from root) with: `mvn clean install -Denforcer.skip` and it would work correctly.

✅ Checking the content of module2 which was shaded:

```
$ jar tvf module2/target/module2-1.0-SNAPSHOT.jar 
     0 Tue Feb 28 15:26:54 CET 2023 META-INF/
    98 Tue Feb 28 15:26:54 CET 2023 META-INF/MANIFEST.MF
     0 Tue Feb 28 15:26:52 CET 2023 org/
     0 Tue Feb 28 15:26:52 CET 2023 org/acme/
     0 Tue Feb 28 15:26:52 CET 2023 org/acme/demo20230228/
     0 Tue Feb 28 15:26:52 CET 2023 org/acme/demo20230228/module2/
   641 Tue Feb 28 15:26:52 CET 2023 org/acme/demo20230228/module2/App2.class
     0 Tue Feb 28 15:25:14 CET 2023 META-INF/maven/
     0 Tue Feb 28 15:25:14 CET 2023 META-INF/maven/org.acme.demo20230228/
     0 Tue Feb 28 15:25:14 CET 2023 META-INF/maven/org.acme.demo20230228/module2/
  1462 Tue Feb 28 15:25:14 CET 2023 META-INF/maven/org.acme.demo20230228/module2/pom.xml
   101 Tue Feb 28 15:26:54 CET 2023 META-INF/maven/org.acme.demo20230228/module2/pom.properties
     0 Tue Feb 28 15:26:52 CET 2023 org/acme/demo20230228/module1/
   390 Tue Feb 28 15:26:52 CET 2023 org/acme/demo20230228/module1/Utils.class
     0 Tue Feb 28 14:30:44 CET 2023 META-INF/maven/org.acme.demo20230228/module1/
   541 Tue Feb 28 14:30:44 CET 2023 META-INF/maven/org.acme.demo20230228/module1/pom.xml
   101 Tue Feb 28 15:26:52 CET 2023 META-INF/maven/org.acme.demo20230228/module1/pom.properties
```

As expected it does contain `org/acme/demo20230228/module1/Utils.class` from module1.

🧐 One weird behaviour start to appear when:

```
$ mvn dependency:tree -pl module3
[INFO] Scanning for projects...
[INFO] 
[INFO] -------------------< org.acme.demo20230228:module3 >--------------------
[INFO] Building module3 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-dependency-plugin:2.8:tree (default-cli) @ module3 ---
[INFO] org.acme.demo20230228:module3:jar:1.0-SNAPSHOT
[INFO] +- org.acme.demo20230228:module2:jar:1.0-SNAPSHOT:compile
[INFO] \- junit:junit:jar:4.13.2:test
[INFO]    \- org.hamcrest:hamcrest-core:jar:1.3:test
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.628 s
[INFO] Finished at: 2023-02-28T15:35:21+01:00
[INFO] ------------------------------------------------------------------------
```

Please notice we're asking the dependency tree of `module3` and indeed as we would expect, `module3` only depends on `module2`, since this latter `module2` it's shading `module1` in its pom.
However this is not really the case.

🚨 Spoiler alert: if computing the dependency tree over the whole project, the result will be **DIFFERENT**. That is, depending on _where_ you ask the dependency:tree, results will be different! Quite scary stuff.

🤔 Enforcer plugin problem:

```
$ mvn clean install
...
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for demo20230228-enforcerwholebuildbug 1.0-SNAPSHOT:
[INFO] 
[INFO] demo20230228-enforcerwholebuildbug ................. SUCCESS [  0.619 s]
[INFO] module1 ............................................ SUCCESS [  0.889 s]
[INFO] module2 ............................................ SUCCESS [  1.124 s]
[INFO] module3 ............................................ FAILURE [  0.024 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.758 s
[INFO] Finished at: 2023-02-28T15:38:24+01:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-enforcer-plugin:3.2.1:enforce (enforce-ban-duplicate-classes) on project module3: 
[ERROR] Rule 0: org.apache.maven.plugins.enforcer.BanDuplicateClasses failed with message:
[ERROR] Duplicate classes found:
[ERROR] 
[ERROR]   Found in:
[ERROR]     org.acme.demo20230228:module1:jar:1.0-SNAPSHOT:compile
[ERROR]     org.acme.demo20230228:module2:jar:1.0-SNAPSHOT:compile
[ERROR]   Duplicate classes:
[ERROR]     org/acme/demo20230228/module1/Utils.class
```

While on the surface, we thought (see earlier) that module3 depends only on module2, still seems module1 is picked up when scanning with enforcer.

As anticipated in the spoiler, when running:

```
$ mvn dependency:tree
...
[INFO] -------------------< org.acme.demo20230228:module3 >--------------------
[INFO] Building module3 1.0-SNAPSHOT                                      [4/4]
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-dependency-plugin:2.8:tree (default-cli) @ module3 ---
[INFO] org.acme.demo20230228:module3:jar:1.0-SNAPSHOT
[INFO] +- org.acme.demo20230228:module2:jar:1.0-SNAPSHOT:compile
[INFO] |  \- org.acme.demo20230228:module1:jar:1.0-SNAPSHOT:compile
[INFO] \- junit:junit:jar:4.13.2:test
[INFO]    \- org.hamcrest:hamcrest-core:jar:1.3:test
[INFO] ------------------------------------------------------------------------
```

🤯 Now Maven, computing the dependency:tree for ALL the modules, is telling us that module3 depends transitively from module1, which was NOT what we got earlier!

😵‍💫 But if we walk inside of module3 now...

```
$ cd module3
$ pwd
/Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3
$ mvn clean install
[INFO] Scanning for projects...
[INFO] 
[INFO] -------------------< org.acme.demo20230228:module3 >--------------------
[INFO] Building module3 1.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:3.1.0:clean (default-clean) @ module3 ---
[INFO] 
[INFO] --- maven-enforcer-plugin:3.2.1:enforce (enforce-ban-duplicate-classes) @ module3 ---
[INFO] Adding ignore: module-info
[INFO] Adding ignore: META-INF/versions/*/module-info
[INFO] Rule 0: org.apache.maven.plugins.enforcer.BanDuplicateClasses passed
[INFO] 
[INFO] --- maven-resources-plugin:3.0.2:resources (default-resources) @ module3 ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/src/main/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.0:compile (default-compile) @ module3 ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.0.2:testResources (default-testResources) @ module3 ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.0:testCompile (default-testCompile) @ module3 ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.1:test (default-test) @ module3 ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running org.acme.demo20230228.module3.AppTest
Hello, World
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.056 s - in org.acme.demo20230228.module3.AppTest
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] 
[INFO] --- maven-jar-plugin:3.0.2:jar (default-jar) @ module3 ---
[INFO] Building jar: /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/target/module3-1.0-SNAPSHOT.jar
[INFO] 
[INFO] >>> maven-source-plugin:3.2.1:jar (attach-sources) > generate-sources @ module3 >>>
[INFO] 
[INFO] --- maven-enforcer-plugin:3.2.1:enforce (enforce-ban-duplicate-classes) @ module3 ---
[INFO] Adding ignore: module-info
[INFO] Adding ignore: META-INF/versions/*/module-info
[INFO] Rule 0: org.apache.maven.plugins.enforcer.BanDuplicateClasses passed
[INFO] 
[INFO] <<< maven-source-plugin:3.2.1:jar (attach-sources) < generate-sources @ module3 <<<
[INFO] 
[INFO] 
[INFO] --- maven-source-plugin:3.2.1:jar (attach-sources) @ module3 ---
[INFO] Building jar: /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/target/module3-1.0-SNAPSHOT-sources.jar
[INFO] 
[INFO] --- maven-install-plugin:2.5.2:install (default-install) @ module3 ---
[INFO] Installing /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/target/module3-1.0-SNAPSHOT.jar to /Users/mmortari/.m2/repository/org/acme/demo20230228/module3/1.0-SNAPSHOT/module3-1.0-SNAPSHOT.jar
[INFO] Installing /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/pom.xml to /Users/mmortari/.m2/repository/org/acme/demo20230228/module3/1.0-SNAPSHOT/module3-1.0-SNAPSHOT.pom
[INFO] Installing /Users/mmortari/git/tmp/demo20230228-enforcerwholebuildbug/module3/target/module3-1.0-SNAPSHOT-sources.jar to /Users/mmortari/.m2/repository/org/acme/demo20230228/module3/1.0-SNAPSHOT/module3-1.0-SNAPSHOT-sources.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  2.672 s
[INFO] Finished at: 2023-02-28T15:43:15+01:00
[INFO] ------------------------------------------------------------------------
```

Now is a success, no complain of enforcer.

🤷 The only solution I found was to make `<optional>true</optional>` the dependency of `module1` inside of `module2/pom.xml`.
