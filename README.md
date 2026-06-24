# seatunnel-shade

Apache SeaTunnel Shade — shaded (relocated) JARs of third-party libraries for Apache SeaTunnel.

Each module wraps a single third-party library, relocating its packages under `org.apache.seatunnel.shade.*` to avoid classpath conflicts when SeaTunnel itself depends on different versions of the same libraries.

## Modules

| Module | Library | Version |
|--------|---------|---------|
| `seatunnel-shade-hazelcast` | Hazelcast | 5.1 |
| `seatunnel-shade-hadoop3-uber` | Hadoop Client | 3.1.4 |
| `seatunnel-shade-jackson` | Jackson | 2.13.3 |
| `seatunnel-shade-guava` | Guava | 27.0-jre |
| `seatunnel-shade-hadoop-aws` | Hadoop AWS | 3.1.4 |
| `seatunnel-shade-arrow` | Apache Arrow | 15.0.1 |
| `seatunnel-shade-hikari` | HikariCP | 4.0.3 |
| `seatunnel-shade-thrift-service` | Apache Doris Thrift | 1.0.0 |
| `seatunnel-shade-janino` | Janino | 3.1.12 |
| `seatunnel-shade-scala-compiler` | Scala Compiler | 2.12.15 |
| `seatunnel-shade-jetty` | Jetty | 9.4.56 |
| `seatunnel-shade-commons-lang3` | Commons Lang3 | 3.18.0 |
| `seatunnel-shade-calcite` | Apache Calcite | 1.38.0 |

## Build

```bash
# Full build (skip tests, skip RAT license check)
mvn -B -DskipTests -Drat.skip=true clean install

# Build with parallelism
mvn -B -DskipTests -Drat.skip=true -T 2C clean install

# Build single module + dependencies
mvn -B -DskipTests -pl seatunnel-shade-guava -am clean install

# Build single module only (no dependencies)
mvn -B -DskipTests -pl seatunnel-shade-guava clean install
```

## Release

### Full release

Deploy all modules:

```bash
mvn -B -DskipTests -Drat.skip=true clean deploy
```

### Partial release (one or a few modules)

When upstream SeaTunnel changes only affect a subset of shade modules, use `-pl` to deploy only the changed modules. Running a full deploy would hit already-published versions and fail with a Nexus 400 error.

**Example**: SeaTunnel added a Calcite plugin and aligned the Scala version — only 3 modules need redeploy.

**Steps**:

```bash
# 1. Verify locally (skip GPG signing)
mvn -B -DskipTests -Drat.skip=true -Dgpg.skip=true \
  -pl seatunnel-shade-calcite,seatunnel-shade-janino,seatunnel-shade-scala-compiler \
  clean install

# 2. Check JAR sizes under target/, then deploy to Apache repository
mvn -B -DskipTests -Drat.skip=true \
  -pl seatunnel-shade-calcite,seatunnel-shade-janino,seatunnel-shade-scala-compiler \
  clean deploy
```

**Key points**:

- Module versions follow `${library.version}-${seatunnel.shade.version}` (e.g. `1.38.0-3.0.0`). As long as the library version changes, there is no conflict with published artifacts.
- **Never** run `mvn clean deploy` without `-pl` — unchanged modules will be redeployed, and Nexus will reject them with a 400 error.
- If only the shade plugin configuration changed (not the library version), you must bump the version manually before deploying.

### Tagging

```bash
git tag v3.0.0.fix
git push origin v3.0.0.fix
```

## Adding a new module

1. Add the new module to `<modules>` and `<properties>` in root `pom.xml`.

2. Create the module POM following `seatunnel-shade-commons-lang3/pom.xml` (Pattern A — produces `original-` + shaded JAR):

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.apache.seatunnel</groupId>
        <artifactId>seatunnel-shade</artifactId>
        <version>3.0.0</version>
    </parent>

    <artifactId>seatunnel-shade-xxx</artifactId>
    <version>${xxx.version}-${seatunnel.shade.version}</version>
    <name>SeaTunnel : Shade : Xxx</name>

    <dependencies>
        <dependency>
            <groupId>...</groupId>
            <artifactId>...</artifactId>
            <version>${xxx.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <phase>package</phase>
                        <configuration>
                            <createDependencyReducedPom>true</createDependencyReducedPom>
                            <dependencyReducedPomLocation>${project.basedir}/target/dependency-reduced-pom.xml</dependencyReducedPomLocation>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <relocations>
                                <relocation>
                                    <pattern>original.package</pattern>
                                    <shadedPattern>${seatunnel.shade.package}.original.package</shadedPattern>
                                </relocation>
                            </relocations>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>flatten-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## Updating SeaTunnel after release

After publishing new shade artifacts, the SeaTunnel project (`apache/seatunnel`) needs corresponding changes to consume the updated versions.

### 1. Update version properties

In SeaTunnel's root `pom.xml`, each shade module has a corresponding version property:

```xml
<properties>
    <seatunnel.shade.version>3.0.0</seatunnel.shade.version>
    <seatunnel.shade.guava.version>27.0-jre</seatunnel.shade.guava.version>
    <seatunnel.shade.janino.version>3.0.11</seatunnel.shade.janino.version>
    <seatunnel.shade.scala-compiler.version>2.12.15</seatunnel.shade.scala-compiler.version>
    <seatunnel.shade.calcite.version>1.38.0</seatunnel.shade.calcite.version>
    <!-- ... other shade modules ... -->
</properties>
```

The effective dependency version is assembled as `${seatunnel.shade.<lib>.version}-${seatunnel.shade.version}` (e.g. `1.38.0-3.0.0`).

When a shade module is updated, bump the corresponding `<lib>.version` property. For example, after upgrading janino from `3.0.11` to `3.1.12`:

```diff
- <seatunnel.shade.janino.version>3.0.11</seatunnel.shade.janino.version>
+ <seatunnel.shade.janino.version>3.1.12</seatunnel.shade.janino.version>
```

If the `seatunnel.shade.version` itself changes (e.g. `3.0.0` → `3.0.1`), it affects **all** shade dependencies and every module must be republished.

### 2. Declare the shaded dependency

All shade modules are centrally managed in SeaTunnel's root `<dependencyManagement>`. For new shade modules, add a corresponding entry following the standard pattern:

```xml
<dependency>
    <groupId>org.apache.seatunnel</groupId>
    <artifactId>seatunnel-shade-<name></artifactId>
    <version>${seatunnel.shade.<name>.version}-${seatunnel.shade.version}</version>
</dependency>
```

Existing entries (all 13 modules):

```xml
<dependency>
    <groupId>org.apache.seatunnel</groupId>
    <artifactId>seatunnel-shade-hadoop3-uber</artifactId>
    <version>${seatunnel.shade.hadoop.version}-${seatunnel.shade.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.seatunnel</groupId>
    <artifactId>seatunnel-shade-guava</artifactId>
    <version>${seatunnel.shade.guava.version}-${seatunnel.shade.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.seatunnel</groupId>
    <artifactId>seatunnel-shade-jackson</artifactId>
    <version>${seatunnel.shade.jackson.version}-${seatunnel.shade.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.seatunnel</groupId>
    <artifactId>seatunnel-shade-calcite</artifactId>
    <version>${seatunnel.shade.calcite.version}-${seatunnel.shade.version}</version>
</dependency>
<!-- ... remaining modules follow the same pattern ... -->
</dependency>
```

### Summary checklist

For each changed shade module:

| Step | Location in SeaTunnel |
|------|----------------------|
| Bump version property | Root `pom.xml` → `<properties>` → `seatunnel.shade.<lib>.version` |
| Declare dependency (if consumed) | Root `pom.xml` → `<dependencyManagement>` |
| Update code imports | All `.java` files using the library |
