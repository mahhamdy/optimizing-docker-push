## Pushing Only Changes technique

Instead of pushing the entire Docker image every time, we can optimize the push process by pushing only the changed files This is done using Docker's layer caching mechanism. When you make changes to your *Dockerfile* or *code*, only the affected layers will be rebuilt and pushed.



Let's apply this to the Maven application: 
Our goal is to **split** the **compiled Java class files** from the used **libs** to push only the changed files.

### Steps

#### Maven:

1. Specify a directory called 'out'  that will contain the jar file & the libs of the application.

   ```
   <properties>
       <output.directory>out</output.directory>
   </properties>
   ```

2. Delete all files and directories within the directory specified by `${output.directory}`.

   ```
   <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-clean-plugin</artifactId>
       <configuration>
           <filesets>
               <fileset>
                   <directory>${output.directory}</directory>
                   <includes>
                       <include>**</include>
                   </includes>
                   <followSymlinks>false</followSymlinks>
               </fileset>
           </filesets>
       </configuration>
   </plugin>
   ```

3. Build a JAR file to be outputted to the `${output.directory}`.

   ```
   <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-jar-plugin</artifactId>
       <configuration>
           <archive>
               <manifest>
                   <addClasspath>true</addClasspath>
                   <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
                   <mainClass> </mainClass>  <!--  add the path of main class here -->
               </manifest>
           </archive>
           <outputDirectory>${output.directory}</outputDirectory>
       </configuration>
   </plugin>
   ```

4. Copy project dependencies with the runtime scope to the `out/lib` directory during the **`package`** phase.

   ```
   <plugin>
       <groupId>org.apache.maven.plugins</groupId>
       <artifactId>maven-dependency-plugin</artifactId>
       <executions>
           <execution>
               <id>copy-dependencies</id>
               <phase>package</phase>
               <goals>
                   <goal>copy-dependencies</goal>
               </goals>
               <configuration>
                   <outputDirectory>${output.directory}/lib</outputDirectory>
                   <overWriteReleases>false</overWriteReleases>
                   <overWriteSnapshots>false</overWriteSnapshots>
                   <overWriteIfNewer>true</overWriteIfNewer>
                   <includeScope>runtime</includeScope>
               </configuration>
           </execution>
       </executions>
   </plugin>
   ```

5. Run Docker commands to build a docker image during the **`install`** phase.

   ```
   <plugin>
       <groupId>org.codehaus.mojo</groupId>
       <artifactId>exec-maven-plugin</artifactId>
       <version>1.6.0</version>
       <executions>
           <!--
             Create new docker image using Dockerfile which must be present in current working directory.
             Tag the image using maven project version information.
           -->
           <execution>
               <id>docker-build</id>
               <phase>install</phase>
               <goals>
                   <goal>exec</goal>
               </goals>
               <configuration>
                   <executable>docker</executable>
                   <workingDirectory>${project.basedir}</workingDirectory>
                   <arguments>
                       <argument>build</argument>
                       <argument>-t</argument>
                       <argument></argument>  <!--  add the docker image tag here -->
                       <argument>.</argument>
                   </arguments>
               </configuration>
           </execution>
       </executions>
   </plugin>
   ```

   

##### DockerFile

```
#1- Use a lightweight Alpine Linux-based image containing the JDK 17.
FROM bellsoft/liberica-openjdk-alpine:17.0.7

#2- Set the working directory inside the Docker container.
WORKDIR /workdir

#3- Copy the log4j2.xml file from the docker directory into the config directory inside the Docker image. (this will be used later for setting log file location)
ADD docker/log4j2.xml config/

#4- Copy the dependencies (libs) & jar file into the Docker image.
COPY out/lib/. libs/
COPY out/afaqy-service-events.jar libs/

5# executes a shell command within the Docker container during the image build process.
RUN sh -c 'touch libs/<app_jar_name>.jar'

6# Runs the Java app with JVM options and system properties. (GC settings, mem allocation, char encoding, and a log4j config file)
ENTRYPOINT ["java", "-jar", "-XX:+UseG1GC", "-XX:+UseStringDeduplication","-Xmx2g", "-Xms32m", "-Dfile.encoding=UTF-8", "-Dlog4j.configurationFile=config/log4j2.xml","libs/app_jar_name.jar"]
```

