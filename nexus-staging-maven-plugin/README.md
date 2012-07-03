# Nexus Staging Maven Plugin

Maven Plugin to control Nexus Staging workflow. While the maven plugin ("staging client") part is OSS, it need a Sonatype Nexus Professional instance 2.1+ on the server side!

# Documentation

Nexus Staging V2 focuses more on client-server automated interaction. 
Hence, the `nexus-staging-maven-plugin` is introduced, with vastly enhanced support. 
*Warning: the "artifactId" of the plugin is newly introduced! The old nexus-maven-plugin is deprecated!* 

## Adding the plugin to your build 

Depending, is it used in Maven3 or Maven2, it almost needs same configuration applied, but Maven2 needs more labour. 

### Maven3 

In Maven3, the simplest needed to be done is to add the plugin to the build, and define it as 
extension and add wanted configuration. Plugin's `LifecycleParticipant` will "automagically" do everything for you: 
it will _disable_ `maven-deploy-plugin:deploy` executions, _inject_ `nexus-staging-maven-plugin:deploy` executions instead. 
This is the simplest way to use the plugin, but in case more control is needed, even in Maven3 it is possible to use the 
plugin in Maven2-way (manually configure all the binding). 

Example snippet of having the plugin in a Profile: 

		<profile>
			<id>nexus-staging</id>
			<build>
				<plugins>
					<plugin>
						<groupId>org.sonatype.plugins</groupId>
						<artifactId>nexus-staging-maven-plugin</artifactId>
						<version>2.1-SNAPSHOT</version>
						<extensions>true</extensions>
						<configuration>
							<nexusUrl>http://localhost:8081/nexus/</nexusUrl>
							<!-- The Base URL of Nexus instance where we want to stage -->
							<serverId>local-nexus</serverId>
							<!-- The server "id" element from settings to use authentication from -->
						</configuration>
					</plugin>
				</plugins>
			</build>
		</profile>


The plugin will activate whenever you pass the `-Pnexus-staging` switch on the CLI (activate the "nexus-staging" profile). 

The lifecycle participant will kick in *only when no binding is detected in modules for `nexus-staging-maven-plugin`*, 
hence, in latter case, it will remain dormant. Example log outputs when plugin kicks in: 

		[cstamas@marvin sisu-goodies (master)]$ mvn clean -Pstaging-test 
		[INFO] Scanning for projects... 
		[INFO] Installing Nexus Staging features: 
		[INFO] ... total of 8 executions of maven-deploy-plugin replaced with nexus-staging-maven-plugin. 
		[INFO] ------------------------------------------------------------------------
		[INFO] Reactor Build Order: 
		[INFO] 
		[INFO] Goodies 
		... 

Plugin not kicking in:

		cstamas@marvin sisu-goodies (master)]$ mvn clean -Pstaging-test 
		[INFO] Scanning for projects... 
		[INFO] Not installing Nexus Staging features, some Staging related goal bindings already present. 
		[INFO] ------------------------------------------------------------------------
		[INFO] Reactor Build Order: 
		[INFO] 
		[INFO] Goodies 
		... 

Above, in both cases Maven3 is used (as lifecycle participant works with it only). In first case, 
lifecycle participant activated itself, and performed 8 "swaps" on the fly. In second case, it 
found `nexus-staging-maven-plugin` bindings already present (hence, the Maven2-way of configuration was applied), and it just stay put. 

### Maven2 (or "Pro" mode) 

In Maven2, or even with Maven3 if you want more control, explicit configuration is needed. 
First, you would want to `skip=true` of "vanilla" `maven-deploy-plugin` or even the best, completely remove 
it from the build (Note: skip parameter is present since version 2.4). Second, you'd want 
to add the plugin `org.sonatype.plugins:nexus-staging-maven-plugin:2.1-SNAPSHOT` to the build. 

Example snippet of having the plugin in a Profile: 

		<profile>
			<id>nexus-staging</id>
			<build>
				<plugins>
					<plugin>
						<groupId>org.apache.maven.plugins</groupId>
						<artifactId>maven-deploy-plugin</artifactId>
						<configuration>
							<skip>true</skip>
						</configuration>
					</plugin>
					<plugin>
						<groupId>org.sonatype.plugins</groupId>
						<artifactId>nexus-staging-maven-plugin</artifactId>
						<version>2.1-SNAPSHOT</version>
						<executions>
							<execution>
								<id>default-deploy</id>
								<phase>deploy</phase>
								<!-- By default, this is the phase deploy goal will bind to -->
								<goals>
									<goal>deploy</goal>
								</goals>
							</execution>
						</executions>
						<configuration>
							<nexusUrl>http://localhost:8081/nexus/</nexusUrl>
							<!-- The Base URL of Nexus instance where we want to stage -->
							<serverId>local-nexus</serverId>
							<!-- The server "id" element from settings to use authentication from -->
						</configuration>
					</plugin>
				</plugins>
			</build>
		</profile>


This above is "minimum" configuration of Nexus, and it will trigger the use of "V2 implicit" mode. 

## Configuring the plugin 

Minimal requirement for configuration are two entries: `nexusUrl` and `serverId`. 

| Configuration | Meaning |
|---------------|---------|
| `nexusUrl` | *Mandatory*. Has to point to the *base URL of target Nexus*. |
| `serverId` | *Mandatory*. Has to hold an ID of a `<server>` section from Maven's `settings.xml` to pick authentication information from. |

With configuration as above, if you have suitable profile in Nexus that will be matched, everything should work. If not profile match possible, upload will fail. 

Still, you can narrow configuration. 

|Configuration (additional to mandatory ones)|Performs staging?|Profile matching happens?|Staging repository created?|Post-staging repository management (close/drop)?|Remarks|
|--------------------------------------------|-----------------|-------------------------|---------------------------|------------------------------------------------|-------|
| nothing more than mandatory ones | Yes | Yes | Yes | Yes | This is the "V2 implicit" way of using Staging V2. Matches the profile once, and manages the staging repository from it's creation to it's end (close or drop) | 
| `stagingProfileId` | Yes | No | Yes | Yes | This is the "V2 explicit" way of using Staging V2: targeted profile, so no match is done. Naturally, the repository type of the profile should match of that being deployed. Manages the staging repository from it's creation to it's end (close or drop) | 
| `stagingProfileId` and `stagingRepositoryId` | Yes | No | No | No | Advanced V2 usage. This is usable when some "external component" (script? CI?) performs V2 actions of repository creation etc, and only "targeted" deploy happens against given (open) staging repository. Since repository is created by some other entity, it will be NOT managed by client (the one creating it should close it too). For example: multi machine build should end up in _same staging repository_ (not doable with V1, see "Oracle problem"). |
| `deployUrl` | No | No | No | No | Here, a simple "atomic deploy" happens against given `deployUrl`. The only difference between "vanilla" deploy (performed by disabled `deploy-maven-plugin` and this, is that "local staging" would still happen, and atomic upload will be used to upload all the artifacts. This is logically equivalent to using plain "deploy" plugin. In contrary to `nexusUrl`, this value has to point to the *base URL of a Nexus repository, not the Nexus base!*| 

This table also presents the "order" how configuration is interpreted: last wins. For example, if all `stagingProfileId`, `stagingRepositoryId` and `deployUrl` is present, `deployUrl` wins, "plain" deploy will happen without using any of the V2 Staging features on server side. 

### Plugin flags 

These "flags" are usually passed in from CLI (`-D...`).

|Flag type|CLI (`-D`)|configuration|Default value|Meaning|
|---------|----------|-------------|-------------|-------|
| Alternate local staging directory (FS directory path) | `altStagingDirectory` | n/a | `null` | Possibility to _explicitly_ define a directory on local FS to use for local staging. Passing in this flag will prevent the "logic" of proper `target` folder selection |
| Description (plain text)| `description` | `<description>` | `null` | Free text, message or explanation to be added for staging operations like when staging repository is created or closed (as part of whole V2 process) |
| Keep staging repository in case of failure (boolean) | `keepStagingRepositoryOnFailure` | `< keepStagingRepositoryOnFailure >` | `false` | Nexus Maven Plugin always tries to "clean up" after itself, hence, in case of upload failure (and potentially having "partially uploaded" artifacts to staging repository) it always tries to drop that same repository. Will not, if this flag is set to `true` |
| Skip whole plugin (boolean) | `skipStaging` | `<skipStaging>` | `false` | Completely skips the `deploy` Mojo (similar as `maven.deploy.skip`) |
| Skip the upload step (boolean) | `skipRemoteStaging` | `<skipRemoteStaging>` | `false` | Performs "local staging" only, skips the upload. Hence, no stage repository created, nor deployed to Nexus (if `deployUrl` specified).|

### Tagging staging repositories 

User is able to simply "decorate" the Staging repository with key-value pairs, that will get stored along Staged repository configuration. 
Simply add following section to `nexus-staging-maven-plugin` configuration section (in other words, create a "map" in plugin configuration, see configuring Maven Mojos): 

		<configuration>
			<!-- The Base URL of Nexus instance where we want to stage -->
			<nexusUrl>http://localhost:8081/nexus/</nexusUrl>
			<!-- The server "id" element from settings to use authentication from -->
			<serverId>local-nexus</serverId>
			<tags>
				<localUsername>${env.USER}</localUsername>
				<javaVersion>${java.version}</javaVersion>
			</tags>
		</configuration>

Configuration is plain "Mojo map", element names will become "keys", and their content the "value". 
Values are evaluated using standard Maven way, so you can source them from system properties, env variables, etc. 

## Plugin Goals 

Below is a short list of existing plugin goals and description how they are intended to be used. 

### Build Action goals 

These goals are the ones that should be bound (either in Maven3-way "magically" or explicitly in Maven2-way, does not matter) 
to phases of your build. Using these goals, one can combine almost any staging solution they need. Examples: 

* using the `deploy` (or `deploy-staged` when `deploy` had set `skipRemoteStaging` parameter) one ends with *closed staging repository*. Still, you can incorporate these goals into your build, and have the repository *released or promoted* even, from your builds. 
* in case of "targeted repository ID" scenario, where the Staging repository is created by some external entity (script or some other entity), and this plugin *does not manage the staging repository* (will just deploy to it but will not try to close it), you can still have it closed from one of your CI nodes by having some conditions met (ie. from a profile or so). 

Hence, their configuration usually sits in the POM (at least the minimal ones, like `serverId` and `nexusUrl`). 

Goals except `deploy` and `deploy-staged` all has one parameter: `stagingRepositoryId`. These goals may receive that parameter from CLI, but in case is not given, will take that value from the properties file ("context") saved to the root of locally staged repository. 

#### `deploy` 

This goal actually performs the whole deployment together with "staging workflow": 

* locally stages, 
* selects a profile (either by server-side matching or using user provided profileId) 
* selects a staging repository (either by creating one "private" or using user provided repositoryId) 
* performs atomic upload into staging repository 
* closes it (or drops if upload failed) 

With this goal, the user has no need for the other ones. It might "skip" the remote staging, then only 1st step is executed. 

#### `deploy-staged` 

This goal performs the "staging workflow" only for previously ran local staging. 

#### `close` 

Closes the staging repository. For advanced use only. 

#### `drop` 

Drops the staging repository. For advanced use only. 

#### `release` 

Releases the staging repository. For advanced use only. 

#### `promote` 

Promotes the staging repository. For advanced use only. It needs extra parameter: `buildPromotionProfileId` 

### RC Action goals 

These goals are remote controlling goals, and they *do not need a project to be executed*, and can be *directly invoked from CLI only*. 

Hence, they are "RC" (as "remote control") goals, made for convenience only to perform some Staging 
Workflow operations using your favorite tool (Maven) running it from a CLI (just for fun, or 
because you are in a headless environment and have no access to Nexus UI). 

All of them expect *explicit configuration* usually passed in over CLI parameters (`-Dfoo=bar...`). 

All of them accept mandatory parameters to connect to Nexus, the `nexusUrl` and `serverId`. 

All of them accept mandatory `stagingRepositoryId` parameter, similarly to other staging goals, 
*with exception that in this case the parameter is split using "," (comma) as delimiter*. 
Hence, all these goals might operate against *one or more staging repository* (bulk operation). 

All of them accept optional `description` parameter, but it's not mandatory. If not specified, a default description will be applied. 

#### `rc-close` 

Closes the specified staging repositories. 

Example invocation: 

		mvn nexus-staging:rc-close -DserverId=local-nexus -DnexusUrl=http://localhost:8081/nexus -DstagingRepositoryId=repo1,repo2 -Ddescription="The reason I close these is..." 


#### `rc-drop` 

Drops the specified staging repositories. 

Example invocation: 

		mvn nexus-staging:rc-drop -DserverId=local-nexus -DnexusUrl=http://localhost:8081/nexus -DstagingRepositoryId=repo1,repo2 -Ddescription="The reason I drop these is..." 

#### `rc-release` 

Releases the specified closed staging repositories. 

Example invocation: 

		mvn nexus-staging:rc-release -DserverId=local-nexus -DnexusUrl=http://localhost:8081/nexus -DstagingRepositoryId=repo1,repo2 -Ddescription="The reason I release these is..." 

#### `rc-promote` 

Performs a build profile promotion on the specified closed staging repositories. This goal *has one extra mandatory parameter*: The `buildPromotionProfileId`. 

Example invocation: 


		mvn nexus-staging:rc-promote -DserverId=local-nexus -DnexusUrl=http://localhost:8081/nexus -DbuildPromotionProfileId=foo -DstagingRepositoryId=repo1,repo2 -Ddescription="The reason I promote these is..." 